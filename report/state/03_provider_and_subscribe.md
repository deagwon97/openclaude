# 03. React 통합 — AppStateProvider & useAppState

[src/state/AppState.tsx](../../src/state/AppState.tsx) 는 `createStore` 가 만든 `AppStateStore` 를 React 트리에 노출한다. `useSyncExternalStore` 를 기반으로 한 **slice-selector 패턴**을 쓴다.

## 1. `AppStateProvider`

```tsx
<AppStateProvider initialState={...} onChangeAppState={onChangeAppState}>
  {children}
</AppStateProvider>
```

내부 동작:

1. `createStore(initialState, onChangeAppState)` 로 스토어 생성 — 한 번만
2. **중첩 방지**: `HasAppStateContext` 로 이중 Provider 감지 → throw
3. **bypass permission 비활성화 가드**: 마운트 시 `isBypassPermissionsModeDisabled()` 체크
4. **settings 변경 리스너**: `useSettingsChange` 훅으로 외부 settings.json 변화 감지 → `applySettingsChange` 로 스토어 반영
5. `AppStoreContext.Provider` 로 스토어를 React 트리에 공급
6. `MailboxProvider`, `VoiceProvider` 를 중첩해 관련 컨텍스트 동시 제공

> 빌드 변종: `feature('VOICE_MODE')` 가 꺼진 외부 빌드에서는 `VoiceProvider` 가 passthrough 로 DCE 된다.

## 2. `useAppState(selector)` — 슬라이스 구독

[src/state/AppState.tsx:143-156](../../src/state/AppState.tsx#L143-L156)

```tsx
export function useAppState(selector) {
  const store = useAppStore();
  const selectorRef = React.useRef(selector);
  const storeRef = React.useRef(store);
  // render 중 ref 업데이트 — 새 함수 identity를 만들지 않아
  // useSyncExternalStore 재동기화/루프를 방지
  selectorRef.current = selector;
  storeRef.current = store;
  const get = React.useCallback(() => {
    return selectorRef.current(storeRef.current.getState());
  }, []);
  return useSyncExternalStore(store.subscribe, get, get);
}
```

- `useSyncExternalStore` — React 18 공식 외부 스토어 구독 API
- `Object.is` 기반 동일성 비교로 slice 값이 바뀐 경우에만 리렌더
- `selectorRef` 기법은 `useCallback` 의 **stable identity** 를 유지하면서도 최신 셀렉터를 부를 수 있게 한다

### 셀렉터 사용 규칙

**O 기존 서브 객체 참조 반환** — 식별자 동일 유지
```tsx
const { text, promptId } = useAppState(s => s.promptSuggestion)
```

**X 새 객체 반환** — `Object.is` 가 항상 변경으로 판단 → 무한 리렌더
```tsx
const x = useAppState(s => ({ text: s.promptSuggestion.text })) // 안됨
```

**여러 필드는 여러 훅으로**:
```tsx
const verbose = useAppState(s => s.verbose)
const model = useAppState(s => s.mainLoopModel)
```

## 3. `useSetAppState()` — 구독 없이 setter만

```tsx
const setAppState = useSetAppState()
setAppState(prev => ({ ...prev, verbose: true }))
```

- 상태에 구독하지 않으므로 **리렌더 없음**
- setter 참조는 안정적

## 4. `useAppStateStore()` — 비-React 전달용

React 외 코드(툴 콜백, 이벤트 리스너)에 `getState`/`setState`/`subscribe` 를 넘길 때.

## 5. `useAppStateMaybeOutsideOfProvider`

Provider 없는 컨텍스트에서도 안전하게 호출 가능 — 스토어가 없으면 `undefined` 반환. Fallback 컴포넌트·에러 바운더리용.

## 6. 비-React 코드에서의 접근

`ToolUseContext` 에 `getAppState` / `setAppState` 가 직접 내장되어 툴 구현체에서 쓴다:

```typescript
async call(input, context) {
  const appState = context.getAppState()
  context.setAppState(prev => ({
    ...prev,
    toolPermissionContext: applyPermissionUpdate(...)
  }))
}
```

QueryEngine·Agent spawn·hook 실행 등도 동일 패턴. React Provider 계층과 툴 실행 계층이 **같은 스토어** 를 공유한다.

## 요약

| API | 용도 | 리렌더 |
|-----|------|--------|
| `useAppState(selector)` | 특정 필드 구독 | slice 변경시만 |
| `useSetAppState()` | setter 획득 | 없음 |
| `useAppStateStore()` | store 원본 | 없음 |
| `useAppStateMaybeOutsideOfProvider` | Provider 외부에서 안전 구독 | slice 변경시만 |
| `context.getAppState()` (툴) | 동기 접근 | N/A |
| `context.setAppState(fn)` (툴) | 업데이트 | 리스너 경유 |
