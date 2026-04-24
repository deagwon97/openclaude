# 05. Teammate 플랜 승인 경로

일반 플랜 모드는 사용자에게 직접 승인을 요청하지만, **Agent Swarms** 에서 `plan_mode_required: true` 로 설정된 teammate 는 **팀 리더**에게 mailbox 경유로 승인을 요청한다.

## 1. Teammate 의 자동 Plan Mode 진입

[src/state/AppStateStore.ts:456-470](../../src/state/AppStateStore.ts#L456-L470) 의 `getDefaultAppState`:

```typescript
const initialMode: PermissionMode =
  teammateUtils.isTeammate() && teammateUtils.isPlanModeRequired()
    ? 'plan'
    : 'default'
```

`plan_mode_required` teammate 는 **시작하자마자 plan 모드** 로 돈다. 진입 경로 3가지(Shift+Tab, `/plan`, EnterPlanMode) 없이 초기 상태.

## 2. `ExitPlanMode` 의 분기

[ExitPlanModeV2Tool.ts:185-238](../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts#L185-L238)

### 2.1 `requiresUserInteraction()`
```typescript
if (isTeammate()) {
  return false  // 로컬 UI 필요 없음
}
return true  // 비-teammate 는 UI 필요
```

### 2.2 `validateInput`
```typescript
if (isTeammate()) {
  return { result: true }  // 모드 검사 생략
}
// 비-teammate: mode !== 'plan' 이면 거부
```

이유: teammate 의 AppState 는 **리더의 모드를 미러링** 할 수 있어 (runAgent.ts 가 일부 모드에서 오버라이드 생략) 로컬 모드 검사가 불안정.

### 2.3 `checkPermissions`
```typescript
if (isTeammate()) {
  return { behavior: 'allow', updatedInput: input }  // UI 거너뜀
}
return { behavior: 'ask', message: 'Exit plan mode?', updatedInput: input }
```

Teammate 는 **권한 UI 를 건너뛰고** call 에서 직접 mailbox 처리.

## 3. Mailbox 경유 승인 요청

[ExitPlanModeV2Tool.ts:263-313](../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts#L263-L313)

```typescript
if (isTeammate() && isPlanModeRequired()) {
  if (!plan) {
    throw new Error(`No plan file found at ${filePath}. Please write your plan to this file before calling ExitPlanMode.`)
  }

  const requestId = generateRequestId(
    'plan_approval',
    formatAgentId(agentName, teamName || 'default'),
  )

  const approvalRequest = {
    type: 'plan_approval_request',
    from: agentName,
    timestamp: new Date().toISOString(),
    planFilePath: filePath,
    planContent: plan,
    requestId,
  }

  await writeToMailbox('team-lead', {
    from: agentName,
    text: jsonStringify(approvalRequest),
    timestamp: new Date().toISOString(),
  }, teamName)

  // Task state 에 awaitingPlanApproval 플래그 세팅
  if (agentTaskId) {
    setAwaitingPlanApproval(agentTaskId, context.setAppState, true)
  }

  return {
    data: {
      plan, isAgent: true, filePath,
      awaitingLeaderApproval: true, requestId,
    },
  }
}
```

### 3.1 승인 요청 페이로드

```json
{
  "type": "plan_approval_request",
  "from": "research-agent",
  "timestamp": "2026-...",
  "planFilePath": ".claude/plans/curious-sparrow-agent-abc.md",
  "planContent": "## Plan\n\n1. ...",
  "requestId": "plan_approval_research-agent_team1_..."
}
```

**JSON 문자열** 로 mailbox 에 기록. 리더 측에서 파싱해 특수 이벤트로 처리.

### 3.2 `writeToMailbox('team-lead', ...)`
[src/utils/teammateMailbox.ts](../../src/utils/teammateMailbox.ts) — 팀 안의 특정 에이전트에게 메시지를 보내는 공용 메커니즘. `team-lead` 는 예약된 이름.

### 3.3 `awaitingPlanApproval` UI 상태

[src/utils/inProcessTeammateHelpers.ts](../../src/utils/inProcessTeammateHelpers.ts) `setAwaitingPlanApproval`:
- Task state 의 상태 전환
- UI (Spinner, TeammateSpinnerLine) 에서 "waiting for approval" 표시

## 4. 리더 측 처리 흐름

리더가 mailbox 에서 이 메시지를 읽으면:

1. `type === 'plan_approval_request'` 감지
2. 특수 UI (approval dialog 또는 inline 알림) 표시
3. 리더가 승인/거부
4. 응답 mailbox 메시지 (type: `plan_approval_response`) 를 teammate 에게 전송
5. teammate 측에서 `awaitingPlanApproval = false` + mode 복원 수행

> 구체적 리더 측 구현은 `utils/swarm/` 과 `components/tasks/` 에 분산되어 있다. 핵심 포인트는 **ExitPlanMode 자체가 로컬에서 모드를 바꾸지 않고, 리더 응답을 기다린다는 것**.

## 5. 왜 이 구조인가

### 5.1 분산된 신뢰 모델
Agent Swarms 는 여러 tmux pane 에 분리된 프로세스로 동작. teammate 가 독립적으로 플랜을 승인해 코드를 쓰면 팀 전체의 의도와 어긋날 수 있다. 리더가 **최종 게이트**.

### 5.2 비동기성
리더가 승인 기다릴 때 teammate 는 멈춰 있지만, **다른 teammate 는 독립적으로 계속 작동**. 리더가 여러 팀원의 플랜을 동시에 관리 가능.

### 5.3 Voluntary vs Required
- `plan_mode_required: false` teammate 도 자발적으로 `EnterPlanMode` 를 호출할 수 있음
- 이 경우 `ExitPlanMode` 시 `isPlanModeRequired() === false` → mailbox 경로 타지 않고 **로컬 exit**
- 자발적 플랜 모드는 승인 없이 스스로 빠져나올 수 있음

## 6. 결정 테이블

| 상태 | ExitPlanMode 동작 |
|------|-----------------|
| 일반 세션 | 사용자 승인 대화상자 |
| teammate + required + 플랜 있음 | mailbox → 리더 승인 대기 |
| teammate + required + 플랜 없음 | throw (플랜 없이 exit 금지) |
| teammate + voluntary plan | 로컬 exit (승인 없이) |
| `--channels` 활성 | 도구 자체 비활성화 |

## 7. 관련 파일

| 역할 | 파일 |
|------|------|
| Mailbox I/O | [src/utils/teammateMailbox.ts](../../src/utils/teammateMailbox.ts) |
| Teammate 신원 헬퍼 | [src/utils/teammate.ts](../../src/utils/teammate.ts) |
| In-process teammate 헬퍼 | [src/utils/inProcessTeammateHelpers.ts](../../src/utils/inProcessTeammateHelpers.ts) |
| Task state 타입 | [src/tasks/InProcessTeammateTask/types.ts](../../src/tasks/InProcessTeammateTask/types.ts) |
| Swarm in-process spawn | [src/utils/swarm/spawnInProcess.ts](../../src/utils/swarm/spawnInProcess.ts) |
| UI 상태 표시 | [src/components/Spinner/TeammateSpinnerLine.tsx](../../src/components/Spinner/TeammateSpinnerLine.tsx) |
