# 03. 실행 엔진

## 1. 메인 진입점: `executeHooks()`

핵심 구현은 [src/utils/hooks.ts](../../src/utils/hooks.ts)의 `executeHooks()`이며, 다음 공개 함수들이 이를 래핑한다.

```
executePreToolHooks(toolName, toolInput, …)
executePostToolHooks(toolName, toolInput, toolResponse, …)
executePostToolUseFailureHooks(toolName, error, …)
executeUserPromptSubmitHooks(prompt, …)
executeSessionStartHooks(source, …)
executeSessionEndHooks(reason, …)
executeStopHooks(…)
executeStopFailureHooks(error, …)
executeSubagentStartHooks(agentId, …)
executeSubagentStopHooks(agentId, transcriptPath, …)
executePreCompactHooks(trigger, …)
executePostCompactHooks(trigger, summary, …)
executeNotificationHooks(message, …)
executePermissionDeniedHooks(toolName, reason, …)
executeSetupHooks(trigger, …)
```

모든 래퍼는 동일하게 `executeHooks(eventName, baseInput, options)`으로 수렴한다.

## 2. 반환 타입: Async Generator

`executeHooks()`는 **async generator**를 반환해 hook 결과를 **스트리밍**한다.

```typescript
async function* executeHooks(event, input, options) {
  // 1. 후보 hook 선정 (02_loading_and_matching.md 참조)
  const hooks = selectHooks(event, input)

  // 2. fast path: 모든 hook이 callback/function 타입이면 직접 호출
  if (allCallbackOrFunction(hooks)) {
    for (const h of hooks) await h.callback(input)
    return
  }

  // 3. slow path: span 이벤트 방출 + 병렬 실행
  emitHookStarted({ event, count: hooks.length })
  for await (const result of runHooksInParallel(hooks, input, options)) {
    yield result   // 스트림으로 흘려보냄
  }
  emitHookFinished()
}
```

- **Fast path**: SDK 내부에서 등록한 콜백만 있으면 abortSignal·span·JSON 파싱을 전부 스킵한다 (측정 6µs → 1.8µs, −70%).
- **Slow path**: 외부 command/http/agent/prompt hook이 섞이면 풀 파이프라인으로 진입한다.

## 3. 병렬 실행

한 이벤트에 걸린 여러 hook은 `Promise.all`류 패턴으로 **동시에** spawn되며, 각 hook은 자신의 타임아웃·AbortSignal을 독립적으로 적용받는다. 결과는 도착하는 순서대로 generator에 yield되지만, aggregator는 최종적으로 모든 결과를 모아서 하나의 `AggregatedHookResult`로 합친다.

```
executeHooks (async gen)
 ├── hook A ─┐
 ├── hook B ─┼── 병렬 spawn
 └── hook C ─┘
            ↓
      yield 개별 결과
            ↓
     aggregateResults()
            ↓
   AggregatedHookResult
```

## 4. AggregatedHookResult 구조

여러 hook의 결과를 하나로 합친 형태:

```typescript
type AggregatedHookResult = {
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'

  // 첫 blocking hook의 에러 (뒤의 것들은 additionalContexts로 흡수)
  blockingError?: { blockingError: string, command: string }

  // 여러 hook이 주입한 컨텍스트 누적
  additionalContexts: string[]

  // Tool input 수정 (PreToolUse만)
  updatedInput?: Record<string, unknown>

  // MCP 툴 출력 수정 (PostToolUse만)
  updatedMCPToolOutput?: unknown

  // 권한 결정
  permissionBehavior?: 'allow' | 'deny' | 'ask'

  // 모델 호출 중단
  preventContinuation?: boolean
  stopReason?: string

  // hook이 사용자에게 보낸 시스템 메시지
  systemMessages: string[]
}
```

`outcome` 우선순위: `blocking > non_blocking_error > cancelled > success`.

## 5. Tool 실행 경로와의 통합

Tool 실행 시 통합 지점은 [src/services/tools/toolHooks.ts](../../src/services/tools/toolHooks.ts):

```
runPreToolUseHooks(tool, input, ctx)
  ├── executePreToolHooks(toolName, input, …)
  ├── result.permissionBehavior === 'deny' → tool call 차단
  ├── result.updatedInput → 이후 tool 호출이 사용할 input 교체
  └── result.additionalContexts → 모델 시스템 메시지로 주입

toolExecutor.execute(tool, (maybe updated) input)
  │
  ├── 성공 → runPostToolUseHooks(tool, input, response, ctx)
  │           ├── executePostToolHooks(…)
  │           ├── isMcpTool(tool) && updatedMCPToolOutput → 응답 교체
  │           └── additionalContexts → 모델에 주입
  │
  └── 실패 → runPostToolUseFailureHooks(tool, error, ctx)
              └── executePostToolUseFailureHooks(…)
```

## 6. UserPromptSubmit 통합

[src/utils/processUserInput/processUserInput.ts](../../src/utils/processUserInput/processUserInput.ts)에서:

1. 사용자가 프롬프트를 제출하면 processUserInput 호출
2. `executeUserPromptSubmitHooks(prompt)` 실행
3. 결과에 `blockingError`가 있으면 원본 프롬프트를 버리고 stderr을 사용자에게 표시
4. `additionalContext`가 있으면 프롬프트에 attachment로 붙여 모델에 전달
5. 그 외에는 정상 플로우 계속

## 7. SDK 이벤트 방출

[src/utils/hooks/hookEvents.ts](../../src/utils/hooks/hookEvents.ts)는 SDK 소비자를 위한 span/progress 이벤트를 방출한다.

```
emitHookStarted({ event, count })
emitHookProgress({ event, hookName, stdoutChunk })
emitHookResponse({ event, hookName, result })
emitHookFinished({ event, outcome })
```

SDK 사용자는 이 이벤트를 구독해 hook 실행 상황을 UI에 표시하거나 메트릭을 수집할 수 있다.

## 8. 비동기 hook 등록 (간단)

`hook.async: true` 또는 `hook.asyncRewake: true`이면 slow path에서 spawn된 프로세스가 foreground 파이프라인을 블록하지 않는다. 대신 [AsyncHookRegistry](../../src/utils/hooks/AsyncHookRegistry.ts)에 등록되어 백그라운드로 계속 실행된다. 상세는 [06_timeout_and_errors.md](06_timeout_and_errors.md) §4를 참조.
