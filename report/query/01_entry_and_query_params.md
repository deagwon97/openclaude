# 01. 쿼리 진입점과 QueryParams

## 관련 파일

- [src/query.ts](../../src/query.ts) — `query()` 엔트리, `QueryParams`, `queryLoop()`
- [src/constants/querySource.js](../../src/constants/querySource.js) — `QuerySource` 열거형
- [src/screens/REPL.tsx](../../src/screens/REPL.tsx) — 메인 스레드 호출자
- [src/tools/AgentTool/runAgent.ts](../../src/tools/AgentTool/runAgent.ts) — 서브에이전트 호출자

---

## `query()` 함수 시그니처

`query()`는 AsyncGenerator 로 구현되어, 스트리밍 이벤트와 메시지를 차례로 `yield` 하며 최종적으로 `Terminal` 객체를 return 합니다. 내부적으로는 거의 모든 일을 `queryLoop()` 에 위임합니다.

```typescript
// src/query.ts
export type QueryParams = {
  messages: Message[]                      // 정규화된 대화 히스토리
  systemPrompt: SystemPrompt               // string[] — 블록 배열
  userContext: { [k: string]: string }     // CLAUDE.md, 날짜, 세션 메모리 등
  systemContext: { [k: string]: string }   // git status, platform, cwd 등
  canUseTool: CanUseToolFn                 // 권한 확인 훅
  toolUseContext: ToolUseContext           // 사용 가능 tool, abortController 등
  fallbackModel?: string                   // 529 overloaded 시 대체 모델
  querySource: QuerySource                 // 호출 컨텍스트 태깅
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }           // Agent 작업 예산
  deps?: QueryDeps                         // 테스트 주입용 의존성
}

export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

### 주요 파라미터 역할

| 필드 | 의미 |
|------|------|
| `messages` | 이미 정규화되어 API 로 바로 보낼 수 있는 메시지 배열 |
| `systemPrompt` | 캐시 블록 단위로 쪼갤 수 있도록 `string[]` 로 유지 (05 문서 참고) |
| `systemContext` | **시스템 프롬프트 뒤**에 키:값 한 줄씩 append 됨 |
| `userContext` | **첫 user 메시지 앞**에 `<system-reminder>` 블록으로 prepend 됨 |
| `toolUseContext` | 사용 가능한 tools, options, abortController, agentId 등을 담은 실행 컨텍스트 |
| `canUseTool` | tool 실행 직전에 호출되는 권한 훅 (거절 시 reject 메시지 삽입) |
| `querySource` | 어떤 주체가 이 쿼리를 시작했는지 식별. 재귀 방지 가드에 사용 |
| `taskBudget` | 서브에이전트의 총 토큰 예산. compact 시 남은 예산을 차감 |

---

## `QuerySource` 값과 용도

`querySource` 는 동일한 `query()` 함수가 다양한 컨텍스트에서 재사용되기 때문에 **누가 호출했는지**를 구분하는 태그입니다. AutoCompact / SessionMemory / Context Collapse 등 **재귀를 유발할 수 있는 경로**에서 재귀 가드로 사용됩니다.

대표적인 값:

| 값 | 의미 |
|----|------|
| `repl_main_thread` | REPL 의 메인 대화 루프 |
| `agent:<type>` | `AgentTool` 이 기동한 서브에이전트 (subagent_type 별로 구분) |
| `compact` | 컨버세이션 compaction 중 내부적으로 발생하는 쿼리 |
| `session_memory` | 세션 메모리 요약 생성 쿼리 |
| `marble_origami` | Context collapse 요약 쿼리 (feature flag) |

```typescript
// src/services/compact/autoCompact.ts — 재귀 가드 예시
if (querySource === 'session_memory' || querySource === 'compact') {
  return false
}
if (feature('CONTEXT_COLLAPSE') && querySource === 'marble_origami') {
  return false
}
```

---

## 호출자 (Callers)

`query()` 는 서로 다른 세 가지 계층에서 호출됩니다. 각자 다른 `querySource` / `toolUseContext` 를 구성하지만, 이후 실행 루프는 동일합니다.

1. **REPL 메인 스레드** — `repl_main_thread`
   - 사용자 프롬프트가 제출될 때마다 새 쿼리 시작
   - REPL UI 가 yield 된 `StreamEvent` / `Message` 를 받아 화면에 렌더
2. **AgentTool / runAgent** — `agent:<type>`
   - 부모 대화와는 **독립된 messages 배열과 시스템 프롬프트**로 새 `query()` 시작
   - yield 되는 메시지를 sidechain transcript 로 기록한 뒤 최종 텍스트만 부모로 반환
   - 자세한 내용은 [../agents/04_execution.md](../agents/04_execution.md) 참고
3. **내부 서브 프로세스** — `compact` / `session_memory` / `marble_origami`
   - compaction·요약 자체가 LLM 호출이므로 `query()` 를 재귀 호출
   - 이때 AutoCompact 는 반드시 비활성화되어야 하므로 querySource 로 재귀 가드

---

## `Terminal` 반환값

쿼리 루프가 종료될 때 돌아오는 객체로, 정상 종료 / 에러 / 중단 여부를 담습니다.

```typescript
type Terminal =
  | { reason: 'done' }
  | { reason: 'aborted' }
  | { reason: 'error', error: Error }
  | { reason: 'max_turns_reached' }
```

호출자는 이 값을 보고 상태를 갱신하거나 REPL 프롬프트를 다시 띄웁니다.

---

## 다음 문서

- [02_query_loop.md](02_query_loop.md) — `queryLoop()` 내부에서 어떤 단계가 어떤 순서로 진행되는지
