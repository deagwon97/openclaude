# 03. Plan Mode 의 권한 처리

Plan Mode 는 단순히 "mode = 'plan'" 필드만 바꾸는 게 아니다. 퍼미션 시스템의 여러 플래그와 자동 모드(auto) 갈림길까지 함께 관리하는 **복합 전환** 이다.

## 1. PermissionMode 전체 목록

[src/utils/permissions/PermissionMode.ts](../../src/utils/permissions/PermissionMode.ts)

```typescript
PERMISSION_MODES = [
  'default',
  'plan',
  'acceptEdits',
  'bypassPermissions',
  'dontAsk',
  'auto',        // ant only (TRANSCRIPT_CLASSIFIER)
  'bubble',      // ant only
]
```

| 모드 | 심볼 | 기본 동작 |
|------|------|---------|
| `default` | | 모든 도구 프롬프트 |
| `plan` | ⏸ | 쓰기 차단, 읽기만 허용 |
| `acceptEdits` | ⏵⏵ | 편집 자동 허용 |
| `bypassPermissions` | ⏵⏵ | 모든 검사 우회 (위험) |
| `dontAsk` | ⏵⏵ | 거부/허용 결정만 유지, 신규 요청은 거부 |
| `auto` | ⏵⏵ | 분류기(classifier)가 자동 결정 (ant only) |

**외부 노출 가능 모드** (`EXTERNAL_PERMISSION_MODES`): `default, plan, acceptEdits, bypassPermissions, dontAsk` — `auto` 와 `bubble` 은 내부 전용.

## 2. `prepareContextForPlanMode` — 진입 시

