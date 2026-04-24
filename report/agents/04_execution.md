# 에이전트 실행 흐름 (Execution)

LLM이 `Agent` 도구를 호출한 이후 서브에이전트가 독립적인 대화 루프로 실행되는 과정입니다.

핵심 파일:
- [src/tools/AgentTool/AgentTool.tsx](../../src/tools/AgentTool/AgentTool.tsx)
- [src/tools/AgentTool/runAgent.ts](../../src/tools/AgentTool/runAgent.ts)
- [src/tools/AgentTool/agentToolUtils.ts](../../src/tools/AgentTool/agentToolUtils.ts)

---

## 1. 진입: `AgentTool.call()`

LLM이 다음과 같은 tool_use를 생성하면 진입합니다.

```json
{
  "name": "Agent",
  "input": {
    "subagent_type": "doc-finder",
    "prompt": "Find docs about context compression",
    "description": "Find context compression docs",
    "model": "sonnet",
    "run_in_background": false,
    "isolation": "worktree"
  }
}
```

[AgentTool.call()](../../src/tools/AgentTool/AgentTool.tsx)은 다음 순서로 실행됩니다.

1. **에이전트 선택** — `subagent_type`으로 `filteredAgents`에서 찾기. Fork 서브에이전트 경로(미지정 + feature flag)는 부모 컨텍스트를 상속하는 특수 경로로 분기
2. **격리 모드 결정** — `isolation` 파라미터 또는 에이전트 정의의 `isolation` 필드에 따라 worktree/remote 세팅
3. **도구 풀 조립** — `resolveAgentTools()`로 허용 도구 계산 ([05_permissions.md](05_permissions.md))
4. **실행 모드 결정** — 동기 vs 비동기 (백그라운드) 판단 ([06_async_and_isolation.md](06_async_and_isolation.md))
5. **`runAgent()` 호출** — AsyncGenerator로 메시지를 순차 yield
6. **결과 수집 및 반환** — 최종 텍스트 블록을 tool_result로 포장

---

## 2. `runAgent()` 내부

`runAgent()`은 `async function*`(AsyncGenerator)로, 서브에이전트가 생성하는 모든 메시지를 호출자에게 순차적으로 흘려보냅니다.

### 2.1 시스템 프롬프트 구성

```typescript
// runAgent.ts
const agentSystemPrompt = override?.systemPrompt
  ?? asSystemPrompt(
    await getAgentSystemPrompt(
      agentDefinition,
      toolUseContext,
      resolvedAgentModel,
      additionalWorkingDirectories,
      resolvedTools,
    ),
  )
```

`getAgentSystemPrompt()`는 두 가지를 합칩니다.

1. **에이전트 본문** — `agentDefinition.getSystemPrompt({ toolUseContext })` (마크다운 본문 또는 `prompt` 필드)
2. **환경 상세** — `enhanceSystemPromptWithEnvDetails()`가 현재 작업 디렉토리, 활성 도구 이름, 추가 working directory 등을 덧붙임

`memory`가 지정된 에이전트는 이후에 메모리 지시사항도 연결됩니다.

### 2.2 User/System Context

`runAgent()`은 부모의 context를 그대로 쓰지 않고 서브에이전트용으로 재구성합니다.

```typescript
const [baseUserContext, baseSystemContext] = await Promise.all([
  override?.userContext ?? getUserContext(),
  override?.systemContext ?? getSystemContext(),
])
```

메모리 최적화 플래그(`tengu_slim_subagent_claudemd`)가 켜져 있고 에이전트가 `omitClaudeMd: true`이면 `CLAUDE.md`와 `gitStatus`를 빼 컨텍스트 크기를 줄입니다. Explore/Plan 같은 읽기 중심 내장 에이전트에 주로 적용됩니다.

### 2.3 권한 모드 오버라이드

에이전트 정의가 `permissionMode`를 지정하면 서브 컨텍스트에 반영됩니다. 단, 부모가 이미 `bypassPermissions`/`acceptEdits`인 경우에는 덮어쓰지 않습니다 (더 안전한 모드로만 전환 가능).

비동기(백그라운드) 에이전트는 권한 프롬프트를 띄울 방법이 없으므로, 기본값이 "자동 거부"로 설정됩니다.

### 2.4 MCP 서버 초기화

```typescript
const { clients, tools: agentMcpTools, cleanup: mcpCleanup } =
  await initializeAgentMcpServers(
    agentDefinition,
    toolUseContext.options.mcpClients,
  )
```

에이전트 정의의 `mcpServers`에 선언된 서버를 연결하고, 새 클라이언트와 도구 스키마를 얻습니다. 부모가 이미 가진 클라이언트는 재사용하며, `cleanup()`은 새로 만든 것만 닫습니다.

### 2.5 Subagent Context 생성

부모 컨텍스트를 복제·격리하여 서브에이전트만의 실행 환경을 만듭니다.

```typescript
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  agentType: agentDefinition.agentType,
  messages: initialMessages,
  readFileState: agentReadFileState,       // 부모 파일 상태 캐시를 클론
  abortController: agentAbortController,   // 부모와 분리된 중단 컨트롤러
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,              // 비동기 에이전트는 앱 상태 비공유
  shareSetResponseLength: true,
})
```

핵심은 **AbortController 분리**입니다. 서브에이전트가 도중에 실패해도 부모 세션은 그대로 유지되며, 반대로 부모가 중단되면 서브에이전트도 신호를 받습니다.

