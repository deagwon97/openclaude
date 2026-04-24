# 02. 메인 쿼리 루프 (`queryLoop`)

## 관련 파일

- [src/query.ts](../../src/query.ts) — `queryLoop()`, `State`, 단계별 헬퍼
- [src/services/compact/autoCompact.ts](../../src/services/compact/autoCompact.ts) — compaction 진입점
- [src/services/tools/StreamingToolExecutor.ts](../../src/services/tools/StreamingToolExecutor.ts) — tool 실행

---

## 전체 구조

`queryLoop()` 은 **`while (true)` 위에서 동작하는 상태 머신**입니다. 각 반복은 한 번의 LLM turn 에 해당하고, 반복 사이에 `State` 라는 불변 스냅샷이 교체됩니다. `continue` 를 유발하는 경로는 (a) 일반 follow-up, (b) AutoCompact 성공, (c) Reactive compact 복구, (d) max output tokens 상향, (e) context collapse drain 재시도 — 다섯 가지뿐입니다.

```typescript
// src/query.ts
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // 직전 반복이 왜 continue 했는지
}
```

---

## 단계별 분해

### 1단계 — 메시지 정규화와 토큰 예산

루프에 들어오자마자 이번 turn 에 API 로 보낼 `messagesForQuery` 를 확정합니다. 과거 compact 경계 이후만 잘라오고, 여러 절약 기법을 순서대로 적용합니다.

```typescript
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

// (1) 거대한 tool 결과의 교체본 적용 — contentReplacementState 기반
const toolResultBudgetResult = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  /* persistCb */ persistReplacements ? ... : undefined,
)
messagesForQuery = toolResultBudgetResult.messages

// (2) 과거 메시지 일부를 잘라 요약으로 대체 (feature gate)
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
}

// (3) Microcompact — 아주 가벼운 로컬 요약
const microcompactResult = await deps.microcompact(
  messagesForQuery, toolUseContext, querySource,
)
messagesForQuery = microcompactResult.messages

// (4) Context collapse — 특정 구간을 요약 블록으로 축소
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
  messagesForQuery = collapseResult.messages
}
```

이 순서가 중요합니다: **tool 결과 예산 → history snip → microcompact → context collapse**. 각 단계는 앞 단계 결과 위에서 동작합니다.

### 2단계 — 시스템 프롬프트 조립

정적 시스템 프롬프트 배열의 **마지막에** 동적 `systemContext` (git 상태 등) 를 한 줄씩 append 해서 최종 system prompt 를 구성합니다. `userContext` 는 이 단계가 아니라 **API 호출 직전에** 첫 user 메시지로 prepend 됩니다.

```typescript
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

상세 구조는 [05_system_prompt.md](05_system_prompt.md), 주입 방식은 [06_context_injection.md](06_context_injection.md) 참고.

### 3단계 — AutoCompact 체크

실제 API 호출 전에 토큰 카운트를 재면서 자동 compaction 이 필요한지 판단합니다. 트리거되면 messages 를 교체하고 `continue` 로 루프를 다시 돕니다.

```typescript
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  { systemPrompt, userContext, systemContext, toolUseContext,
    forkContextMessages: messagesForQuery },
  querySource,
  tracking,
  snipTokensFreed,
)

if (compactionResult) {
  const postCompactMessages = buildPostCompactMessages(compacted)
  for (const msg of postCompactMessages) yield msg
  state = {
    messages: postCompactMessages,
    autoCompactTracking: { compacted: true, turnCounter: 0, turnId: ... },
    transition: { reason: 'autocompact' },
    ...
  }
  continue
}
```

자세한 동작은 [07_autocompact.md](07_autocompact.md) 참고.

### 4단계 — Claude API 호출 (스트리밍)

`deps.callModel()` 은 AsyncGenerator 를 돌려주며, 델타 이벤트·완성된 assistant 메시지·SystemAPIError 메시지를 순차 `yield` 합니다. 루프는 각 이벤트를 소비하면서 필요시 **스트리밍 도중** tool 을 병렬 실행합니다.

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fastMode: appState.fastMode,
    querySource,
    maxOutputTokensOverride,
    ...
  },
})) {
  // 스트리밍 실패 → 이미 쌓인 assistant 메시지를 tombstone 으로 무효화
  if (streamingFallbackOccured) {
    for (const msg of assistantMessages) {
      yield { type: 'tombstone' as const, message: msg }
    }
    assistantMessages.length = 0
  }

  // tool_use 블록이 확정되면 스트리밍 실행기에 바로 넘김
  if (useStreamingToolExecution && streamingToolExecutor) {
    streamingToolExecutor.addTool(toolUseBlock, assistantMessage)
    for (const result of streamingToolExecutor.getCompletedResults()) {
      if (result.message) {
        yield result.message
        toolResults.push(...)
      }
    }
  }

  yield yieldMessage
}
```

