# 에이전트 등록 & System Prompt 주입 (Registration)

로드된 에이전트들이 `AgentTool`을 통해 LLM에게 "사용 가능한 서브에이전트 목록"으로 노출되는 과정입니다.

핵심 파일:
- [src/tools/AgentTool/AgentTool.tsx](../../src/tools/AgentTool/AgentTool.tsx)
- [src/tools/AgentTool/prompt.ts](../../src/tools/AgentTool/prompt.ts)

---

## 1. AgentTool의 prompt() 훅

OpenClaude의 모든 tool은 LLM에게 자신을 어떻게 설명할지 결정하는 `prompt()` 훅을 가집니다. `AgentTool.prompt()`는 호출 시점마다 다음 단계를 거쳐 현재 세션에서 사용 가능한 에이전트를 선별합니다.

```typescript
// src/tools/AgentTool/AgentTool.tsx 의 prompt() 훅
async prompt({
  agents,                    // getAgentDefinitionsWithOverrides() 결과
  tools,                     // 현재 세션의 도구 풀
  getToolPermissionContext,
  allowedAgentTypes,         // 상위에서 제한한 경우만 지정
}) {
  // 1) MCP 서버 요구 조건을 만족하는 에이전트만 남김
  const agentsWithMcpRequirementsMet =
    filterAgentsByMcpRequirements(agents, mcpServersWithTools)

  // 2) 권한 규칙에 의해 차단된 에이전트 제거
  const filteredAgents = filterDeniedAgents(
    agentsWithMcpRequirementsMet,
    toolPermissionContext,
    AGENT_TOOL_NAME,
  )

  // 3) 프롬프트 텍스트 생성
  return await getPrompt(filteredAgents, isCoordinator, allowedAgentTypes)
}
```

이 훅이 반환한 문자열은 AgentTool의 `description`으로 사용되며, 결과적으로 LLM의 tool 스키마에 포함됩니다.

---

## 2. 필터링 단계

### 2.1 MCP 요구 조건

에이전트 정의에 `mcpServers` 필드가 있으면, 해당 MCP 서버가 실제로 연결되어 있어야만 에이전트가 노출됩니다. `filterAgentsByMcpRequirements()`가 담당합니다.

### 2.2 권한 규칙

`filterDeniedAgents()`는 `ToolPermissionContext`의 deny 규칙을 확인하여, `Agent(foo)` 형태의 규칙으로 막힌 에이전트 타입을 제외합니다. 즉 사용자가 특정 서브에이전트 사용을 금지했다면 LLM에게 아예 노출되지 않습니다.

### 2.3 `allowedAgentTypes` 제한

상위 context에서 허용 목록을 강제한 경우(예: coordinator 모드), 그 목록 밖의 에이전트는 제외됩니다.

---

## 3. 프롬프트 텍스트 포맷

필터링된 에이전트는 한 줄씩 포맷팅되어 AgentTool 설명에 들어갑니다. [prompt.ts](../../src/tools/AgentTool/prompt.ts) 기준 포맷:

```
- {agentType}: {whenToUse} (Tools: {toolsDescription})
```

예시 (실제 출력):

```
- general-purpose: General-purpose agent for researching complex questions... (Tools: *)
- Explore: Fast agent specialized for exploring codebases... (Tools: All tools except Agent, ExitPlanMode, Edit, Write, NotebookEdit)
- doc-finder: Finds relevant documentation files in the repo (Tools: Glob, Grep, Read)
```

`toolsDescription`은 에이전트의 `tools` 필드에 따라 다음처럼 축약됩니다.

| 정의 상태 | 표시 |
|----------|------|
| 미정의 / `'*'` | `*` (또는 `All tools`) |
| 구체적 목록 | 콤마로 연결된 도구 이름 |
| 빈 배열 | `(none)` |
| 내장 에이전트 | 추가로 "except ..." 형태의 제외 목록을 명시 |

---

## 4. System Prompt에 주입하는 두 가지 경로

에이전트 목록을 LLM이 볼 수 있게 하는 방법은 두 가지이며, [shouldInjectAgentListInMessages()](../../src/tools/AgentTool/prompt.ts)에 따라 결정됩니다 (기본값: attachment 경로).

### 4.1 Tool description 경로 (기본값 off)

`AgentTool.prompt()`의 반환 문자열이 tool description으로 들어가 tool 스키마의 일부가 됩니다. 단점: 에이전트 목록이 변할 때마다 tool 정의가 바뀌어 **프롬프트 캐시가 무효화**됩니다.

### 4.2 Attachment 경로 (기본값 on)

GrowthBook 플래그 `tengu_agent_list_attach`가 켜져 있을 때 사용됩니다. 에이전트 목록은 별도의 `agent_listing_delta` attachment로 사용자 메시지 쪽에 삽입되고, tool description은 정적인 문구로 고정됩니다. 결과적으로 tool 정의가 불변이 되어 프롬프트 캐시가 유지됩니다.

---

## 5. 등록 흐름 다이어그램

```
activeAgents  (02_loading.md의 결과)
       │
       ▼
┌───────────────────────────────┐
│  AgentTool.prompt() 훅 호출   │
├───────────────────────────────┤
│ 1. filterAgentsByMcpReq...    │
│ 2. filterDeniedAgents          │
│ 3. allowedAgentTypes 제한      │
└────────────┬──────────────────┘
             │
             ▼
      getPrompt() → 텍스트
             │
     ┌───────┴────────┐
     ▼                ▼
 [tool desc]     [attachment]
     │                │
     └───────┬────────┘
             ▼
      LLM 컨텍스트에 포함
             │
             ▼
   LLM이 Agent tool을
   subagent_type 지정해 호출
             │
             ▼
     AgentTool.call()  (→ 04_execution.md)
```

---

## 6. 디버깅 팁

- 특정 에이전트가 LLM에 보이지 않는다면 다음 순서로 확인:
  1. [02_loading.md](02_loading.md)의 파싱 단계에서 누락되었는지 (Zod 검증 실패)
  2. `mcpServers` 조건을 만족하지 않는지
  3. deny 규칙(`Agent(foo)`)이 걸려 있지 않은지
- `--dump-system-prompt` 플래그로 실제 LLM에게 전달되는 프롬프트를 확인 가능 (에이전트 목록 포함 여부 점검)
