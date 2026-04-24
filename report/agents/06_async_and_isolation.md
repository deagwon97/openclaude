# 비동기 실행 & 격리 (Async & Isolation)

서브에이전트를 백그라운드로 돌리거나, 파일시스템을 격리된 worktree에서 실행하는 경로를 다룹니다.

핵심 파일:
- [src/tools/AgentTool/AgentTool.tsx](../../src/tools/AgentTool/AgentTool.tsx)
- [src/tools/AgentTool/runAgent.ts](../../src/tools/AgentTool/runAgent.ts)

---

## 1. 동기 vs 비동기 판정

`AgentTool.call()`은 다음 조건 중 하나라도 참이면 서브에이전트를 비동기(백그라운드)로 실행합니다.

| 트리거 | 설명 |
|--------|------|
| `run_in_background: true` | LLM이 직접 지정한 입력 파라미터 |
| 에이전트 정의의 `background: true` | 항상 백그라운드 |
| Coordinator 모드 | 코디네이터가 자식을 띄울 때 기본 비동기 |
| Fork 서브에이전트 기능 플래그 | 활성화 시 fork 경로는 비동기로 |
| Proactive / Kairos 모드 | 주기적/프로액티브 에이전트 |

동기 경로에서도 **120초 이상 실행되면 자동으로 백그라운드로 전환**됩니다 (상위 UI가 블로킹되지 않도록 하는 안전장치).

---

## 2. 비동기 실행 흐름

```
AgentTool.call()
      │
      ├─ shouldRunAsync 판정
      │
      ├─ registerAsyncAgent({ agentId, description, prompt, ... })
      │       │
      │       └─ 백그라운드 태스크 등록
      │          (부모 abort controller와 분리된 컨트롤러 사용)
      │
      ├─ 에이전트 이름 레지스트리 갱신 (agentNameRegistry)
      │   ─ name 파라미터가 주어지면 SendMessage 라우팅 키로 사용
      │
      └─ 즉시 task_id 반환
              │
              ▼
LLM은 이후 TaskOutput(task_id)로 폴링하거나
SendMessage로 메시지를 보냄
```

### 2.1 결과 저장

비동기 에이전트의 최종 결과는 메모리에 유지되지 않고, JSON 파일로 저장됩니다. 부모는 `TaskOutput` 도구로 이 파일을 읽어 결과를 확인합니다. 자세한 도구 스펙은 [../tools/03_agent_task.md](../tools/03_agent_task.md)의 `TaskOutput` 섹션 참고.

### 2.2 에이전트 이름 레지스트리

`name` 파라미터가 주어지면 `agentNameRegistry`(Map)에 `name → agentId` 매핑이 등록됩니다. 이후 부모 또는 다른 에이전트가 `SendMessage` 도구로 `to: "researcher"` 같은 사람 친화적 이름을 써서 메시지를 보낼 수 있습니다. Agent Swarms 기능의 핵심 축입니다.

### 2.3 자동 거부 모드

비동기 에이전트는 권한 프롬프트를 띄울 UI가 없으므로, [05_permissions.md](05_permissions.md)에서 설명한 대로:

- `ASYNC_AGENT_ALLOWED_TOOLS` 화이트리스트로 도구 좁힘
- `canShowPermissionPrompts = false`로 권한 프롬프트 자동 거부
- `permissionMode`의 bubble 전환도 비활성

---

## 3. Worktree 격리

`isolation: 'worktree'`로 실행하면 서브에이전트가 별도의 git worktree에서 작업합니다.

```typescript
// AgentTool.tsx
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`
  worktreeInfo = await createAgentWorktree(slug)
}
```

### 3.1 장점

- 병렬 에이전트가 서로의 파일 변경을 간섭하지 않음
- 실패 시 worktree만 버리면 부모 작업 디렉토리는 무결
- 커밋을 생성해도 별도 브랜치에 격리됨

### 3.2 자동 정리

```typescript
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {}
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit)
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot)
    }
  }
}
```

**변경 사항이 없으면 worktree가 자동 삭제되고, 변경이 있으면 보존됩니다.** 보존된 경우 경로와 브랜치 정보가 결과에 포함되어 부모가 이어받을 수 있습니다.

### 3.3 Remote 격리

`isolation: 'remote'`는 원격 호스트에서 에이전트를 실행합니다. 로컬 리소스를 쓰지 않아 대규모 병렬 실행에 적합하며, `checkRemoteAgentEligibility()`로 가능 여부를 먼저 판단합니다.

---

## 4. 부모 ↔ 자식 메시지 전달 (비동기 경로)

동기 경로와 달리, 비동기 경로에서는 메시지 교환이 명시적인 도구 호출을 통해 이뤄집니다.

| 방향 | 메커니즘 |
|------|---------|
| 부모 → 자식 | `SendMessage` 도구 (Agent Swarms 기능) |
| 자식 → 부모 | `SyntheticOutput` 도구로 결과 파일에 기록, 부모가 `TaskOutput`으로 조회 |
| 자식 종료 | `TaskStop` 도구 또는 자체 완료 |
| 상태 조회 | `TaskList`, `TaskGet` |

이 도구들은 비동기 에이전트 화이트리스트에 포함되거나 `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`에 등록되어 있어야 사용할 수 있습니다.

---

## 5. 정리(Cleanup) 책임 요약

| 대상 | 정리 시점 | 함수 |
|------|----------|------|
| MCP 클라이언트(새로 만든 것만) | `runAgent()` finally | `mcpCleanup()` |
| 세션 훅 | `runAgent()` finally | `clearSessionHooks()` |
| 파일 상태 캐시 | `runAgent()` finally | `agentToolUseContext.readFileState.clear()` |
| Bash 백그라운드 태스크 | `runAgent()` finally | `killShellTasksForAgent(agentId, ...)` |
| Worktree (변경 없을 때) | AgentTool.call() finally | `removeAgentWorktree()` |
| Async agent task | TaskStop 또는 자체 완료 | `stopTask(task_id)` |
| 초기 메시지 배열 | `runAgent()` finally | `initialMessages.length = 0` |

---

## 6. 실행 모드 결정 다이어그램

```
           AgentTool.call() 입력
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
  run_in_background?   definition.background?
         │                   │
         └────────┬──────────┘
                  │
                  ▼
             shouldRunAsync
          ┌────┴────┐
        Yes         No
         │           │
         ▼           ▼
  registerAsyncAgent   runAgent() 직접 호출
         │              (동기 generator)
         ▼                     │
   task_id 반환                 ▼
   (부모는 TaskOutput으로     120초 초과?
    나중에 조회)                 │
                           ┌────┴────┐
                         Yes         No
                           │           │
                           ▼           ▼
                    백그라운드 전환   완료까지 대기
                           │           │
                           ▼           ▼
                        task_id   tool_result
```

---

## 7. 정리

- **격리는 선택**, **비동기는 자동+선택** — 두 축은 독립적으로 조합 가능
- **부모와 자식의 AbortController 분리**가 안정성의 핵심
- **비동기 경로는 권한 프롬프트가 불가능**하므로 도구 화이트리스트와 `permissionMode`가 더 엄격하게 적용
- **Worktree는 변경 유무에 따라 자동 정리/보존** — 부모 작업 디렉토리 오염 방지
