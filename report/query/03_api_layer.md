# 03. API 호출 레이어 (Streaming / Non-streaming)

## 관련 파일

- [src/services/api/claude.ts](../../src/services/api/claude.ts) — `queryModel`, `queryModelWithStreaming`, `executeNonStreamingRequest`, `buildSystemPromptBlocks`
- [src/services/api/withRetry.ts](../../src/services/api/withRetry.ts) — 재시도 래퍼, `FallbackTriggeredError`

---

## 스트리밍 쿼리 엔트리

`queryLoop` 의 `deps.callModel` 은 기본적으로 `queryModelWithStreaming` 을 가리킵니다. 내부에서는 VCR 래퍼로 테스트 녹음·재생 지원을 얹은 다음, 실제 `queryModel` 을 generator 로 delegate 합니다.

```typescript
// src/services/api/claude.ts
export async function* queryModelWithStreaming({
  messages,
  systemPrompt,
  thinkingConfig,
  tools,
  signal,
  options,
}: { ... }): AsyncGenerator<
  StreamEvent | AssistantMessage | SystemAPIErrorMessage,
  void
> {
  return yield* withStreamingVCR(messages, async function* () {
    yield* queryModel(messages, systemPrompt, thinkingConfig, tools, signal, options)
  })
}
```

스트리밍으로 돌아오는 이벤트는 세 종류입니다:

| 이벤트 | 의미 |
|--------|------|
| `StreamEvent` | content block delta, thinking 델타 등 SDK 에서 올라오는 저수준 이벤트 |
| `AssistantMessage` | 한 assistant turn 이 완성되면 통째로 한 번 |
| `SystemAPIErrorMessage` | 복구 불가 에러를 user-visible 시스템 메시지로 래핑 |

`queryLoop` 는 이 세 종류를 그대로 상위에 `yield` 하며, `AssistantMessage` 를 받을 때 `tool_use` 블록을 훑어서 `StreamingToolExecutor.addTool()` 를 호출합니다.

---

## Non-streaming 폴백

스트리밍이 끊기거나 네트워크 환경이 불안정하면 non-streaming 경로로 떨어집니다. 이 경로는 **`anthropic.beta.messages.create`** 를 한 번 호출하는 형태이며, Claude API 의 10분 제한 때문에 `max_tokens` 와 `thinking budget` 을 보수적으로 낮춥니다.

```typescript
// src/services/api/claude.ts
export async function* executeNonStreamingRequest(
  clientOptions: { model, fetchOverride?, source, providerOverride? },
  retryOptions: {
    model, fallbackModel?, thinkingConfig, fastMode?, signal,
    initialConsecutive529Errors?, querySource?,
  },
  paramsFromContext: (context: RetryContext) => BetaMessageStreamParams,
  onAttempt: (attempt: number, start: number, maxOutputTokens: number) => void,
  captureRequest: (params: BetaMessageStreamParams) => void,
  originatingRequestId?: string | null,
): AsyncGenerator<SystemAPIErrorMessage, BetaMessage> {
  const fallbackTimeoutMs = getNonstreamingFallbackTimeoutMs()
  const generator = withRetry(
    () => getAnthropicClient({ ... }),
    async (anthropic, attempt, context) => {
      const adjustedParams = adjustParamsForNonStreaming(
        retryParams,
        MAX_NON_STREAMING_TOKENS,  // 64,000 토큰 상한
      )
      return await anthropic.beta.messages.create(
        { ...adjustedParams, model: normalizeModelStringForAPI(...) },
        { signal: retryOptions.signal, timeout: fallbackTimeoutMs },
      )
    },
    { model, fallbackModel, ... }
  )
}
```

제약과 이유:

- **`max_tokens` 최대 64,000** — Claude API 의 non-streaming 10분 제한을 초과하지 않기 위한 보수적 기본값
- **`timeout`** — 원격 세션은 120s, 로컬은 300s 로 달리 설정 (`getNonstreamingFallbackTimeoutMs`)
- **thinking budget** — 항상 `max_tokens - 1` 이하로 조정됨

---

## 재시도와 모델 폴백

`withRetry` 는 지수 백오프·일시적 에러 재시도·모델 폴백 트리거를 하나로 묶어 주는 래퍼입니다. 주요 분기는 다음과 같습니다.

| 조건 | 동작 |
|------|------|
| 529 Overloaded 연속 발생 | `FallbackTriggeredError` throw → 루프가 `fallbackModel` 로 재진입 |
| `APIConnectionTimeoutError` | 백오프 후 동일 모델로 재시도 |
| `APIUserAbortError` | 즉시 상위로 re-throw (사용자 취소) |
| 기타 5xx | 백오프 후 재시도 |

```typescript
// src/services/api/withRetry.ts
export class FallbackTriggeredError extends APIError {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
    public readonly originalError: APIError,
  ) { ... }
}
```

`queryLoop` 는 이 에러를 잡아 `currentModel` 을 교체하고, 지금까지 부분적으로 yield 된 assistant 메시지를 tool_result 짝이 없는 상태로 남기지 않기 위해 `yieldMissingToolResultBlocks` 로 보정한 뒤 `continue` 합니다.

---

## 시스템 프롬프트 → API 블록 변환

API 로 보내기 직전, `SystemPrompt` 배열은 `buildSystemPromptBlocks` 를 거쳐 Anthropic SDK 가 요구하는 `TextBlockParam[]` 으로 직렬화됩니다. 이 단계에서 **블록별 `cache_control`** 이 결정됩니다.

```typescript
// src/services/api/claude.ts
export function buildSystemPromptBlocks(
  systemPrompt: SystemPrompt,
  enablePromptCaching: boolean,
  options?: {
    skipGlobalCacheForSystemPrompt?: boolean
    querySource?: QuerySource
  },
): TextBlockParam[] {
  return splitSysPromptPrefix(systemPrompt, {
    skipGlobalCacheForSystemPrompt: options?.skipGlobalCacheForSystemPrompt,
  }).map(block => ({
    type: 'text' as const,
    text: block.text,
    ...(enablePromptCaching && block.cacheScope !== null && {
      cache_control: getCacheControl({
        scope: block.cacheScope,
        querySource: options?.querySource,
      }),
    }),
  }))
}
```

- `splitSysPromptPrefix` 가 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 마커 기준으로 정적/동적 블록을 나눕니다.
- `cacheScope: 'global'` 블록만 `cache_control` 이 붙고, `null` 블록은 캐시 없이 전송됩니다.
- 블록 분할과 캐시 경계에 대한 자세한 내용은 [05_system_prompt.md](05_system_prompt.md) 참고.

---

## 한 번의 호출에서 일어나는 일

```
queryLoop ── calls deps.callModel()
                │
                ▼
      queryModelWithStreaming (VCR wrapper)
                │
                ▼
      queryModel ── Anthropic SDK messages.stream(...)
                │
          ┌─────┴──────────────────────┐
          ▼                            ▼
    StreamEvent delta            AssistantMessage (완성)
          │                            │
          ▼                            ▼
     queryLoop.yield()          tool_use 블록 감지 →
                                StreamingToolExecutor.addTool()
```

에러가 나면:

```
SDK error
  │
  ▼
withRetry ── 529 누적? ──► FallbackTriggeredError throw
  │                           │
  ▼                           ▼
지수 백오프 재시도        queryLoop 에서 catch → fallbackModel 로 재진입
```

---

## 다음 문서

- [04_tool_execution.md](04_tool_execution.md) — `StreamingToolExecutor` 가 tool 을 어떻게 병렬/순차로 실행하는가
