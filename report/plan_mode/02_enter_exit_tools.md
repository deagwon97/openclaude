# 02. EnterPlanMode / ExitPlanMode 도구

## 1. `EnterPlanMode`

### 1.1 정의
[src/tools/EnterPlanModeTool/EnterPlanModeTool.ts](../../src/tools/EnterPlanModeTool/EnterPlanModeTool.ts)

- **파라미터 없음**: `z.strictObject({})`
- **shouldDefer: true** — 지연 로드되는 도구 (`ToolSearch` 로만 전체 스키마 획득 가능)
- **isReadOnly: true** — 모드 전환만 하므로 읽기 전용
- **isEnabled**: `--channels` 활성 시 false

### 1.2 Agent 컨텍스트 차단
```typescript
async call(_input, context) {
  if (context.agentId) {
    throw new Error('EnterPlanMode tool cannot be used in agent contexts')
  }
  ...
}
```

### 1.3 모드 전환

```typescript
handlePlanModeTransition(appState.toolPermissionContext.mode, 'plan')

context.setAppState(prev => ({
  ...prev,
  toolPermissionContext: applyPermissionUpdate(
    prepareContextForPlanMode(prev.toolPermissionContext),
    { type: 'setMode', mode: 'plan', destination: 'session' },
  ),
}))
```

2단 처리:
1. `prepareContextForPlanMode` — `prePlanMode` 스태시 + 자동 모드 갈림길 처리
2. `applyPermissionUpdate` with `destination: 'session'` — 세션 단위 모드 변경 (재시작 시 default 로 복귀)

### 1.4 도구 결과 메시지

