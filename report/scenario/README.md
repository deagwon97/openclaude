# OpenClaude 동작 시나리오 — 폴더 읽기 순서

사용자가 `openclaude`를 실행한 순간부터 응답을 받기까지의 흐름을 따라가며 [../](../) 의 폴더들을 읽는 순서를 정리한 문서.

각 폴더 자체는 정적인 주제별 정리이지만, **실제 런타임에서 어떤 순서로 동작하는가**를 따라가면 시스템 전체가 한 줄로 꿰어진다.

---

## 단계별 폴더 읽기 순서

| 단계 | 폴더 | 시점 | 한 줄 요약 |
|------|------|------|-----------|
| 1 | [../state/](../state/) | 부팅 직후 | 모든 subsystem이 공유하는 `AppState` 스토어 초기화 |
| 2 | [../memory/](../memory/) | 부팅 직후 | history.jsonl, 세션 트랜스크립트, MEMORY.md 로드 |
| 3 | [../rules/](../rules/) | `.claude/` 스캔 | CLAUDE.md / `.claude/rules/*.md` 발견·파싱 |
| 4 | [../commands/](../commands/) | `.claude/` 스캔 | 내장 슬래시 커맨드 + `.claude/commands/*.md` |
| 5 | [../skills/](../skills/) | `.claude/` 스캔 | bundled skills + `.claude/skills/**/SKILL.md` |
| 6 | [../agents/](../agents/) | `.claude/` 스캔 | 내장 에이전트 + `.claude/agents/*.md` |
| 7 | [../hooks/](../hooks/) | `settings.json` 로드 | 27개 라이프사이클 이벤트에 hook 매칭 |
| 8 | [../tools/](../tools/) | 도구 등록 | 내장 도구 + 위에서 로드한 사용자 정의가 합쳐짐 |
| 9 | [../context/](../context/) | 사용자 입력 시점 | 메시지·CLAUDE.md·git 상태가 system prompt로 조립 |
| 10 | [../query/](../query/) | API 호출 | `query()` 진입 → `queryLoop()` → 응답 스트리밍 |
| 11 | [../plan_mode/](../plan_mode/) | 선택적 | Shift+Tab / `/plan` 시 활성화되는 2단계 워크플로우 |

---

## 내장(Built-in) 자산 카탈로그

각 카테고리는 **사용자 정의(`.claude/...`)** + **CLI에 번들된 내장(built-in)** 두 소스에서 합쳐진다. 시나리오를 따라가며 어떤 내장 자산이 함께 등록되는지를 표로 정리.

### 내장 Agents — [src/tools/AgentTool/built-in/](../../src/tools/AgentTool/built-in/)
| 이름 | 파일 | 활성 조건 |
|------|------|-----------|
| `general-purpose` | [generalPurposeAgent.ts](../../src/tools/AgentTool/built-in/generalPurposeAgent.ts) | 항상 |
| `statusline-setup` | [statuslineSetup.ts](../../src/tools/AgentTool/built-in/statuslineSetup.ts) | 항상 |
| `Explore` | [exploreAgent.ts](../../src/tools/AgentTool/built-in/exploreAgent.ts) | `BUILTIN_EXPLORE_PLAN_AGENTS` feature flag |
| `Plan` | [planAgent.ts](../../src/tools/AgentTool/built-in/planAgent.ts) | 〃 |
| `claude-code-guide` | [claudeCodeGuideAgent.ts](../../src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts) | 비-SDK 엔트리포인트 |
| `verification` | [verificationAgent.ts](../../src/tools/AgentTool/built-in/verificationAgent.ts) | `VERIFICATION_AGENT` flag |

