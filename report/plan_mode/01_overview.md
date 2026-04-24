# 01. Plan Mode 개요

## 1. 무엇인가

Plan Mode 는 특수한 **퍼미션 모드** 다 (`PermissionMode = 'plan'`). 이 모드에 있는 동안:

- **쓰기 도구는 차단** — Edit, Write, Bash(쓰기), NotebookEdit, MultiEdit 등이 퍼미션 거부
- **읽기 도구만 허용** — Read, Glob, Grep, Agent (서브에이전트로 더 깊은 탐색)
- **`AskUserQuestion`** — 설계 선택지 질문 가능
- LLM 은 **플랜 파일**(`.claude/plans/{slug}.md`) 에 계획을 작성
- **`ExitPlanMode`** 가 호출되면 사용자에게 승인 대화상자가 뜨고, 승인 시 이전 모드로 복원되어 구현 단계로 전환

## 2. 왜 별도 모드인가

일반 Default 모드에서 "먼저 계획하고 나서 하라" 고 프롬프트로만 지시하면 LLM 이 임의로 무시하거나 중간에 쓰기 도구를 호출할 수 있다. Plan Mode 는 **하네스 수준에서 쓰기 권한을 제거**해서 이를 강제한다.

## 3. 진입 경로 3가지

### 3.1 Shift+Tab (사용자)
Footer 의 모드 pill 을 Shift+Tab 으로 순환. Default → Accept Edits → Plan → Default.

### 3.2 `/plan` 슬래시 커맨드
[src/commands/plan/plan.tsx](../../src/commands/plan/plan.tsx) — REPL 에 설명 문구를 출력하고 모드를 전환.

### 3.3 `EnterPlanMode` 도구 (LLM)
LLM 이 복잡한 구현 작업을 시작하기 전에 proactive 하게 호출. [src/tools/EnterPlanModeTool/prompt.ts](../../src/tools/EnterPlanModeTool/prompt.ts) 에 "이럴 때 사용하라" 가이드가 7가지 상황으로 자세히 기술되어 있다.

## 4. Plan Mode 세션의 특수성

### 4.1 `prePlanMode` 스태시
진입 시 **이전 모드가 `prePlanMode` 에 보관**된다. 이 필드는 `toolPermissionContext.prePlanMode` 에 있으며, Exit 시 이 값으로 모드를 복원한다.

```typescript
// prepareContextForPlanMode (permissionSetup.ts:1460-1491)
if (currentMode !== 'plan') {
  return { ...context, prePlanMode: currentMode }
}
```

### 4.2 플랜 파일
세션마다 고유한 `slug` (예: `curious-sparrow`) 기반 파일:
- **메인 세션**: `.claude/plans/{slug}.md`
- **서브에이전트**: `.claude/plans/{slug}-agent-{agentId}.md`

LLM 은 이 파일을 Edit/Write 으로 직접 수정. (쓰기 도구가 차단되어 있지만 플랜 파일 경로는 예외 허용.)

### 4.3 진입/퇴장 attachment
`handlePlanModeTransition(from, to)` 이 flag 2개를 관리:

| Flag | 동작 |
|------|------|
| `needsPlanModeExitAttachment` | true 시 다음 쿼리에 `plan_mode_exit` attachment 주입 |
| `needsAutoModeExitAttachment` | auto 모드에서 나올 때 동일 |

Plan → Default 전환 시 flag 가 켜지고, 다음 API 호출에서 이 메타 메시지가 함께 송신되어 LLM 이 "플랜 모드가 끝났고 이제 구현 단계" 라는 것을 인지한다.

## 5. Exit 후 구현 흐름

```
ExitPlanMode 도구 호출 (LLM)
  ↓
checkPermissions: 'ask' (사용자 승인 대화상자)
  ↓
승인되면 call() 실행:
  - setHasExitedPlanMode(true)
  - setNeedsPlanModeExitAttachment(true)
  - mode ← prePlanMode (복원)
  - strippedDangerousRules 복원 (자동 모드 갈림길 처리)
  - pendingPlanVerification = { plan, verificationStarted, verificationCompleted }
  ↓
REPL.tsx 가 다음 턴을 트리거 — 복원된 모드로 구현 시작
  ↓
(옵션) VerifyPlanExecution 도구가 배경 검증 수행
```

## 6. `allowedPrompts` — 세션 단위 권한

`ExitPlanMode` 호출 시 LLM 은 구현에 필요한 **의미 기반 권한**을 요청할 수 있다:

```typescript
allowedPrompts: [
  { tool: 'Bash', prompt: 'run tests' },
  { tool: 'Bash', prompt: 'install dependencies' },
]
```

사용자 승인 후 이 prompts 는 `initialMessage.allowedPrompts` 로 AppState 에 저장되고, 구현 단계에서 해당 의미에 부합하는 Bash 호출이 자동 허용된다. 특정 명령 화이트리스트보다 **의미 단위** 로 작동해서 유연함.

## 7. 주의 사항

### 7.1 `--channels` 모드와 배타적
Enter/Exit 도구 모두 `isEnabled()` 에서 `getAllowedChannels().length > 0` 일 때 비활성화. 이유: Telegram/iMessage 채널 경유 사용자는 터미널 대화상자를 볼 수 없어 Plan Mode 가 **트랩**이 됨 (진입은 가능한데 나올 방법 없음). 두 도구 모두 동시 비활성화.

### 7.2 Agent 컨텍스트에서 Enter 불가
```typescript
if (context.agentId) {
  throw new Error('EnterPlanMode tool cannot be used in agent contexts')
}
```
서브에이전트는 이미 실행되고 있으므로 플랜 진입이 의미 없음.

### 7.3 Teammate + `plan_mode_required`
팀 리더가 `plan_mode_required: true` 인 teammate 를 소환하면 해당 teammate 는 **자동으로 plan 모드로 시작**. ExitPlanMode 시 팀 리더에게 mailbox 로 승인 요청 전송. 자세한 내용: [05_teammate_approval.md](05_teammate_approval.md).

## 8. 관련 상태 필드

| AppState 필드 | 용도 |
|--------------|------|
| `toolPermissionContext.mode === 'plan'` | 현재 플랜 모드 여부 |
| `toolPermissionContext.prePlanMode` | 진입 전 모드 (복원용) |
| `toolPermissionContext.strippedDangerousRules` | auto 모드 stripped rules 백업 |
| `initialMessage` | Exit 후 구현 단계의 초기 메시지 + allowedPrompts |
| `pendingPlanVerification` | 배경 검증 트리거 |
| `isUltraplanMode` | Ultraplan (원격 세션에서의 플랜) 플래그 |