[EnterPlanModeTool.ts:103-125](../../src/tools/EnterPlanModeTool/EnterPlanModeTool.ts#L103-L125)

두 버전 분기:

**인터뷰 페이즈 활성** (`isPlanModeInterviewPhaseEnabled() === true`):
```
Entered plan mode. ...
DO NOT write or edit any files except the plan file. Detailed workflow instructions will follow.
```
뒤이어 별도의 attachment 로 자세한 workflow 가 주입된다.

**일반 모드**:
```
In plan mode, you should:
1. Thoroughly explore the codebase to understand existing patterns
2. Identify similar features and architectural approaches
3. Consider multiple approaches and their trade-offs
4. Use AskUserQuestion if you need to clarify the approach
5. Design a concrete implementation strategy
6. When ready, use ExitPlanMode to present your plan for approval

Remember: DO NOT write or edit any files yet. This is a read-only exploration and planning phase.
```

### 1.5 도구 프롬프트 (LLM 에게 언제 쓸지 알려줌)

[src/tools/EnterPlanModeTool/prompt.ts](../../src/tools/EnterPlanModeTool/prompt.ts)

7가지 상황에서 **proactive 하게** 사용하도록 유도:
1. 새 기능 구현
2. 여러 접근법 가능
3. 기존 동작/구조 변경
4. 아키텍처 결정
5. 다중 파일 변경 (>2-3 파일)
6. 요구사항 불명확
7. 사용자 선호가 중요한 경우 — `AskUserQuestion` 을 쓸 것 같으면 **대신** EnterPlanMode

## 2. `ExitPlanMode` (V2)

### 2.1 정의
[src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts](../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts)

- **파라미터**: `allowedPrompts` 배열 (선택) — 세션 단위 의미 권한 요청
- **내부 입력**: `plan`, `planFilePath` 은 `normalizeToolInput` 에서 디스크로부터 주입
- **shouldDefer: true**, **isReadOnly: false** (디스크 write)

### 2.2 입력 스키마의 이중 버전

```typescript
// 내부 스키마 — call() 가 보는 것
const inputSchema = z.strictObject({
  allowedPrompts: z.array(allowedPromptSchema()).optional(),
}).passthrough()

// SDK/hook 스키마 — normalizeToolInput 이 주입
export const _sdkInputSchema = inputSchema().extend({
  plan: z.string().optional(),
  planFilePath: z.string().optional(),
})
```

이유: LLM 이 플랜 내용을 도구 호출 인자로 직접 넘기는 게 아니라 **디스크에 작성한 후 호출** 하는 패턴. SDK/hook 은 normalized 된 버전을 봐야 해서 `_sdkInputSchema` 가 따로 있음.

### 2.3 `validateInput` — 모드 체크

```typescript
const mode = getAppState().toolPermissionContext.mode
if (mode !== 'plan') {
  logEvent('tengu_exit_plan_mode_called_outside_plan', {...})
  return {
    result: false,
    message: 'You are not in plan mode. This tool is only for exiting plan mode after writing a plan. If your plan was already approved, continue with implementation.',
    errorCode: 1,
  }
}
```

Deferred 도구라 LLM 이 모드 관계없이 호출 시도 가능 → validateInput 에서 차단.

### 2.4 `checkPermissions` — 사용자 승인

- **Teammate**: `behavior: 'allow'` (UI 거치지 않음; call 이 mailbox 경로 처리)
- **비-teammate**: `behavior: 'ask'`, `message: 'Exit plan mode?'`

### 2.5 CCR 웹 UI 에서의 플랜 편집

```typescript
const inputPlan = 'plan' in input && typeof input.plan === 'string'
  ? input.plan
  : undefined
const plan = inputPlan ?? getPlan(context.agentId)

if (inputPlan !== undefined && filePath) {
  await writeFile(filePath, inputPlan, 'utf-8').catch(e => logError(e))
  void persistFileSnapshotIfRemote()
}
```

CCR 웹 UI 가 사용자에게 플랜을 보여주고 편집을 허용한다. 편집된 플랜은 `permissionResult.updatedInput` 로 들어와 `input.plan` 에 있고, 이것을 디스크에 다시 쓴다.

### 2.6 Exit 처리 — mode 복원

[ExitPlanModeV2Tool.ts:357-420](../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts#L357-L420)

```typescript
context.setAppState(prev => {
  if (prev.toolPermissionContext.mode !== 'plan') return prev  // no-op 가드
  setHasExitedPlanMode(true)
  setNeedsPlanModeExitAttachment(true)
  let restoreMode = prev.toolPermissionContext.prePlanMode ?? 'default'

  // Circuit breaker: prePlanMode='auto' 인데 게이트가 off 면 → default 로
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    if (restoreMode === 'auto' && !isAutoModeGateEnabled()) {
      restoreMode = 'default'
    }
    autoModeStateModule?.setAutoModeActive(restoreMode === 'auto')
    if (autoWasUsedDuringPlan && !finalRestoringAuto) {
      setNeedsAutoModeExitAttachment(true)
    }
  }

  // stripped rules 복원
  let baseContext = prev.toolPermissionContext
  if (restoringToAuto) {
    baseContext = stripDangerousPermissionsForAutoMode(baseContext)
  } else if (prev.toolPermissionContext.strippedDangerousRules) {
    baseContext = restoreDangerousPermissions(baseContext)
  }

  return {
    ...prev,
    toolPermissionContext: { ...baseContext, mode: restoreMode },
  }
})
```

3가지 정리 동시 수행:
1. `mode ← prePlanMode` 복원
2. auto 모드 상태 동기화 (game-on during plan → restore)
3. stripped dangerous rules 복원

### 2.7 `awaitingLeaderApproval` 분기

Teammate + `plan_mode_required` 인 경우:
```typescript
if (isTeammate() && isPlanModeRequired()) {
  // mode 를 바꾸지 않고
  // team-lead mailbox 로 plan_approval_request 전송
  // task state: awaitingPlanApproval = true
  return { data: { plan, isAgent: true, filePath, awaitingLeaderApproval: true, requestId } }
}
```
모드 전환은 **리더가 승인한 후** 일어난다. 자세한 내용: [05_teammate_approval.md](05_teammate_approval.md).

## 3. 도구 등록

두 도구 모두:
- **shouldDefer: true** — `getAllBaseTools()` 에서 제외되고, `ToolSearch` 로만 전체 schema 가져옴
- Deferred 도구 리스트에는 항상 이름만 노출 (post-compact, post-clear 에도 LLM 이 기억할 수 있게)

## 4. 주의: `hasExitedPlanModeInSession`

`bootstrap/state.ts` 의 STATE 플래그. 세션 내에서 한 번이라도 ExitPlanMode 를 성공했는지 기록. 분석 이벤트 `tengu_exit_plan_mode_called_outside_plan` 가 이 값을 함께 로깅하여 "플랜 모드가 아닌데 호출된 이유" (승인 후 재호출인지, 완전 오용인지) 를 구분할 수 있게 한다.
