# 04. Tool 실행 (`StreamingToolExecutor`)

## 관련 파일

- [src/services/tools/StreamingToolExecutor.ts](../../src/services/tools/StreamingToolExecutor.ts) — 실행 엔진
- [src/query.ts](../../src/query.ts) — `runTools` 폴백 경로와 실행기 소비자
- [src/constants/tools.ts](../../src/constants/tools.ts) — `BASH_TOOL_NAME` 등 단독 실행 대상 상수

---

## 왜 "Streaming" Tool Executor 인가

과거 경로는 assistant 메시지가 **완전히 수신된 후**에 tool 을 한꺼번에 실행하는 방식(`runTools`)이었습니다. 스트리밍 실행기는 한 단계 앞당겨서, **스트림 도중 `tool_use` 블록이 완성될 때마다** 즉시 실행 큐에 넣습니다. 그 결과:

- I/O-bound tool (Glob, Grep, WebFetch 등) 이 모델 토큰 생성과 병렬로 돈다
- 모델이 여러 tool 을 한 turn 에 호출하면 동시성 안전한 것들은 나란히 실행된다
- 결과는 `getCompletedResults()` 가 **non-blocking** 으로 yield 하므로, 이미 끝난 tool 은 스트림 도중에도 상위로 흘러간다

폴백 경로(`runTools`)는 스트리밍이 비활성화되었거나 실행기가 에러로 폐기된 경우에만 사용됩니다.

---

## 내부 상태

```typescript
// src/services/tools/StreamingToolExecutor.ts
type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'

type TrackedTool = {
  id: string
  block: ToolUseBlock
  assistantMessage: AssistantMessage
  status: ToolStatus
  isConcurrencySafe: boolean
  promise?: Promise<void>
  results?: Message[]
  contextModifiers?: ContextModifier[]
  pendingProgress: ProgressUpdate[]
}

export class StreamingToolExecutor {
  private tools: TrackedTool[] = []
  private hasErrored = false
  private discarded = false
  private siblingAbortController: AbortController

  constructor(
    private readonly toolDefinitions: Tools,
    private readonly canUseTool: CanUseToolFn,
    toolUseContext: ToolUseContext,
  ) {
    this.toolUseContext = toolUseContext
    this.siblingAbortController = createChildAbortController(
      toolUseContext.abortController,
    )
  }
}
```

`siblingAbortController` 는 이 turn 의 tool 들끼리만 공유하는 자식 abort 신호입니다. 어느 한 tool 이 실패하면 **형제 tool 만** 취소하고, 부모(쿼리 전체 abort)는 건드리지 않습니다.

---

## 큐잉과 실행 스케줄

`addTool` 은 스트리밍 중 호출되어 tool 을 큐에 넣고 가능한 즉시 실행을 시작시킵니다.

```typescript
addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
  const toolDefinition = findToolByName(this.toolDefinitions, block.name)
  if (!toolDefinition) {
    this.tools.push({ ..., results: [createUserMessage(/* not found */)] })
    return
  }

  const isConcurrencySafe = toolDefinition.concurrencySafe !== false
  this.tools.push({
    id: block.id,
    block,
    assistantMessage,
    status: 'queued',
    isConcurrencySafe,
    pendingProgress: [],
  })
  void this.startToolsIfReady()
}
```

실행 스케줄 규칙:

```typescript
private async startToolsIfReady(): Promise<void> {
  const queued = this.tools.filter(t => t.status === 'queued')
  const executing = this.tools.filter(t => t.status === 'executing')

  // 규칙 1: Bash 가 돌고 있으면 다른 tool 은 멈춘다 (부작용 격리)
  if (executing.some(t => t.block.name === BASH_TOOL_NAME)) return

  // 규칙 2: 비동시안전 tool 이 돌고 있으면 다른 tool 은 멈춘다
  if (executing.some(t => !t.isConcurrencySafe)) return

  for (const tool of queued) {
    if (!tool.isConcurrencySafe && executing.length > 0) continue
    tool.status = 'executing'
    tool.promise = this.executeTool(tool)
  }
}
```

요약하면:
- **Bash** 는 단독 실행 (다른 tool 과 절대 겹치지 않음)
- `concurrencySafe === false` 인 tool 도 단독 실행
- 나머지는 서로 병렬 가능

---

## 실제 실행 경로

