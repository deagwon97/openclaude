# 05. 셀렉터 — 파생 상태 추출

[src/state/selectors.ts](../../src/state/selectors.ts) 는 `AppState` 에서 자주 쓰는 파생 값을 뽑는 **순수 함수** 모음이다.

```
/**
 * Keep selectors pure and simple — just data extraction, no side effects.
 */
```

## 설계 원칙

- **순수**: 입력만 보고 결정, 로깅·I/O·상태 mutation 없음
- **타입 좁히기**: AppState 필드의 union 타입을 `isInProcessTeammateTask` 같은 타입 가드로 좁혀 반환
- **Pick 타입**: 필요한 필드만 받아서 의존성 명시

## 1. `getViewedTeammateTask(appState)`

[src/state/selectors.ts:18-40](../../src/state/selectors.ts#L18-L40)

```typescript
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>,
): InProcessTeammateTaskState | undefined
```

반환 규칙:
- `viewingAgentTaskId` 가 없으면 `undefined`
- 해당 ID의 task 가 없으면 `undefined`
- task 가 있어도 `InProcessTeammateTaskState` 타입이 아니면 `undefined`
- 위 조건 모두 통과하면 그 task 반환 (타입 가드로 narrow 됨)

주목: **Pick** 으로 받는다. `AppState` 전체 대신 사용할 필드만 명시 — 테스트 작성 쉽고 의존성이 명확.

## 2. `getActiveAgentForInput(appState)` — 입력 라우팅

[src/state/selectors.ts:59-76](../../src/state/selectors.ts#L59-L76)

사용자가 프롬프트를 입력했을 때 **어느 에이전트의 컨텍스트로 갈지** 결정.

```typescript
type ActiveAgentForInput =
  | { type: 'leader' }                              // 기본
  | { type: 'viewed'; task: InProcessTeammateTaskState }  // teammate 뷰잉 중
  | { type: 'named_agent'; task: LocalAgentTaskState }    // local agent 뷰잉 중
```

결정 로직:
1. `getViewedTeammateTask` 가 반환되면 `'viewed'`
2. `viewingAgentTaskId` 는 있는데 teammate 가 아니고 `local_agent` 타입이면 `'named_agent'`
3. 아무것도 없으면 `'leader'`

**Discriminated union** 덕에 호출측은 `switch (result.type)` 로 완전한 타입 안전 분기 가능.

## 왜 셀렉터가 적은가

OpenClaude 는 **셀렉터 계층을 얇게** 유지한다. Redux/zustand 의 reselect 같은 메모이제이션 계층은 없다. 이유:

- `useAppState(selector)` 자체가 `Object.is` 로 재렌더를 억제 — slice 안정성만 보장하면 됨
- 대부분의 UI 는 기존 서브 객체 참조를 바로 쓴다 (`s.mcp`, `s.plugins.enabled`)
- 계산 비용이 큰 파생값은 드물다 — 대부분 단순 lookup

이 파일에 있는 2개만 "여러 곳에서 재사용 + 타입 narrow 로직이 복잡" 해서 추출됐다.

## 확장 가이드

새 셀렉터를 추가할 때:
1. **순수하게** — `AppState` 외부 의존 없음
2. **Pick 타입** 으로 받기 — 전체 AppState 대신 사용 필드만
3. **Discriminated union** 으로 반환 — 호출측이 타입 안전하게 분기 가능
4. **사이드이펙트는 `onChangeAppState` 로** — 셀렉터는 읽기 전용
