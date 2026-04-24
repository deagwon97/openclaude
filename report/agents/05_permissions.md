# 도구 필터링 & 권한 (Permissions)

서브에이전트가 실제로 사용할 수 있는 도구 집합이 결정되는 과정입니다. 여러 단계의 필터가 중첩되어 있어, 에이전트 정의에서 허용한 도구가 런타임에는 차단되는 경우가 있습니다.

핵심 파일:
- [src/tools/AgentTool/agentToolUtils.ts](../../src/tools/AgentTool/agentToolUtils.ts)
- [src/constants/tools.ts](../../src/constants/tools.ts)

---

## 1. 필터 파이프라인 개요

```
[부모 세션 도구 풀]
       │
       ▼
filterToolsForAgent()
 ├─ MCP 도구는 통과
 ├─ Plan 모드에선 ExitPlanMode 통과
 ├─ ALL_AGENT_DISALLOWED_TOOLS 차단
 ├─ 커스텀 에이전트면 CUSTOM_AGENT_DISALLOWED_TOOLS 차단
 ├─ 비동기면 ASYNC_AGENT_ALLOWED_TOOLS 화이트리스트
 └─ 프로세스 내 팀 멤버 예외
       │
       ▼
에이전트 정의의 tools 필드 교집합
       │
       ▼
에이전트 정의의 disallowedTools 차집합
       │
       ▼
resolvedTools  (runAgent()로 전달)
```

각 단계는 [resolveAgentTools()](../../src/tools/AgentTool/agentToolUtils.ts)에서 순차적으로 적용됩니다.

---

## 2. 글로벌 차단 집합

정의 위치: [src/constants/tools.ts](../../src/constants/tools.ts)

### 2.1 `ALL_AGENT_DISALLOWED_TOOLS` (tools.ts:36-46)

모든 서브에이전트(내장 포함)에게 차단되는 도구.

| 도구 | 차단 이유 |
|------|----------|
| `TaskOutput` | 백그라운드 태스크 루프 방지 |
| `ExitPlanMode` / `EnterPlanMode` | Plan 모드는 메인 스레드 추상화 |
| `AskUserQuestion` | 서브에이전트는 사용자와 직접 대화 불가 |
| `TaskStop` | 메인 스레드 태스크 상태 필요 |
| `Agent` (조건부) | `USER_TYPE !== 'ant'`일 때 중첩 에이전트 금지 |
| `Workflow` (조건부) | `WORKFLOW_SCRIPTS` 기능일 때 재귀 실행 방지 |

```typescript
export const ALL_AGENT_DISALLOWED_TOOLS = new Set([
  TASK_OUTPUT_TOOL_NAME,
  EXIT_PLAN_MODE_V2_TOOL_NAME,
  ENTER_PLAN_MODE_TOOL_NAME,
  ...(process.env.USER_TYPE === 'ant' ? [] : [AGENT_TOOL_NAME]),
  ASK_USER_QUESTION_TOOL_NAME,
  TASK_STOP_TOOL_NAME,
  ...(feature('WORKFLOW_SCRIPTS') ? [WORKFLOW_TOOL_NAME] : []),
])
```

> 공통 패턴: **서브에이전트에 들어간 순간 "메인 스레드 전용 기능"과 "재귀 유발 가능 도구"는 차단된다.**

### 2.2 `CUSTOM_AGENT_DISALLOWED_TOOLS` (tools.ts:48-50)

커스텀 에이전트(사용자 정의)에게만 추가로 적용되는 차단 집합. 현재는 `ALL_AGENT_DISALLOWED_TOOLS`를 그대로 포함하며, 내장 에이전트보다 더 엄격하게 제한하고 싶은 도구가 생기면 여기에 추가됩니다.

```typescript
export const CUSTOM_AGENT_DISALLOWED_TOOLS = new Set([
  ...ALL_AGENT_DISALLOWED_TOOLS,
])
```

### 2.3 `ASYNC_AGENT_ALLOWED_TOOLS` (tools.ts:55-71)

비동기(백그라운드) 에이전트용 **화이트리스트**. 동기 에이전트는 블랙리스트 방식이지만, 비동기 에이전트는 여기에 명시된 도구만 사용할 수 있습니다.

```typescript
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
  SKILL_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
  TOOL_SEARCH_TOOL_NAME,
  ENTER_WORKTREE_TOOL_NAME,
  EXIT_WORKTREE_TOOL_NAME,
])
```