등록 진입점: [builtInAgents.ts:22 `getBuiltInAgents()`](../../src/tools/AgentTool/builtInAgents.ts#L22)

### 내장 Skills — [src/skills/bundled/](../../src/skills/bundled/)
| Skill | 파일 | 활성 조건 |
|-------|------|-----------|
| `update-config` | [updateConfig.ts](../../src/skills/bundled/updateConfig.ts) | 항상 |
| `keybindings-help` | [keybindings.ts](../../src/skills/bundled/keybindings.ts) | 항상 |
| `debug` | [debug.ts](../../src/skills/bundled/debug.ts) | 항상 |
| `simplify` | [simplify.ts](../../src/skills/bundled/simplify.ts) | 항상 |
| `batch` | [batch.ts](../../src/skills/bundled/batch.ts) | 항상 |
| `loop` | [loop.ts](../../src/skills/bundled/loop.ts) | 항상 (visibility는 KAIROS 게이트) |
| `dream` | [dream.ts](../../src/skills/bundled/) | `KAIROS` / `KAIROS_DREAM` flag |
| `hunter` | hunter.ts | `REVIEW_ARTIFACT` flag |
| `scheduleRemoteAgents` | [scheduleRemoteAgents.ts](../../src/skills/bundled/scheduleRemoteAgents.ts) | `AGENT_TRIGGERS_REMOTE` flag |
| `claude-api` | [claudeApi.ts](../../src/skills/bundled/claudeApi.ts) | `BUILDING_CLAUDE_APPS` flag |
| `claudeInChrome` | [claudeInChrome.ts](../../src/skills/bundled/claudeInChrome.ts) | 자동 활성화 조건 |
| `runSkillGenerator` | runSkillGenerator.ts | `RUN_SKILL_GENERATOR` flag |

등록 진입점: [bundled/index.ts `initBundledSkills()`](../../src/skills/bundled/index.ts)

### 내장 Commands — [src/commands/](../../src/commands/) (113개+)
[commands.ts](../../src/commands.ts) 상단에서 모두 import. 주요 그룹:

| 그룹 | 커맨드 예시 |
|------|-------------|
| 세션 제어 | `/clear`, `/resume`, `/compact`, `/context`, `/memory`, `/rewind`, `/share` |
| 워크플로우 | `/init`, `/plan`, `/fast`, `/passes`, `/review`, `/security-review`, `/bughunter` |
| Git/PR | `/commit`, `/commit-push-pr`, `/diff`, `/pr_comments`, `/autofix-pr`, `/auto-fix` |
| 설정 | `/config`, `/permissions`, `/hooks`, `/theme`, `/vim`, `/model`, `/output-style`, `/keybindings` |
| 확장 | `/mcp`, `/agents`, `/skills`, `/plugin`, `/install-github-app`, `/install-slack-app` |
| 정보 | `/help`, `/status`, `/doctor`, `/usage`, `/cost`, `/version`, `/release-notes` |
| 온보딩 | `/onboarding`, `/onboard-github`, `/login`, `/logout`, `/privacy-settings` |
| 실험적 | `/dream`, `/brief`, `/assistant`, `/voice`, `/bridge`, `/ultraplan` (각각 feature flag) |

### 내장 Tools — [src/tools/](../../src/tools/) (44개+)
[../tools/](../tools/) 문서가 카테고리별로 모두 다룸. 핵심 분류:
- 파일 시스템: `Read`, `Edit`, `Write`, `Glob`, `Grep`, `NotebookEdit`
- 실행: `Bash`, `PowerShell`
- 에이전트/태스크: `Agent`, `TaskCreate/Get/List/Update/Output/Stop`
- 웹: `WebFetch`, `WebSearch`
- 계획: `EnterPlanMode`, `ExitPlanMode`, `EnterWorktree`, `ExitWorktree`, `TodoWrite`
- 사용자 상호작용: `AskUserQuestion`, `Brief`
- 스케줄링: `ScheduleCron`, `RemoteTrigger`, `Sleep`, `Monitor`
- MCP: `MCPTool`, `McpAuth`, `ReadMcpResource`, `ListMcpResources`
- 개발지원: `Config`, `LSPTool`, `REPLTool`, `ToolSearch`
- 조건부: `SendMessage`, `Workflow`, `SyntheticOutput`, `SuggestBackgroundPR`, `TeamCreate`, `TeamDelete`, `Tungsten`, `VerifyPlanExecution`

### 내장 Hooks — `builtinHook` source
[hooksConfigManager.ts:356](../../src/utils/hooks/hooksConfigManager.ts#L356)에 `source: 'builtinHook'` 케이스가 존재하지만, 현재는 `USER_TYPE === 'ant'` (Anthropic 내부)에서만 활성화. 일반 사용자는 사실상 **내장 hook 없음** — 모든 hook은 `settings.json`(user/project/managed/plugin)에서 로드된다.

### 내장 Rules
별도의 "내장 rule 파일"은 없다. 대신 시스템 프롬프트 자체가 [src/constants/prompts.ts](../../src/constants/prompts.ts)에 하드코딩된 정책 블록들의 모음이고, 사용자 rule은 그 위에 `claudeMd`/`nested_memory`로 얹힌다. `autoMode`에서만 [autoMode.ts:153 `defaultRules`](../../src/cli/handlers/autoMode.ts#L153)로 자동 모드 기본 룰을 주입한다.

### 내장 Plugins — [src/plugins/bundled/](../../src/plugins/bundled/)
[bundled/index.ts](../../src/plugins/bundled/index.ts) `initBuiltinPlugins()`는 현재 스캐폴딩만 존재하고 등록된 플러그인은 없다. `/plugin` UI 토글 대상 자리.

---

## Phase 1 — 부팅 (사용자 입력 전)

### 1. [../state/](../state/) — 전역 상태 스토어
**왜 먼저?** 다른 모든 컴포넌트가 `AppState`를 통해 데이터를 공유한다. `createStore`가 만들어지지 않으면 아무것도 시작할 수 없다.

읽기 순서:
- [01_store.md](../state/01_store.md) — 20줄짜리 Observable 구현
- [02_appstate_shape.md](../state/02_appstate_shape.md) — `AppState` 필드 분류
- [03_provider_and_subscribe.md](../state/03_provider_and_subscribe.md) — React 통합
- [04_on_change_side_effects.md](../state/04_on_change_side_effects.md) — 상태 변경 전파
- [05_selectors.md](../state/05_selectors.md) — 파생 상태 추출

### 2. [../memory/](../memory/) — 영구 메모리 로드
**왜 두 번째?** state 초기화 직후 `~/.claude/history.jsonl`과 세션 트랜스크립트를 읽어 상태에 채워 넣는다. 자동 메모리 디렉토리도 이 시점에 스캔된다.

읽기 순서:
- [01_conversation_history.md](../memory/01_conversation_history.md) — 전역 히스토리
- [05_session_vs_persistent.md](../memory/05_session_vs_persistent.md) — 세션 vs 영구 메모리 구분
- 03/04 (메시지 큐, 토큰 측정)은 Phase 4에서 다시 등장

---

## Phase 2 — `.claude/` 디렉토리 스캔

순서는 **system prompt 조립에 들어가는 우선순위**를 따른다.

### 3. [../rules/](../rules/) — 규칙·CLAUDE.md
**먼저 읽어야 하는 이유:** rules는 단순히 한 종류의 설정 파일이 아니라 **CLAUDE.md 패밀리 전체**(memory 파일)의 처리 메커니즘을 다룬다. 다른 `.claude/` 자산들이 이 메모리 위에 얹힌다.

핵심 흐름:
- [01_overview.md](../rules/01_overview.md) — rules vs CLAUDE.md
- [03_load_flow.md](../rules/03_load_flow.md) — eager vs dynamic 로드
- [04_injection.md](../rules/04_injection.md) — system prompt 주입 vs `nested_memory` attachment
- [08_output_styles.md](../rules/08_output_styles.md) — `.claude/output-styles/`

### 4. [../commands/](../commands/) — 슬래시 커맨드
- [01-overview.md](../commands/01-overview.md) — 전체 파이프라인
- [02-discovery.md](../commands/02-discovery.md) — 파일 발견
- [04-registration.md](../commands/04-registration.md) — 레지스트리 등록
- [06-special-syntax.md](../commands/06-special-syntax.md) — `$ARGUMENTS`, `!`bash``, `@file`

### 5. [../skills/](../skills/) — Skills
**commands 다음에 읽는 이유:** Skills는 commands 로더(`loadSkillsDir`)와 코드를 공유하고, "모델이 호출하는 Skill 도구"라는 한 단계 위 레이어다.
- [01-loading.md](../skills/01-loading.md)
- [02-format.md](../skills/02-format.md)
- [04-execution-modes.md](../skills/04-execution-modes.md) — inline vs fork

### 6. [../agents/](../agents/) — 커스텀 서브에이전트
- [01_definition.md](../agents/01_definition.md)
- [02_loading.md](../agents/02_loading.md)
- [03_registration.md](../agents/03_registration.md) — AgentTool에 등록
- [04_execution.md](../agents/04_execution.md) — `runAgent()` 루프

### 7. [../hooks/](../hooks/) — 라이프사이클 훅
**`.claude/` 스캔 마지막에 읽는 이유:** hook은 위 단계들이 만들어 둔 자산(도구, 커맨드, 에이전트)들의 **실행 라이프사이클**에 끼어드는 mechanism이다.
- [01_events_and_types.md](../hooks/01_events_and_types.md) — 27개 이벤트
- [02_loading_and_matching.md](../hooks/02_loading_and_matching.md)
- [03_execution_engine.md](../hooks/03_execution_engine.md)

### 8. [../tools/](../tools/) — 도구 카탈로그
**왜 여기?** 내장 도구 + Phase 2에서 로드된 사용자 정의 도구가 합쳐져 최종 도구 셋이 결정된다.
- [01_filesystem.md](../tools/01_filesystem.md) ~ [10_conditional.md](../tools/10_conditional.md) — 카테고리별 훑기

---

## Phase 3 — 사용자 입력 → 응답

### 9. [../context/](../context/) — 컨텍스트 조립
사용자가 첫 입력을 보내는 순간:
- [05_system_prompt.md](../context/05_system_prompt.md) — Phase 2에서 모은 모든 자산이 system prompt로 합쳐짐
- [01_message_history.md](../context/01_message_history.md) — 메시지 생성
- [03_tool_context.md](../context/03_tool_context.md) — tool 결과의 컨텍스트 포함

### 10. [../query/](../query/) — 메인 루프
- [01_entry_and_query_params.md](../query/01_entry_and_query_params.md) — 진입점
- [02_query_loop.md](../query/02_query_loop.md) — while 루프
- [03_api_layer.md](../query/03_api_layer.md) — 스트리밍/재시도
- [04_tool_execution.md](../query/04_tool_execution.md) — 병렬 도구 실행
- [05_system_prompt.md](../query/05_system_prompt.md) — 정적/동적 블록 (context의 system_prompt와 cross-reference)
- [06_context_injection.md](../query/06_context_injection.md)
- [07_autocompact.md](../query/07_autocompact.md) — 자동 압축이 루프에 개입하는 지점

다시 [../memory/](../memory/)로 돌아가서:
- [02_context_compression.md](../memory/02_context_compression.md)
- [03_message_queue.md](../memory/03_message_queue.md)
- [04_token_management.md](../memory/04_token_management.md)

---

## Phase 4 — 선택적: Plan Mode

### 11. [../plan_mode/](../plan_mode/)
사용자가 Shift+Tab을 누르거나 `/plan`을 호출할 때만 활성화. Phase 3 위에 얹히는 권한 모드 + 2개 도구 + 파일시스템 조합.
- [01_overview.md](../plan_mode/01_overview.md)
- [02_enter_exit_tools.md](../plan_mode/02_enter_exit_tools.md)
- [03_permissions.md](../plan_mode/03_permissions.md)
- [04_plan_file_and_slug.md](../plan_mode/04_plan_file_and_slug.md)
- [06_interview_phase.md](../plan_mode/06_interview_phase.md) — V2 인터뷰

---

## 한 줄 정리

```
[부팅]
  state ─► memory
            │
            ▼
[.claude/ 스캔]
  rules ─► commands ─► skills ─► agents ─► hooks
                                              │
                                              ▼
[도구 카탈로그 확정]
                                            tools
                                              │
                                              ▼
[사용자 입력]
                                            context ─► query
                                                         │
                                                         ▼
                                                   (선택) plan_mode
```

읽기를 한 번에 끝내고 싶다면 **state → memory → rules → commands → skills → agents → hooks → tools → context → query → plan_mode** 순으로 각 폴더의 README와 01부터 차례로 보면 된다.
