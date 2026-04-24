# 07. Coordinator Mode — 멀티 워커 오케스트레이션

`CLAUDE_CODE_COORDINATOR_MODE=1` 환경 변수로 활성화되는 특수 모드. 메인 에이전트가 실행 주체에서 **워커(worker) 조율자**로 역할이 바뀐다. 모든 실제 작업은 `Agent` tool로 생성되는 서브에이전트에 위임되고, 메인은 합성(synthesis)과 사용자 커뮤니케이션만 담당한다.

## 1. 활성화

[src/coordinator/coordinatorMode.ts:36-41](../../src/coordinator/coordinatorMode.ts#L36-L41)

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

- **빌드 게이트**: `feature('COORDINATOR_MODE')` (bun:bundle 데드코드 제거)
- **런타임 게이트**: `CLAUDE_CODE_COORDINATOR_MODE` 환경 변수
- 환경 변수를 **라이브로 읽음** — 캐시 없음, 세션 중 토글 가능

## 2. 세션 재개 시 모드 정합성 보장

[src/coordinator/coordinatorMode.ts:49-78](../../src/coordinator/coordinatorMode.ts#L49-L78)

`matchSessionMode(sessionMode)` — 재개된 세션과 현재 모드가 일치하지 않을 때 **환경 변수를 뒤집어** 맞춘다.

```
sessionMode='coordinator', 현재=normal  →  env.CLAUDE_CODE_COORDINATOR_MODE=1
sessionMode='normal',      현재=coord   →  env.CLAUDE_CODE_COORDINATOR_MODE 삭제
```

분석 이벤트 `tengu_coordinator_mode_switched` 발송 후 경고 문자열을 리턴한다.

## 3. 컨텍스트 주입

### 3.1 User Context — 워커의 도구 목록 설명

[src/coordinator/coordinatorMode.ts:80-109](../../src/coordinator/coordinatorMode.ts#L80-L109)

`getCoordinatorUserContext(mcpClients, scratchpadDir)` 가 반환하는 `workerToolsContext` 는 시스템 프롬프트에 주입되어 **"워커가 어떤 도구를 쓸 수 있는가"** 를 LLM에 명시한다.

- **Simple 모드** (`CLAUDE_CODE_SIMPLE`): `Bash, Read, Edit` 만
- **일반 모드**: `ASYNC_AGENT_ALLOWED_TOOLS` 에서 내부 워커 전용 툴 제외

내부 전용 (프롬프트에서 숨김):
```typescript
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

MCP 서버가 연결되어 있으면 서버 이름 목록이 함께 주입되고, `scratchpad` 게이트(`tengu_scratch`)가 켜져 있으면 워커들이 공용으로 읽고 쓸 수 있는 스크래치패드 경로도 문자열에 붙는다.

### 3.2 Circular Dependency 회피

`isScratchpadGateEnabled()` 가 `utils/permissions/filesystem.ts` 의 `isScratchpadEnabled()` 와 **중복 정의**되어 있다. 이유: `filesystem.ts → permissions → … → coordinatorMode` 순환 import 방지. 실제 스크래치패드 경로는 `QueryEngine.ts` 에서 의존성 주입으로 전달된다.

## 4. 시스템 프롬프트 (Coordinator 페르소나)

[src/coordinator/coordinatorMode.ts:111-369](../../src/coordinator/coordinatorMode.ts#L111-L369)

`getCoordinatorSystemPrompt()` 는 250줄이 넘는 장문의 프롬프트를 반환한다. 핵심 섹션:

| 섹션 | 내용 |
|------|------|
| 1. Your Role | coordinator 정의 — "매 메시지는 사용자에게 보내는 것, 워커는 conversation partner 가 아니다" |
| 2. Your Tools | `Agent`, `SendMessage`, `TaskStop`, `subscribe_pr_activity` |
| 3. Workers | 워커 능력 (simple vs 일반 모드) |
| 4. Task Workflow | Research → Synthesis → Implementation → Verification 4단계 |
| 5. Writing Worker Prompts | 프롬프트 합성의 중요성, continue vs spawn 결정 테이블 |
| 6. Example Session | 전체 예시 세션 |

### 4.1 Task-notification 프로토콜

워커 결과는 **user-role 메시지**에 `<task-notification>` XML 로 전달된다:

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable}</summary>
<result>{agent final text}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

`<task-id>` 값이 `SendMessage` 의 `to` 필드로 사용 가능한 에이전트 ID다.

### 4.2 Continue vs Spawn 결정 테이블

프롬프트에 내장된 결정 테이블 — coordinator 가 언제 같은 워커를 이어 쓰고 언제 새 워커를 띄울지 판단하도록 가이드한다:

| 상황 | 메커니즘 | 이유 |
|------|---------|------|
| 리서치한 파일을 그대로 편집 | **Continue** (`SendMessage`) | 파일이 이미 컨텍스트에 있음 |
| 리서치는 넓고 구현은 좁음 | **Spawn fresh** (`Agent`) | 탐색 노이즈 제거 |
| 실패 재시도/수정 | **Continue** | 에러 컨텍스트 보존 |
| 다른 워커가 쓴 코드 검증 | **Spawn fresh** | fresh eyes 필요 |
| 잘못된 접근 재시도 | **Spawn fresh** | anchoring 방지 |

## 5. 다른 모드와의 관계

- **Coordinator 모드가 켜져 있을 때**: AgentTool이 spawn 하는 워커는 `subagent_type: "worker"` 를 사용 (일반 모드의 `general-purpose` 와 다름)
- **플랜 모드와의 상호작용**: 워커의 `plan_mode_required` 속성을 통해 워커가 ExitPlanMode 시 리더(coordinator)에게 mailbox 로 승인 요청을 보냄 (see [../plan_mode/04_teammate_approval.md](../plan_mode/04_teammate_approval.md))

## 6. 관련 도구

- `Agent` — 워커 생성 (서브에이전트 문서 [../agents/04_execution.md](04_execution.md) 참조)
- `SendMessage` — 기존 워커에게 follow-up 메시지 전송
- `TaskStop` — 실행 중 워커 중단 (tool_id 로 취소)
- `subscribe_pr_activity` — coordinator 가 직접 호출 (워커에 위임 금지), GitHub 이벤트를 user 메시지로 받음

## 7. 핵심 파일

| 역할 | 파일 |
|------|------|
| 모드 판정 & 시스템 프롬프트 | [src/coordinator/coordinatorMode.ts](../../src/coordinator/coordinatorMode.ts) |
| 워커 허용 도구 상수 | [src/constants/tools.ts](../../src/constants/tools.ts) (`ASYNC_AGENT_ALLOWED_TOOLS`) |
| 스크래치패드 게이트 | [src/utils/permissions/filesystem.ts](../../src/utils/permissions/filesystem.ts) |
| 내부 워커 전용 도구 | `TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool`, `SyntheticOutputTool` |