**주의:** `prependUserContext` 는 매 호출마다 적용됩니다. 캐시 입장에서 user context 블록은 항상 "첫 user 메시지 바로 앞"에 머물러야 키 일관성이 유지됩니다.

### 5단계 — 스트리밍 에러 복구

스트림 중 발생하는 예외는 크게 세 부류입니다. 각각 루프 재진입으로 복구합니다.

```typescript
catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true

    // 지금까지 부분적으로 yield 된 assistant 메시지를 tool_result 짝 없이
    // 남기지 않도록 보정 — 누락된 tool_result 블록을 합성해서 yield
    yield* yieldMissingToolResultBlocks(assistantMessages, ...)
    assistantMessages.length = 0

    if (streamingToolExecutor) {
      streamingToolExecutor.discard()
      streamingToolExecutor = new StreamingToolExecutor(...)
    }
    continue
  }
  throw innerError
}
```

- **FallbackTriggeredError** — 529 overloaded 등으로 `fallbackModel` 로 교체 후 재시도.
- **Prompt too long (413)** — 보류(`withheld`)해 두었다가 6단계에서 compact·collapse 로 복구.
- **Max output tokens** — 상한을 `ESCALATED_MAX_TOKENS` 로 올려 재시도.

### 6단계 — Tool 실행과 결과 병합

스트리밍 중에 이미 일부 tool 은 실행되기 시작했을 수 있습니다. 스트림이 끝나면 **남은 tool 들**을 모두 완료시키고 결과를 `toolResults` 에 모읍니다. `normalizeMessagesForAPI` 를 거쳐 user 타입 메시지만 추리는 것이 핵심 — tool_result 블록은 user 메시지 형태로 API 에 보내야 하기 때문입니다.

```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()          // 스트리밍 모드
  : runTools(toolUseBlocks, assistantMessages,           // 순차 모드
             canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message
    toolResults.push(
      ...normalizeMessagesForAPI(
        [update.message],
        toolUseContext.options.tools,
      ).filter(_ => _.type === 'user'),
    )
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

Tool 실행기 내부 규칙(동시성, abort 전파 등)은 [04_tool_execution.md](04_tool_execution.md) 참고.

### 7단계 — 종료 또는 Follow-up

한 turn 의 후처리는 "이 루프를 끝낼까, 한 번 더 돌릴까" 결정입니다.

1. **Prompt-too-long / media-size 보류(withheld)** → collapse drain 또는 reactive compact 를 시도해서 성공하면 해당 reason 으로 `continue`.
2. **Max output tokens 보류** → `maxOutputTokensOverride` 를 상향하고 `continue`.
3. **`needsFollowUp === true`** (이번 assistant 메시지에 tool_use 가 있었음) → tool_result 를 합친 새 messages 로 `continue`.
4. **그 외** → `return { reason: 'done' }` 으로 터미널 반환.

```typescript
if (needsFollowUp) {
  state = {
    messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
    toolUseContext: updatedToolUseContext,
    pendingToolUseSummary: nextPendingToolUseSummary,
    transition: { reason: 'follow_up_needed' },
    ...
  }
  continue
}

return { reason: 'done' }
```

---

## 한 장 요약

```
State(messages, turnCount, ...) ──┐
                                  │
  ┌───────────────────────────────▼─────────────────────────────┐
  │ 1. normalize (budget / snip / microcompact / collapse)      │
  │ 2. build full system prompt (+ systemContext)               │
  │ 3. autoCompact? ── continue ────────────────────────────┐   │
  │ 4. for await message of callModel(...):                 │   │
  │      • stream fallback → tombstone                      │   │
  │      • tool_use → StreamingToolExecutor.addTool()       │   │
  │      • yield message                                    │   │
  │ 5. catch FallbackTriggeredError → continue (with model) │   │
  │ 6. drain tools → toolResults                             │   │
  │ 7. withheld? → collapse/reactive compact → continue      │   │
  │    maxOutput? → escalate → continue                      │   │
  │    needsFollowUp? → continue                             │   │
  │    else → return { done }                                │   │
  └──────────────────────────┬──────────────────────────────┘   │
                             └──────────────────────────────────┘
```

---

## 다음 문서

- [03_api_layer.md](03_api_layer.md) — `callModel()` 이 실제로 Anthropic SDK 를 어떻게 호출하는가
