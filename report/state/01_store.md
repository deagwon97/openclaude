# 01. Store Primitive — `createStore`

하네스의 상태 저장소는 외부 라이브러리(Redux/Zustand 등)가 아니라 **20여 줄짜리 자체 구현**이다.

## 전체 코드

[src/state/store.ts](../../src/state/store.ts)

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return     // ← no-op 가드
      state = next
      onChange?.({ newState: next, oldState: prev })  // ← 사이드이펙트 훅
      for (const listener of listeners) listener()    // ← 리스너 알림
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

## 설계 포인트

### 1. Updater 기반 `setState`
`setState(prev => next)` 형태만 허용. 직접 값 전달(`setState(newState)`)은 불가 — 이전 상태를 보고 합성하는 패턴을 강제한다.

### 2. `Object.is` 동일성 검사
업데이터가 같은 참조를 반환하면 **onChange도 리스너도 호출하지 않는다**. 조건부 업데이트(`prev.mode === 'plan' ? {...prev, ...} : prev`)에서 불필요한 리렌더를 막는 핵심.

### 3. `onChange` 단일 진입점
스토어가 바뀔 때 **반드시** 실행되는 콜백 1개. `onChangeAppState` 가 여기에 연결되어:
- CCR 원격 메타데이터 동기화
- 설정 파일 저장
- auth 캐시 무효화
등을 한 곳에서 처리한다. 분산된 `useEffect` 에 의존하지 않음.

### 4. 리스너 Set
React 외 코드도 구독 가능. `useSyncExternalStore` 가 이 `subscribe`/`getState` 를 그대로 먹는다.

## 제네릭 컨테이너

`Store<T>` 는 `AppState` 에 고정된 게 아니다. 같은 primitive가:
- `AppStateStore = Store<AppState>` — 메인 스토어
- `pluginCommandsStore` — 플러그인 커맨드용 작은 스토어 ([src/state/pluginCommandsStore.ts](../../src/state/pluginCommandsStore.ts))

## 왜 외부 라이브러리를 안 쓰나

- **번들 크기**: CLI 는 번들 크기가 중요 (bun single-file executable)
- **단순함**: listener 등록/해제 + onChange 1개면 충분
- **타입 단순성**: 슬라이스·미들웨어·selectors 계층 없이 `AppState` 를 직접 다룸