---

## 3. 대화 루프

서브에이전트는 일반적인 `query()` 루프를 한 번 더 돌립니다. 즉 OpenClaude의 같은 쿼리 엔진이 재귀적으로 사용됩니다.

```typescript
for await (const message of query({
  messages: initialMessages,
  systemPrompt: agentSystemPrompt,
  userContext: resolvedUserContext,
  systemContext: resolvedSystemContext,
  canUseTool,
  toolUseContext: agentToolUseContext,
  querySource,
  maxTurns: maxTurns ?? agentDefinition.maxTurns,
})) {
  if (isRecordableMessage(message)) {
    await recordSidechainTranscript([message], agentId, lastRecordedUuid)
    yield message
  }
}
```

모든 메시지는 두 가지 대상에 기록됩니다.

- **sidechain transcript**: `subagents/{agentId}/`에 JSONL로 영속 저장. `transcriptSubdir`가 지정되면 해당 디렉토리로 그룹화
- **yield**: 상위 `AgentTool.call()`이 진행 UI에 표시

쿼리 엔진 자체에 대한 상세 설명은 [../query/query-loop.md](../query/query-loop.md) 참고.

---

## 4. 부모 ↔ 자식 메시지 전달

### 4.1 부모 → 자식

| 경로 | 초기 메시지 |
|------|-------------|
| 일반 | 사용자 메시지 1개 — `createUserMessage({ content: prompt })` |
| Fork (`isForkPath`) | 부모의 전체 대화 이력 (`forkContextMessages`) 포함 |

Fork 경로는 [forkSubagent.ts](../../src/tools/AgentTool/forkSubagent.ts)에서 별도로 처리되며, 주로 부모 컨텍스트를 유지해야 하는 재귀적 reasoning 작업에 사용됩니다.

### 4.2 자식 → 부모

동기 실행의 경우, 서브에이전트가 yield한 마지막 assistant 메시지의 텍스트 블록이 `tool_result`의 본문이 됩니다. 진행 중 메시지는 UI에 실시간으로 흐르지만, 최종 결과는 텍스트 1개로 압축됩니다.

비동기의 경우에는 결과가 파일로 저장되고 `task_id`만 반환됩니다 ([06_async_and_isolation.md](06_async_and_isolation.md)).

---

## 5. 정리(Cleanup)

`runAgent()`의 `finally` 블록에서 다음을 수행합니다.

1. **MCP cleanup** — `mcpCleanup()`로 새로 만든 MCP 클라이언트만 닫음
2. **세션 훅 정리** — `agentDefinition.hooks`가 있었다면 `clearSessionHooks(rootSetAppState, agentId)`
3. **프롬프트 캐시 추적 정리** — `cleanupAgentTracking(agentId)` (기능 플래그)
4. **파일 상태 캐시 해제** — `agentToolUseContext.readFileState.clear()`
5. **초기 메시지 배열 비우기** — 메모리 해제
6. **Bash 백그라운드 작업 종료** — `killShellTasksForAgent(agentId, ...)`로 서브에이전트가 띄운 백그라운드 셸 태스크 정리

서브에이전트 수명의 모든 리소스가 이 블록에서 회수되도록 설계되어 있으며, 이는 부모 세션에 부작용이 남지 않게 하려는 핵심 방어선입니다.

---

## 6. 최종 결과 형태

`finalizeAgentTool()`이 반환하는 구조:

```typescript
{
  agentId: string,
  agentType?: string,
  content: Array<{ type: 'text', text: string }>,
  totalToolUseCount: number,
  totalDurationMs: number,
  totalTokens: number,
  usage: { input_tokens, output_tokens, ... }
}
```

부모는 이를 일반 `tool_result`로 감싸 다음 대화 턴에 투입합니다. LLM 입장에서는 "Agent를 한 번 호출했더니 긴 텍스트가 돌아왔다"로 보이며, 서브에이전트 내부의 모든 tool_use는 sidechain transcript에만 남고 부모 컨텍스트에는 노출되지 않습니다.

---

## 7. 실행 흐름 다이어그램

```
LLM: Agent tool_use
        │
        ▼
AgentTool.call()
 ├─ 에이전트 선택
 ├─ isolation 결정
 ├─ resolveAgentTools() → 허용 도구 계산
 ├─ 동기/비동기 판단
 └─ runAgent() 호출
        │
        ▼
runAgent() (async generator)
 ├─ getAgentSystemPrompt()  ──→ 본문 + env details
 ├─ userContext / systemContext 구성
 ├─ permissionMode 오버라이드
 ├─ initializeAgentMcpServers()
 ├─ createSubagentContext()  ──→ 독립 AbortController, clone readFileState
 │
 │   ┌────────── query() 루프 ──────────┐
 │   │  LLM 호출 → tool_use → tool 실행   │
 │   │  → tool_result → 다음 턴 ...      │
 │   │  (maxTurns 까지 반복)             │
 │   └─────────────┬───────────────────┘
 │                 │  메시지마다 yield
 │                 ▼
 │          sidechain transcript 기록
 │                 │
 └─ finally:
    ├─ mcpCleanup()
    ├─ clearSessionHooks()
    ├─ readFileState.clear()
    └─ killShellTasksForAgent()
        │
        ▼
finalizeAgentTool()
        │
        ▼
tool_result → 부모 LLM 컨텍스트
```