```typescript
private async executeTool(tool: TrackedTool): Promise<void> {
  // 1) 이 시점에 이미 abort 되었나? (사용자 취소 / 형제 실패)
  const abortReason = this.getAbortReason(tool)
  if (abortReason !== null) {
    tool.results = [this.createCancelledToolResult(tool, abortReason)]
    tool.status = 'completed'
    return
  }

  const definition = findToolByName(this.toolDefinitions, tool.block.name)
  if (!definition) {
    tool.results = [ /* not found error */ ]
    tool.status = 'completed'
    return
  }

  try {
    // 2) 권한 훅 — 사용자가 거절하면 reject 메시지로 대체
    const canUse = await this.canUseTool(definition)
    if (!canUse) {
      tool.results = [REJECT_MESSAGE]
      tool.status = 'completed'
      return
    }

    // 3) 실제 tool 본체 실행
    const messages = await runToolUse(
      definition,
      tool.block,
      this.toolUseContext,
      this.siblingAbortController,   // 실패 시 형제 tool 에 abort 전파
    )
    tool.results = messages

    if (definition.contextModifier) {
      tool.contextModifiers = [definition.contextModifier]
    }
  } catch (error) {
    this.hasErrored = true
    this.erroredToolDescription = this.getToolDescription(tool)
    tool.results = [ /* error result */ ]

    // 형제 tool 들을 모두 취소한다 (이번 turn 한정)
    this.siblingAbortController.abort()
  } finally {
    tool.status = 'completed'
  }
}
```

`contextModifier` 가 있으면 tool 이 `ToolUseContext` 자체를 수정할 수 있습니다 (예: 새로 발견한 워킹 디렉토리를 등록). 수정자는 결과와 함께 `MessageUpdate.newContext` 로 상위에 전달됩니다.

---

## 결과 소비 API

실행기는 두 가지 방식으로 결과를 건네줍니다.

```typescript
// 스트리밍 중 — 이미 완료된 것만 non-blocking 으로 yield
*getCompletedResults(): Generator<MessageUpdate, void> {
  for (const tool of this.tools) {
    if (tool.status === 'yielded') continue
    if (tool.status === 'completed' && tool.results) {
      tool.status = 'yielded'
      for (const result of tool.results) {
        yield { message: result, newContext: tool.contextModifiers }
      }
    }
  }
}

// 스트리밍 종료 후 — 남은 tool 의 promise 를 기다린 뒤 순서대로 yield
async *getRemainingResults(): AsyncGenerator<MessageUpdate, void> {
  const remaining = this.tools.filter(t => t.status !== 'yielded')
  for (const tool of remaining) {
    if (tool.promise) await tool.promise
  }
  for (const result of this.getCompletedResults()) yield result
}
```

`queryLoop` 는 스트리밍 중 `getCompletedResults()` 를 주기적으로 훑고, 스트림이 닫히면 `getRemainingResults()` 로 마무리합니다.

---

## 결과의 다음 행선지

실행기에서 나온 `update.message` 는 그대로 한 번 `yield` 되어 REPL 에 표시되고, **동시에** `normalizeMessagesForAPI` 를 거쳐 `toolResults` 버퍼에 누적됩니다. 한 turn 이 끝나고 follow-up 이 필요하면 다음 반복의 `messagesForQuery` 에 `[...이전 messages, ...assistantMessages, ...toolResults]` 로 합쳐져 API 로 재전송됩니다.

```typescript
for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message
    toolResults.push(
      ...normalizeMessagesForAPI([update.message], toolUseContext.options.tools)
         .filter(_ => _.type === 'user'),
  )
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

여기서 `filter(_ => _.type === 'user')` 가 중요합니다 — Claude API 에서 `tool_result` 블록은 user 메시지 content 의 일부이므로 user 타입만 통과시켜야 합니다.

---

## 폐기 (Discard) 경로

스트리밍 실패로 `FallbackTriggeredError` 가 발생해 모델을 교체해야 할 때, 지금까지 쌓인 실행기 상태는 의미가 없어집니다. 이때 `discard()` 로 abort 신호를 보내고 새 인스턴스를 생성합니다.

```typescript
// queryLoop catch 블록 안
if (streamingToolExecutor) {
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

---

## 다음 문서

- [05_system_prompt.md](05_system_prompt.md) — 시스템 프롬프트가 어떤 블록들로 조립되고 어디서 캐시 경계가 생기는지