백그라운드 에이전트가 권한 프롬프트를 띄울 수 없기 때문에, 부작용이 큰 도구(예: 사용자 상호작용, 원격 트리거)를 처음부터 제거하는 방어선입니다.

### 2.4 `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` (tools.ts:77-88)

Agent Swarms 기능의 "팀 멤버" 에이전트에게만 주입되는 도구들. `TaskCreate/Get/List/Update`, `SendMessage`, 그리고 조건부로 Cron 도구들이 포함됩니다.

---

## 3. `filterToolsForAgent()` 상세

[agentToolUtils.ts](../../src/tools/AgentTool/agentToolUtils.ts)의 `filterToolsForAgent()`는 위 집합들을 실제로 적용합니다.

```typescript
export function filterToolsForAgent({
  tools,
  isBuiltIn,
  isAsync = false,
  permissionMode,
}): Tools {
  return tools.filter(tool => {
    // 1) MCP 도구는 무조건 통과
    if (tool.name.startsWith('mcp__')) return true

    // 2) Plan 모드 + ExitPlanMode 는 예외적으로 허용
    if (toolMatchesName(tool, EXIT_PLAN_MODE_V2_TOOL_NAME) &&
        permissionMode === 'plan') return true

    // 3) 전역 차단
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false

    // 4) 커스텀 에이전트 추가 차단
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false

    // 5) 비동기 에이전트 화이트리스트
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) {
      // 프로세스 내 팀 멤버는 예외
      if (isAgentSwarmsEnabled() && isInProcessTeammate()) {
        if (toolMatchesName(tool, AGENT_TOOL_NAME)) return true
        if (IN_PROCESS_TEAMMATE_ALLOWED_TOOLS.has(tool.name)) return true
      }
      return false
    }

    return true
  })
}
```

---

## 4. 에이전트 정의 필드 적용

글로벌 필터 통과 이후, 에이전트 정의의 `tools` / `disallowedTools`가 추가로 적용됩니다.

| 단계 | 동작 |
|------|------|
| `tools` 미정의 / `['*']` | 전 단계 결과 그대로 통과 |
| `tools` 구체적 목록 | 해당 도구만 남김 (교집합) |
| `disallowedTools` 목록 | 해당 도구 제거 (차집합) |

즉 순서상 **글로벌 차단 → 에이전트 화이트리스트 → 에이전트 블랙리스트** 순으로 적용되어, 가장 좁은 집합이 남습니다.

---

## 5. 권한 모드 오버라이드

에이전트 정의의 `permissionMode`는 도구 목록이 아닌 **권한 판정 방식**을 바꿉니다.

| 모드 | 의미 |
|------|------|
| `plan` | 읽기 전용 모드. 수정 도구는 권한 요청 단계에서 거부 |
| `acceptEdits` | 편집 자동 승인 (프롬프트 생략) |
| `bypassPermissions` | 모든 권한 프롬프트 생략 |

[runAgent.ts](../../src/tools/AgentTool/runAgent.ts)의 권한 모드 적용 로직은 **더 안전한 방향으로만 바꿀 수 있습니다**. 부모가 `bypassPermissions`인 상태에서 자식이 `plan`으로 전환하는 것은 허용되지만, 반대 방향은 무시됩니다.

비동기 에이전트는 권한 프롬프트를 표시할 UI가 없으므로, `canShowPermissionPrompts`가 기본 false로 세팅되어 모든 프롬프트가 자동 거부됩니다.

---

## 6. 디버깅: "왜 내 에이전트가 이 도구를 못 쓰지?"

체크 순서:

1. **글로벌 차단**인가? → `ALL_AGENT_DISALLOWED_TOOLS` 확인
2. **커스텀 에이전트 차단**인가? → `CUSTOM_AGENT_DISALLOWED_TOOLS` 확인
3. **비동기 경로로 들어갔는가?** → 그렇다면 `ASYNC_AGENT_ALLOWED_TOOLS`에 들어 있어야 함
4. **에이전트 정의의 `tools` 필드**가 해당 도구를 빠뜨리지 않았는가?
5. **`disallowedTools`**가 명시적으로 차단하고 있지 않은가?
6. **`permissionMode: 'plan'`** 상태에서 수정 도구를 호출하고 있지 않은가?
7. **deny 규칙**이 `ToolPermissionContext`에 등록되어 있는가? (부모 세션 설정)

모든 단계는 `filterToolsForAgent()` → `resolveAgentTools()` 한 체인 안에서 확인할 수 있습니다.
