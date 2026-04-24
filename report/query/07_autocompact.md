# 07. AutoCompact 와 쿼리 루프의 상호작용

`compactConversation()` 자체의 구현 세부는 [../context/02_context_compression.md](../context/02_context_compression.md) 를 참고하세요. 이 문서는 **compaction 이 쿼리 루프에 언제·어떻게 끼어드는지**에 초점을 둡니다.

## 관련 파일

- [src/services/compact/autoCompact.ts](../../src/services/compact/autoCompact.ts) — `shouldAutoCompact`, `autoCompactIfNeeded`
- [src/query.ts](../../src/query.ts) — 루프 3단계(proactive)와 7단계(reactive) 호출 지점
- [src/services/compact/compact.ts](../../src/services/compact/compact.ts) — `compactConversation()`

---

## 두 가지 개입 경로

쿼리 루프는 컴팩션을 **두 번** 시도할 수 있습니다.

1. **Proactive (3단계)** — API 호출 **전에** 토큰 카운트를 재보고, 임계값을 넘을 것 같으면 선제 compaction.
2. **Reactive (7단계)** — API 호출이 `413 prompt-too-long` 이나 `media-size` 에러로 실패했을 때, 에러를 보류(`withheld`)해 둔 상태로 compaction 을 시도하고 성공하면 재시도.

두 경로 모두 성공하면 `messagesForQuery` 를 교체하고 **루프 `continue`** 로 같은 turn 을 다시 시작합니다.

---

## Proactive: `shouldAutoCompact`

임계값은 모델의 effective context window 에서 `AUTOCOMPACT_BUFFER_TOKENS = 13,000` 만큼 뺀 값입니다. 환경변수로 테스트용 퍼센트 오버라이드가 가능합니다.

```typescript
// src/services/compact/autoCompact.ts
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

  const envPercent = process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
  if (envPercent) {
    const parsed = parseFloat(envPercent)
    if (!isNaN(parsed) && parsed > 0 && parsed <= 100) {
      const percentageThreshold = Math.floor(
        effectiveContextWindow * (parsed / 100),
      )
      return Math.min(percentageThreshold, autocompactThreshold)
    }
  }
  return autocompactThreshold
}
```

판정 함수는 재귀 가드를 먼저 체크합니다. `session_memory`, `compact`, `marble_origami` 같은 내부 쿼리가 다시 compaction 을 트리거하면 무한 재귀가 되기 때문입니다.

```typescript
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
  snipTokensFreed = 0,
): Promise<boolean> {
  if (querySource === 'session_memory' || querySource === 'compact') return false
  if (feature('CONTEXT_COLLAPSE') && querySource === 'marble_origami') return false
  if (!isAutoCompactEnabled()) return false

  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const { isAboveAutoCompactThreshold } =
    calculateTokenWarningState(tokenCount, model)

  return isAboveAutoCompactThreshold
}
```

---

## Proactive 호출 — 루프 3단계

```typescript
// src/query.ts — 3단계
queryCheckpoint('query_autocompact_start')
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  { systemPrompt, userContext, systemContext, toolUseContext,
    forkContextMessages: messagesForQuery },
  querySource,
  tracking,
  snipTokensFreed,
)
queryCheckpoint('query_autocompact_end')

if (compactionResult) {
  logEvent('tengu_auto_compact_succeeded', {
    preCompactTokenCount,
    postCompactTokenCount,
    truePostCompactTokenCount,
    compactionUsage: { input_tokens, output_tokens,
                        cache_read_input_tokens, cache_creation_input_tokens },
    queryChainId: queryChainIdForAnalytics,
    queryDepth: queryTracking.depth,
  })

  // 서브에이전트 예산 차감
  if (params.taskBudget) {
    const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
    taskBudgetRemaining = Math.max(
      0,
      (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
    )
  }

  const postCompactMessages = buildPostCompactMessages(compacted)
  for (const msg of postCompactMessages) yield msg

  state = {
    messages: postCompactMessages,
    autoCompactTracking: {
      compacted: true,
      turnCounter: 0,
      turnId: deps.uuid(),
    },
    transition: { reason: 'autocompact' },
    ...
  }
  continue
}
```

핵심 부수효과:

- `postCompactMessages` 는 **사용자 화면에도 yield** 됩니다. compact 경계 마커와 요약 메시지가 transcript 에 남아 나중에 "여기서 한 번 요약됐다" 는 흔적이 됩니다.
- `taskBudget` 이 있는 경우 (서브에이전트) 컴팩션이 소비한 프리-컴팩트 컨텍스트 만큼 예산을 차감합니다.
- `autoCompactTracking.turnId` 가 갱신되어, 같은 compact 체인 안에서 바로 다시 compact 가 트리거되지 않도록 합니다.

---

## `autoCompactIfNeeded` — 서킷 브레이커와 세션 메모리 경로

실제 compaction 로직은 `autoCompactIfNeeded` 안에 있습니다. 서킷 브레이커와 세션 메모리 경로가 중요한 디테일입니다.

