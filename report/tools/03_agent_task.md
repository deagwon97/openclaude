# 에이전트 & 태스크 도구 (Agent & Task Tools)

독립적인 에이전트를 실행하고, 백그라운드 태스크를 관리하는 도구들입니다.

---

## Agent (AgentTool)

**소스**: [src/tools/AgentTool/AgentTool.tsx](../../src/tools/AgentTool/AgentTool.tsx)

### 설명
독립적인 컨텍스트에서 서브에이전트를 실행하는 도구. 복잡한 다단계 작업을 병렬 처리하거나, 특정 역할(탐색, 코드 리뷰 등)을 가진 에이전트에게 위임할 수 있다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `prompt` | string | 필수 | 에이전트에게 줄 지시사항 |
| `description` | string | 선택 | 에이전트 태스크 설명 (3-5 단어) |
| `subagent_type` | string | 선택 | 사용할 에이전트 타입 (내장 또는 커스텀) |
| `isolation` | enum | 선택 | `"worktree"` 지정 시 격리된 git worktree에서 실행 |
| `run_in_background` | boolean | 선택 | 백그라운드 실행 여부 |
| `model` | string | 선택 | 사용할 Claude 모델 ID |

### 동작 흐름
1. 에이전트 타입 결정 (내장 에이전트 vs 커스텀 에이전트)
2. 권한 컨텍스트 상속 (부모 세션 권한 기반)
3. 격리 모드 시 git worktree 생성
4. 독립적인 대화 루프 실행 (`runAgent()`)
5. 결과 반환 (포그라운드) 또는 태스크 등록 (백그라운드)

### 내장 에이전트 타입
- `general-purpose`: 범용 에이전트
- `Explore`: 코드베이스 탐색 전문
- `Plan`: 아키텍처 설계 전문
- `claude-code-guide`: Claude Code 관련 질의 전문

### 특징
- **자동 백그라운드**: 120초 이상 실행 시 자동으로 백그라운드로 전환
- **메모리 스냅샷**: `agentMemory.ts`를 통해 에이전트 간 메모리 공유
- **Worktree 격리**: `isolation: "worktree"` 지정 시 독립적인 git 브랜치에서 작업
- **원격 실행**: `checkRemoteAgentEligibility()`로 원격 실행 가능 여부 판단

### 에이전트 tool 허용/차단 목록
```typescript
// 에이전트가 사용할 수 없는 tool들
ALL_AGENT_DISALLOWED_TOOLS    // 모든 에이전트 차단 목록
CUSTOM_AGENT_DISALLOWED_TOOLS // 커스텀 에이전트 차단 목록
ASYNC_AGENT_ALLOWED_TOOLS     // 비동기 에이전트 허용 목록
COORDINATOR_MODE_ALLOWED_TOOLS // coordinator 모드 허용 목록
```

---

## TaskOutput (TaskOutputTool)

**소스**: [src/tools/TaskOutputTool/TaskOutputTool.tsx](../../src/tools/TaskOutputTool/TaskOutputTool.tsx)

### 설명
백그라운드에서 실행 중인 태스크(에이전트 또는 쉘 명령)의 출력을 조회하거나 완료될 때까지 대기하는 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `task_id` | string | 필수 | 조회할 태스크 ID |
| `block` | boolean | 선택 | 태스크 완료까지 대기 여부 (기본값: true) |
| `timeout` | number | 선택 | 대기 최대 시간 ms (0~600000, 기본값: 30000) |

### 반환값
- `retrieval_status`: `"success"` / `"timeout"` / `"not_ready"`
- `task`: 태스크 상태, 설명, 출력 결과
  - Shell 태스크: stdout/stderr, 종료 코드
  - Agent 태스크: 프롬프트, 최종 결과

---

## TaskStop (TaskStopTool)

**소스**: [src/tools/TaskStopTool/TaskStopTool.ts](../../src/tools/TaskStopTool/TaskStopTool.ts)

### 설명
실행 중인 백그라운드 태스크를 강제 종료하는 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `task_id` | string | 선택 | 종료할 태스크 ID |
| `shell_id` | string | 선택 | 구버전 호환 (deprecated, `task_id` 사용 권장) |

### 동작
- 태스크 존재 여부 및 실행 중 여부 확인
- `stopTask()`를 통해 태스크 중지 (에이전트: AbortController, 쉘: 프로세스 종료)

### 별칭 (aliases)
- `KillShell` (이전 버전 호환용)

---

## TaskCreate (TaskCreateTool)

**소스**: [src/tools/TaskCreateTool/TaskCreateTool.ts](../../src/tools/TaskCreateTool/TaskCreateTool.ts)

### 설명
새로운 태스크를 생성하는 도구. `isTodoV2Enabled()` 조건이 true일 때만 활성화된다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `subject` | string | 필수 | 태스크 제목 |
| `description` | string | 필수 | 수행할 작업 내용 |
| `activeForm` | string | 선택 | 진행 중 표시 텍스트 (예: "Running tests") |
| `metadata` | object | 선택 | 추가 메타데이터 |

### 반환값
- `task.id`: 생성된 태스크 ID
- `task.subject`: 태스크 제목

---

## TaskGet (TaskGetTool)

**소스**: [src/tools/TaskGetTool/TaskGetTool.ts](../../src/tools/TaskGetTool/TaskGetTool.ts)

### 설명
특정 태스크의 상세 정보를 조회하는 도구. `isTodoV2Enabled()` 조건이 true일 때만 활성화된다.

---

## TaskUpdate (TaskUpdateTool)

**소스**: [src/tools/TaskUpdateTool/TaskUpdateTool.ts](../../src/tools/TaskUpdateTool/TaskUpdateTool.ts)

### 설명
태스크의 상태나 내용을 업데이트하는 도구. `isTodoV2Enabled()` 조건이 true일 때만 활성화된다.

---

## TaskList (TaskListTool)

**소스**: [src/tools/TaskListTool/TaskListTool.ts](../../src/tools/TaskListTool/TaskListTool.ts)

### 설명
현재 세션의 태스크 목록을 조회하는 도구. `isTodoV2Enabled()` 조건이 true일 때만 활성화된다.

---

> **참고**: TaskCreate/Get/Update/List 4종은 TodoV2 모드가 활성화된 경우에만 사용됩니다.
> TodoV2가 비활성화된 경우에는 [TodoWrite](05_planning.md#todowrite-todowritetool)를 대신 사용합니다.
