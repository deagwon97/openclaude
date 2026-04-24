# 전역 상태 (AppState) 시스템

OpenClaude 하네스의 런타임 상태는 하나의 `AppState` 객체에 집중되어 있고, `AppStateStore` 라는 얇은 observable 스토어로 관리된다. React UI·QueryEngine·툴 실행·퍼미션·세션 재개 등 거의 모든 subsystem이 이 스토어를 통해 데이터를 공유한다.

## 문서 목록

| 파일 | 주제 |
|------|------|
| [01_store.md](01_store.md) | `createStore` — 20줄짜리 Observable 구현 |
| [02_appstate_shape.md](02_appstate_shape.md) | `AppState` 타입 — 필드별 용도 분류 |
| [03_provider_and_subscribe.md](03_provider_and_subscribe.md) | `AppStateProvider` React 통합, `useSyncExternalStore` |
| [04_on_change_side_effects.md](04_on_change_side_effects.md) | `onChangeAppState` — 상태 변경을 외부 시스템에 전파 |
| [05_selectors.md](05_selectors.md) | 순수 셀렉터 — 파생 상태 추출 |

---

## 핵심 파일

| 역할 | 파일 |
|------|------|
| Store primitive | [src/state/store.ts](../../src/state/store.ts) |
| AppState 타입 & 기본값 | [src/state/AppStateStore.ts](../../src/state/AppStateStore.ts) |
| React Provider | [src/state/AppState.tsx](../../src/state/AppState.tsx) |
| 변경 사이드이펙트 | [src/state/onChangeAppState.ts](../../src/state/onChangeAppState.ts) |
| 셀렉터 | [src/state/selectors.ts](../../src/state/selectors.ts) |
| Teammate 뷰 헬퍼 | [src/state/teammateViewHelpers.ts](../../src/state/teammateViewHelpers.ts) |
| 플러그인 커맨드 스토어 | [src/state/pluginCommandsStore.ts](../../src/state/pluginCommandsStore.ts) |

---

## 전체 구조 한눈에 보기

```
                        ┌───────────────────────┐
   setAppState(fn)  →   │  AppStateStore        │  ←  subscribe(listener)
                        │  (Store<AppState>)    │
                        └───────────┬───────────┘
                                    │
                    변경 시 onChange 호출
                                    │
                     ┌──────────────┴──────────────┐
                     ▼                             ▼
          onChangeAppState()              notify listeners
          - CCR/SDK 동기화                 (useSyncExternalStore)
          - settings 저장                      │
          - auth 캐시 무효화                    ▼
          - 환경변수 재적용                 React 리렌더
          - plan mode 전환 훅
```

- **단일 스토어**: 모든 하네스 상태가 한 객체. 개별 zustand slice 같은 게 아님.
- **불변 업데이트**: `setAppState(prev => ({...prev, x: y}))` — `Object.is` 동일성 검사로 no-op 생략.
- **외부 방출**: 상태 변경이 설정 파일·원격 세션·분석 이벤트·환경 변수로 **자동 전파**됨 (`onChangeAppState`).

---

## 읽는 쪽 (3가지 방식)

1. **React 컴포넌트**: `useAppState()` 훅 → `useSyncExternalStore` 로 구독
2. **비-React 코드** (툴, QueryEngine): `context.getAppState()` / `context.setAppState()` 로 직접 접근
3. **셀렉터**: `getViewedTeammateTask(appState)`, `getActiveAgentForInput(appState)` — 순수 함수

---

## 관련 문서

- 퍼미션 모드: [../plan_mode/03_permissions.md](../plan_mode/03_permissions.md)
- 툴 컨텍스트: [../context/03_tool_context.md](../context/03_tool_context.md)
- 세션 상태: [../context/04_session_state.md](../context/04_session_state.md)
