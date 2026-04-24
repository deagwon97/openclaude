# 04. onChangeAppState — 상태 변경의 외부 전파

[src/state/onChangeAppState.ts](../../src/state/onChangeAppState.ts) 는 `createStore` 의 `onChange` 콜백에 연결되는 **단일 파일**이다. 이전 상태와 새 상태를 비교해 외부 시스템(CCR, settings.json, auth 캐시, 환경 변수)에 변경을 전파한다.

## 왜 한 곳에 모았나

이 파일의 주석 요지:

> 이전에는 mode 변경이 8+ 경로에서 개별적으로 CCR에 알려야 했고 (Shift+Tab, /plan 커맨드, rewind, REPL bridge, ExitPlanMode 등), 대부분 빼먹었다. 결과: `external_metadata.permission_mode` 가 stale. 여기로 모으면 모든 setAppState 경로가 **자동**으로 동기화된다.

**원칙**: 상태 mutation 은 `setAppState` 만 쓰고, 그 결과로 일어나야 할 일은 여기서 한 번만 선언.

## 1. 퍼미션 모드 변경 → CCR / SDK 알림

[src/state/onChangeAppState.ts:67-94](../../src/state/onChangeAppState.ts#L67-L94)

```
prevMode !== newMode
  ↓
toExternalPermissionMode(prevMode) vs (newMode)
  ↓ 달라졌으면
notifySessionMetadataChanged({ permission_mode, is_ultraplan_mode })
  (→ ccrClient.reportMetadata)
  ↓ 항상
notifyPermissionModeChanged(newMode)
  (→ print.ts SDK status stream)
```

주목:
- **External 모드로 먼저 변환**: `bubble`, `auto` 같은 내부 전용 모드 이름을 CCR 에 노출하지 않음
- **External 이 동일하면 CCR 알림 생략**: `default → bubble → default` 같은 내부 토글 노이즈 무시
- **Ultraplan 라이프사이클**: 첫 플랜 사이클의 시작에서만 `is_ultraplan_mode: true` 송신. RFC 7396 에 따라 `null` 은 키 삭제.

## 2. `mainLoopModel` → settings.json 저장

[src/state/onChangeAppState.ts:97-120](../../src/state/onChangeAppState.ts#L97-L120)

```
mainLoopModel: null    → settings.model = undefined (삭제)
mainLoopModel: 'opus'  → settings.model = 'opus' (저장)
                       → setMainLoopModelOverride(...)
                       → (provider profile 환경이면) profile 에도 저장
```

`/model` 커맨드로 모델을 바꾸면 **다음 세션 재개시에도 그 모델이 유지**되는 이유가 여기에 있다.

## 3. `expandedView` → globalConfig

[src/state/onChangeAppState.ts:122-136](../../src/state/onChangeAppState.ts#L122-L136)

`expandedView` 는 3-state (`none|tasks|teammates`) 인데 globalConfig 에는 `showExpandedTodos`, `showSpinnerTree` 두 개의 boolean 으로 저장된다 — **레거시 호환**. 여기서 변환한다.

## 4. `verbose` → globalConfig

`--verbose` 토글이 전역 설정으로 영속화.

## 5. `tungstenPanelVisible` → globalConfig (ant only)

ant 직원용 tmux 패널 sticky 토글. `isAntEmployee()` 가드.

## 6. `settings` 변경 → 캐시 무효화 + 환경 변수 재적용

[src/state/onChangeAppState.ts:162-178](../../src/state/onChangeAppState.ts#L162-L178)

```
settings 변화 감지
  ↓
clearApiKeyHelperCache()
clearAwsCredentialsCache()
clearGcpCredentialsCache()
  ↓
settings.env 변화 시 → applyConfigEnvironmentVariables()
```

**이유**: `apiKeyHelper` 커맨드를 바꾸거나 AWS profile 을 스왑했을 때 **즉시 반영**되어야 함. 캐시가 stale 하면 다음 API 호출이 옛 자격으로 나감.

`settings.env` 변경은 **additive-only** — 새 변수 추가/기존 덮어쓰기만 하고 삭제는 안 함. 이미 로드된 라이브러리의 환경 변수 의존성 손상 방지.

## 7. `externalMetadataToAppState` — 역방향 복원

[src/state/onChangeAppState.ts:26-43](../../src/state/onChangeAppState.ts#L26-L43)

외부 시스템(CCR) 이 원격 세션의 mode 나 ultraplan 플래그를 바꾸면, 이 함수가 역방향으로 `AppState` 를 업데이트한다. 워커 재시작 시 CCR 의 `external_metadata` 에서 모드를 복원할 때 사용.

## 호출 지점

```typescript
// main.tsx 혹은 비슷한 부트스트랩 지점
<AppStateProvider
  initialState={...}
  onChangeAppState={onChangeAppState}  // ← 여기에 연결
>
```

`createStore` 가 이 콜백을 내부에 저장하고, 매 `setState` 마다 `newState !== prevState` 인 경우에 호출.

## 사이드이펙트 맵 요약

| AppState 필드 변경 | 파급 |
|------------------|------|
| `toolPermissionContext.mode` | CCR 메타데이터, SDK status stream |
| `mainLoopModel` | settings.json, 오버라이드, provider profile |
| `expandedView` | globalConfig (2개 boolean 으로 변환) |
| `verbose` | globalConfig |
| `tungstenPanelVisible` (ant) | globalConfig |
| `settings` | apiKey/AWS/GCP 캐시 무효화, env 재적용 |

## 주의

`onChangeAppState` 는 **동기 실행**이다. 무거운 I/O (분석 이벤트 발송 등) 는 내부 함수에서 async fire-and-forget 으로 처리된다. 여기가 느려지면 UI 응답성이 떨어짐 — 주의해서 새 사이드이펙트를 추가해야 한다.
