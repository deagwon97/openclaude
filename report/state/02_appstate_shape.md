# 02. AppState 형태 — 필드별 분류

[src/state/AppStateStore.ts](../../src/state/AppStateStore.ts) 의 `AppState` 타입은 **569줄**에 걸친 100개 이상의 필드를 가진 대규모 객체다. 하네스의 거의 모든 런타임 상태가 여기 모여 있다.

```typescript
export type AppState = DeepImmutable<{
  // 100+ 개의 immutable 필드
}> & {
  // immutable로 만들 수 없는 필드 (Map, function, 상호 참조 등)
}
```

`DeepImmutable` 로 immutable 부분과 mutable 부분을 분리. `tasks` 처럼 함수 타입을 포함하는 건 immutable 영역 밖으로 뺀다.

## 필드 분류

### A. 설정 & 세션 정체성

| 필드 | 용도 |
|------|------|
| `settings` | 병합된 전역/프로젝트/로컬 settings.json |
| `verbose` | --verbose 플래그 |
| `mainLoopModel` | 메인 루프 모델 (opus/sonnet/null=default) |
| `mainLoopModelForSession` | 세션 단위 모델 오버라이드 |
| `agent` | --agent CLI 플래그로 지정된 에이전트 이름 |
| `kairosEnabled` | Assistant mode (settings + gate + trust) — 단일 진실 |

### B. 권한 & 퍼미션

| 필드 | 용도 |
|------|------|
| `toolPermissionContext` | 퍼미션 모드, 허용/거부 규칙, prePlanMode, strippedDangerousRules |
| `denialTracking` | 분류기 모드의 거부 카운터 (YOLO, headless) |
| `workerSandboxPermissions` | 리더 측 워커 샌드박스 승인 큐 |
| `pendingWorkerRequest` / `pendingSandboxRequest` | 워커 측 대기 중 승인 요청 |

### C. 태스크 & 서브에이전트

| 필드 | 용도 |
|------|------|
| `tasks` | `{ [taskId]: TaskState }` — 모든 background task (Agent, teammate, workflow) |
| `agentNameRegistry` | `name → AgentId` 매핑 (SendMessage 라우팅용) |
| `foregroundedTaskId` | 메인뷰에 표시되는 태스크 |
| `viewingAgentTaskId` | 트랜스크립트 뷰잉 중인 태스크 |
| `todos` | `{ [agentId]: TodoList }` — 에이전트별 TodoWrite 리스트 |

### D. MCP & 플러그인

| 필드 | 용도 |
|------|------|
| `mcp.clients` / `mcp.tools` / `mcp.commands` / `mcp.resources` | MCP 서버 연결 상태 |
| `mcp.pluginReconnectKey` | `/reload-plugins` 트리거 키 |
| `plugins.enabled` / `plugins.disabled` | 로드된 플러그인 |
| `plugins.errors` | 플러그인 로드/초기화 에러 |
| `plugins.installationStatus` | 백그라운드 설치 진행 상태 |
| `plugins.needsRefresh` | on-disk 변경 감지 플래그 |

### E. UI 뷰 상태

| 필드 | 용도 |
|------|------|
| `expandedView` | 'none' / 'tasks' / 'teammates' |
| `viewSelectionMode`, `selectedIPAgentIndex`, `coordinatorTaskIndex` | Agent 선택/뷰잉 상태머신 |
| `footerSelection` | footer pill 포커스 |
| `activeOverlays` | Escape 키 coordination |
| `statusLineText`, `spinnerTip` | 하단 상태바 |

### F. 원격 세션 (Bridge / Assistant)

| 필드 | 용도 |
|------|------|
| `remoteSessionUrl`, `remoteConnectionStatus` | --remote viewer 모드 |
| `remoteBackgroundTaskCount` | 원격 데몬의 백그라운드 태스크 수 |
| `replBridge*` (10여개) | Always-on bridge 의 상세 상태머신 |
| `replBridgePermissionCallbacks`, `channelPermissionCallbacks` | 원격 경로에서의 퍼미션 콜백 |

### G. 플랜 모드

| 필드 | 용도 |
|------|------|
| `initialMessage` | 플랜 모드 exit 후 처리할 초기 메시지 + allowedPrompts |
| `pendingPlanVerification` | 배경 검증 상태 (`VerifyPlanExecution` 트리거용) |
| `isUltraplanMode` | 원격 세션 측 ultraplan 플래그 |
| `ultraplanLaunching`, `ultraplanSessionUrl`, `ultraplanPendingChoice`, `ultraplanLaunchPending` | Ultraplan 플로우 상태 |

### H. Speculation & Prompt Suggestion

| 필드 | 용도 |
|------|------|
| `speculation` | 유휴 상태에서 LLM 이 다음 응답을 예측 실행하는 기능 |
| `speculationSessionTimeSavedMs` | 누적 절약 시간 |
| `promptSuggestionEnabled` / `promptSuggestion` | 제안 프롬프트 생성 결과 |
| `thinkingEnabled` | extended thinking 토글 |

### I. 팀 & Coordinator

| 필드 | 용도 |
|------|------|
| `teamContext` | Agent Swarms 의 팀 정보 (leader, tmux panes, teammates) |
| `standaloneAgentContext` | 단독 에이전트의 이름/색 |
| `inbox` | 메시지 수신함 (에이전트간 메시지) |

### J. 파일 & 기여 추적

| 필드 | 용도 |
|------|------|
| `fileHistory` | 스냅샷/추적 파일 (undo 용) |
| `attribution` | Co-author 추적 |

### K. 기타 내부 기능

| 필드 | 조건 |
|------|------|
| `tungsten*` | tmux 통합 (ant only) |
| `bagel*` | WebBrowser tool |
| `computerUseMcpState` | chicago MCP (ant only) |
| `replContext` | REPL tool VM 컨텍스트 |
| `companionReaction`, `companionPetAt` | `/buddy` 감상평 |
| `effortValue`, `fastMode`, `advisorModel` | 내부 기능 토글 |

## getDefaultAppState()

[src/state/AppStateStore.ts:456-569](../../src/state/AppStateStore.ts#L456-L569)

모든 필드의 초기값을 정의. 주목할 점:

- `initialMode: PermissionMode` 는 teammate + `plan_mode_required` 여부에 따라 `'plan'` 또는 `'default'` 로 결정됨
- `thinkingEnabled: shouldEnableThinkingByDefault()` — gate 기반 조건부
- `promptSuggestionEnabled: shouldEnablePromptSuggestion()` — 동일

초기값 계산을 위해 `teammate.ts` 를 **lazy require** — 순환 의존성 회피.

## 왜 이 크기인가

- **단일 소스의 진실**: UI 렌더·QueryEngine·Tool 실행·원격 동기화가 동일 상태를 공유해야 함
- **Session resume**: 세션 재개 시 한 번에 복원할 수 있음
- **Observer 단순화**: 변경 지점 1곳 (`onChangeAppState`) 에서 모든 downstream 파급 처리