[src/utils/permissions/permissionSetup.ts:1460-1491](../../src/utils/permissions/permissionSetup.ts#L1460-L1491)

```typescript
export function prepareContextForPlanMode(
  context: ToolPermissionContext,
): ToolPermissionContext {
  const currentMode = context.mode
  if (currentMode === 'plan') return context

  // 1. Auto 모드 + TRANSCRIPT_CLASSIFIER 경로
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const planAutoMode = shouldPlanUseAutoMode()

    if (currentMode === 'auto') {
      if (planAutoMode) {
        // auto → plan: auto 를 유지 (opt-in)
        return { ...context, prePlanMode: 'auto' }
      }
      // auto → plan: auto 를 내림
      autoModeStateModule?.setAutoModeActive(false)
      setNeedsAutoModeExitAttachment(true)
      return { ...restoreDangerousPermissions(context), prePlanMode: 'auto' }
    }

    if (planAutoMode && currentMode !== 'bypassPermissions') {
      // opt-in: 플랜 동안 auto 를 활성화
      autoModeStateModule?.setAutoModeActive(true)
      return {
        ...stripDangerousPermissionsForAutoMode(context),
        prePlanMode: currentMode,
      }
    }
  }

  // 2. 일반 경로: 현재 모드를 prePlanMode 로 스태시
  return { ...context, prePlanMode: currentMode }
}
```

### 2.1 핵심 필드 변화

| 필드 | before | during plan | after exit |
|------|--------|-------------|------------|
| `mode` | e.g. `default` | `'plan'` | `prePlanMode` |
| `prePlanMode` | undefined | `'default'` (이전값) | `undefined` |
| `strippedDangerousRules` | undefined | (auto opt-in 시) {...} | undefined (restore) |

### 2.2 `shouldPlanUseAutoMode`
ant 전용 설정. 플랜 모드 동안 auto 분류기 활성화 여부. 기본 off; `useAutoModeDuringPlan` 설정이 true 면 on.

**시나리오 3가지** (TRANSCRIPT_CLASSIFIER):
- `auto → plan` + opt-in ✅  →  `auto` 유지
- `auto → plan` + opt-out ❌  →  `auto` 끄고 rules 복원
- `default → plan` + opt-in ✅  →  `auto` 켜고 dangerous rules strip

## 3. `strippedDangerousRules` — 자동 모드 안전 장치

[src/utils/permissions/permissionSetup.ts:510-578](../../src/utils/permissions/permissionSetup.ts#L510-L578)

```typescript
export function stripDangerousPermissionsForAutoMode(
  context: ToolPermissionContext,
): ToolPermissionContext
```

Auto 모드(분류기 기반) 에서는 사용자가 설정 파일에 허용한 "위험한" 규칙(`Bash(rm:*)` 등) 이 분류기를 우회하게 해선 안 된다. 이 함수가:

1. dangerous rules 를 stash 에 빼내어 `strippedDangerousRules` 필드에 보관
2. 현재 context 에서 dangerous rules 제거

`restoreDangerousPermissions` 가 역변환. Exit 시 stash 를 원복한다.

### 3.1 왜 "strip" 이 필요한가
- 사용자 설정: `allow: ['Bash(rm *)']`
- Auto 모드: 분류기가 문장 단위로 위험도 판단
- 만약 strip 하지 않으면: 분류기가 "위험" 이라고 판단해도, prefix rule 매칭이 먼저 allow 를 반환 → 분류기 bypass

### 3.2 `transitionPlanAutoMode` — 플랜 중 설정 변경
```typescript
export function transitionPlanAutoMode(
  context: ToolPermissionContext,
): ToolPermissionContext
```

플랜 중에 `useAutoModeDuringPlan` 설정이 바뀌면 (다른 터미널에서 편집) 동적으로 auto 를 on/off 한다. `applySettingsChange` 에서 호출.

## 4. Circuit Breaker — Exit 시 자동 모드 게이트 체크

[ExitPlanModeV2Tool.ts:328-346](../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts#L328-L346)

```typescript
if (feature('TRANSCRIPT_CLASSIFIER')) {
  const prePlanRaw = appState.toolPermissionContext.prePlanMode ?? 'default'
  if (
    prePlanRaw === 'auto' &&
    !(permissionSetupModule?.isAutoModeGateEnabled() ?? false)
  ) {
    const reason = permissionSetupModule?.getAutoModeUnavailableReason()
    gateFallbackNotification = permissionSetupModule?.getAutoModeUnavailableNotification(reason)
    // → auto 대신 default 로 복원
  }
}
```

**시나리오**: 사용자가 auto 모드에서 플랜에 진입. 플랜 중 circuit breaker 가 발동 (분류기 에러율 초과, 설정으로 auto 비활성 등) → Exit 시 `prePlanMode='auto'` 이지만 gate 가 off → 안전하게 `default` 로 복원하고 사용자에게 알림.

## 5. `needsPlanModeExitAttachment` — 메타 메시지 주입

[src/bootstrap/state.ts:1349-1363](../../src/bootstrap/state.ts#L1349-L1363)

```typescript
export function handlePlanModeTransition(fromMode, toMode): void {
  if (toMode === 'plan' && fromMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = false   // 진입 시 초기화
  }
  if (fromMode === 'plan' && toMode !== 'plan') {
    STATE.needsPlanModeExitAttachment = true    // 퇴장 시 세트
  }
}
```

`toMode === 'plan' && fromMode !== 'plan'` 시 flag 를 **false 로** 초기화하는 이유: 사용자가 모드를 빠르게 토글(`default → plan → default → plan`) 했을 때 `plan_mode_exit` attachment 가 잘못 주입되는 것을 방지.

### 5.1 Attachment 실제 주입
다음 쿼리의 context 빌드 시:
- `needsPlanModeExitAttachment === true` → `plan_mode_exit` attachment 추가
- 이 attachment 는 "플랜 모드가 끝났고 지금부터 구현" 이라는 system reminder 로 변환됨

[src/utils/attachments.ts](../../src/utils/attachments.ts) 의 `plan_mode_exit` case 가 처리.

## 6. Permission 모드 사이드이펙트 (onChangeAppState)

[src/state/onChangeAppState.ts:67-94](../../src/state/onChangeAppState.ts#L67-L94)

Plan ↔ Default 전환이 감지되면:
- `notifySessionMetadataChanged({ permission_mode: 'plan' | 'default' })` — CCR 원격 메타데이터 동기화
- `notifyPermissionModeChanged(newMode)` — SDK status stream

**Ultraplan 특례**: `plan` 으로 진입하면서 `isUltraplanMode` 도 함께 true 로 켜진 경우에만 `is_ultraplan_mode: true` 를 송신 (초기 1회).

## 7. 요약 표

| 동작 | 파일 |
|------|------|
| prePlanMode 스태시 | `permissionSetup.ts: prepareContextForPlanMode` |
| dangerous rules strip | `permissionSetup.ts: stripDangerousPermissionsForAutoMode` |
| 플랜 중 설정 변경 대응 | `permissionSetup.ts: transitionPlanAutoMode` |
| mode 복원 | `ExitPlanModeV2Tool.ts: call()` |
| exit attachment flag | `bootstrap/state.ts: handlePlanModeTransition` |
| CCR 동기화 | `state/onChangeAppState.ts` |