```typescript
// src/services/compact/autoCompact.ts
export async function autoCompactIfNeeded(
  messages: Message[],
  toolUseContext: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  querySource?: QuerySource,
  tracking?: AutoCompactTrackingState,
  snipTokensFreed?: number,
): Promise<{
  wasCompacted: boolean
  compactionResult?: CompactionResult
  consecutiveFailures?: number
}> {
  // 서킷 브레이커: 연속 실패 임계치 초과하면 중단
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }
  }

  const model = toolUseContext.options.mainLoopModel
  if (!(await shouldAutoCompact(messages, model, querySource, snipTokensFreed))) {
    return { wasCompacted: false }
  }

  const recompactionInfo: RecompactionInfo = {
    isRecompactionInChain: tracking?.compacted === true,
    turnsSincePreviousCompact: tracking?.turnCounter ?? -1,
    previousCompactTurnId: tracking?.turnId,
    autoCompactThreshold: getAutoCompactThreshold(model),
    querySource,
  }

  // EXPERIMENT: 먼저 "세션 메모리" 로 가볍게 압축 시도
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages,
    toolUseContext.agentId,
    recompactionInfo.autoCompactThreshold,
  )
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
      notifyCompaction(querySource ?? 'compact', toolUseContext.agentId)
    }
    markPostCompaction()
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // 폴백: 일반 compactConversation
  try {
    const compactionResult = await compactConversation(
      messages,
      toolUseContext,
      cacheSafeParams,
      /* suppressUserQuestions */ true,
      undefined,
      /* isAutoCompact */ true,
      recompactionInfo,
    )
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    return { wasCompacted: true, compactionResult, consecutiveFailures: 0 }
  } catch (error) {
    const prevFailures = tracking?.consecutiveFailures ?? 0
    const nextFailures = prevFailures + 1
    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

포인트:

- **서킷 브레이커** — 연속 실패가 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` 를 넘으면 이후 turn 에서는 AutoCompact 시도 자체를 건너뛴다. 실패가 반복되면 프롬프트 루프가 DoS 되는 것을 막기 위함.
- **세션 메모리 우선** — `trySessionMemoryCompaction` 이 먼저 시도되고, 이미 세션 메모리로 짐을 덜 수 있으면 LLM 호출 없이 해결.
- **일반 compaction 은 LLM 호출** — 실패 시 counter 만 올리고 조용히 `wasCompacted: false` 반환 → 해당 turn 은 **원본 messages 로 API 호출이 진행**되며, 그 결과가 413 이면 7단계의 reactive compact 에서 다시 시도.

---

## Reactive: 7단계 복구 경로

스트리밍/호출에서 `413 prompt_too_long` 이 발생하면 그 에러를 바로 올리지 않고 `withheld` 플래그로 보류합니다. 7단계에서 우선 **context collapse drain** 을 시도하고, 그래도 안 되면 **reactive compact** 를 시도합니다.

```typescript
// src/query.ts — 7단계 중 일부
if (isWithheld413) {
  const drained = contextCollapse.recoverFromOverflow(...)
  if (drained.committed > 0) {
    state = {
      messages: drained.messages,
      transition: { reason: 'collapse_drain_retry', committed: drained.committed },
      ...
    }
    continue
  }
}

if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({
    hasAttempted: hasAttemptedReactiveCompact,
    querySource,
    aborted: toolUseContext.abortController.signal.aborted,
    messages: messagesForQuery,
    cacheSafeParams: {
      systemPrompt, userContext, systemContext, toolUseContext,
      forkContextMessages: messagesForQuery,
    },
  })

  if (compacted) {
    if (params.taskBudget) {
      const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
      taskBudgetRemaining = Math.max(
        0,
        (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
      )
    }
    const postCompactMessages = buildPostCompactMessages(compacted)
    for (const msg of postCompactMessages) yield msg
    state = {
      messages: postCompactMessages,
      hasAttemptedReactiveCompact: true,   // 한 turn 에 두 번 시도하지 않도록
      transition: { reason: 'reactive_compact_retry', ... },
      ...
    }
    continue
  }
}
```

`hasAttemptedReactiveCompact` 는 같은 turn 안에서 reactive 시도를 1회로 제한합니다. 여기서도 실패하면 원래의 413 에러가 상위로 올라가고 쿼리 루프는 에러로 종료됩니다.

---

## Transition 메시지의 종류

`State.transition.reason` 은 다음 반복이 "왜 들어왔는지" 를 명시합니다. 로깅·분석·UI 힌트에 사용됩니다.

| reason | 유발 조건 |
|--------|-----------|
| `autocompact` | Proactive compaction 성공 |
| `reactive_compact_retry` | 413/media-size 이후 reactive compact 성공 |
| `collapse_drain_retry` | 413 이후 context collapse drain 이 토큰 확보 |
| `max_output_tokens_escalate` | max_output_tokens 보류 → 상한 상향 |
| `follow_up_needed` | 이번 turn 의 tool_use 결과를 반영해서 다시 호출 |

---

## 한 장 요약

```
                ┌──► [3단계] shouldAutoCompact? ── YES ──► autoCompactIfNeeded
                │                                            ├─ trySessionMemory
                │                                            └─ compactConversation
                │                                               │
                │                                               ▼
                │                                       messages 교체 + continue
Query turn ─────┤
                │
                │     API 호출
                │       │
                │       ├─ 성공 → 6,7단계로
                │       │
                │       └─ 413 / media 에러 보류 ─► [7단계]
                │                                     ├─ collapse drain ── 성공 ── continue
                │                                     └─ reactive compact ─ 성공 ── continue
                │                                                         └ 실패 ── throw
```

---

## 관련 문서

- [02_query_loop.md](02_query_loop.md) — 전체 루프 단계 정의
- [../context/02_context_compression.md](../context/02_context_compression.md) — `compactConversation` 자체의 동작
