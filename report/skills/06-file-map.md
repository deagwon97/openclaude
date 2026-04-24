# 06. 핵심 파일 & 함수 빠른 참조 맵

Skills 관련 구현이 분산되어 있어, 이 페이지는 "어디에 뭐가 있는지" 를 한눈에 볼 수 있게 정리한 참조 시트이다. 줄 번호는 조사 시점 기준이며, 이후 변경될 수 있다.

## 6.1 디렉토리

- [src/skills/](../../src/skills/) — Skill 로딩, bundled skill 정의
- [src/tools/SkillTool/](../../src/tools/SkillTool/) — SkillTool 정의, 프롬프트, UI
- [src/utils/processUserInput/](../../src/utils/processUserInput/) — slash command 처리
- [src/utils/hooks/](../../src/utils/hooks/) — 세션 훅 등록 / 저장소
- [src/utils/frontmatterParser.ts](../../src/utils/frontmatterParser.ts) — Frontmatter YAML 파싱
- [src/commands.ts](../../src/commands.ts) — 모든 Command 통합 로더
- [src/constants/prompts.ts](../../src/constants/prompts.ts) — 시스템 프롬프트 조립

## 6.2 파일별 핵심 함수

### [src/skills/loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts)

| 함수 | 역할 |
|---|---|
| `getSkillDirCommands(cwd)` | 모든 소스(managed/user/project/additional)에서 skill 로드. memoized |
| `loadSkillsFromSkillsDir(dir, source)` | 하나의 `.claude/skills` 디렉토리를 재귀 스캔 |
| `loadSkillsFromCommandsDir(cwd)` | 레거시 `.claude/commands/` 호환 로더 |
| `findSkillMarkdownFiles(basePath)` | `SKILL.md` 재귀 검색 (심볼릭 링크 추적) |
| `parseSkillFrontmatterFields(fm, md, name)` | Frontmatter → 타입드 필드 변환, hooks 스키마 검증 |
| `createSkillCommand({...})` | 최종 `Command` 객체 생성 + `getPromptForCommand` 클로저 주입 |
| `discoverSkillDirsForPaths(paths, cwd)` | 파일 경로들로부터 동적으로 `.claude/skills` 발견 |
| `addSkillDirectories(dirs)` | 발견된 디렉토리를 `dynamicSkills` 맵에 병합 |
| `conditionalSkills` (모듈 변수) | `paths` frontmatter 가 있는 skill 의 보관소 |

### [src/tools/SkillTool/SkillTool.ts](../../src/tools/SkillTool/SkillTool.ts)

| 심볼 | 역할 |
|---|---|
| `SkillTool` (export) | `buildTool(...)` 로 만든 tool 인스턴스 |
| `inputSchema` | `{ skill: string, args?: string }` Zod 스키마 |
| `SkillTool.validateInput()` | skill 존재/유형 검증 |
| `SkillTool.checkPermissions()` | deny/allow/safe-properties 기반 권한 결정 |
| `SkillTool.call()` | inline vs fork 분기 |
| `executeForkedSkill(...)` | `runAgent()` 로 서브에이전트 실행, `skill_progress` 스트리밍 |
| `prepareForkedCommandContext(...)` | Forked 실행용 에이전트/메시지/컨텍스트 준비 |

### [src/tools/SkillTool/prompt.ts](../../src/tools/SkillTool/prompt.ts)

| 함수 / 상수 | 역할 |
|---|---|
| `getPrompt(cwd)` | SkillTool 의 설명 프롬프트 (memoized) |
| `formatCommandsWithinBudget(commands, ctxTokens)` | 토큰 예산 안에서 skill 목록 포맷 |
| `getCharBudget(ctxTokens)` | 예산 계산 (`ctxTokens * 4 * 0.01`, 기본 8000) |
| `SKILL_BUDGET_CONTEXT_PERCENT` = `0.01` | 컨텍스트의 1% 할당 |
| `DEFAULT_CHAR_BUDGET` = `8_000` | 기본 예산 |
| env `SLASH_COMMAND_TOOL_CHAR_BUDGET` | 예산 override |

### [src/utils/processUserInput/processSlashCommand.tsx](../../src/utils/processUserInput/processSlashCommand.tsx)

| 함수 | 역할 |
|---|---|
| `processPromptSlashCommand(...)` | `/skill-name` 사용자 입력 및 SkillTool 의 inline 경로 |
| `getMessagesForPromptSlashCommand(cmd, args, ctx)` | skill 본문을 메시지로 변환, hooks 등록, permissions attachment 부착 |
| `executeForkedSlashCommand(...)` | 사용자가 직접 `/skill-name` 을 쳤을 때의 fork 경로 |

### [src/utils/hooks/registerSkillHooks.ts](../../src/utils/hooks/registerSkillHooks.ts)

| 함수 | 역할 |
|---|---|
| `registerSkillHooks(setAppState, sessionId, hooks, name, root)` | skill 의 hooks 필드를 세션 훅으로 등록 (once 지원) |

### [src/utils/hooks/sessionHooks.ts](../../src/utils/hooks/sessionHooks.ts)

| 심볼 | 역할 |
|---|---|
| `SessionHooksState` | 세션별 훅 저장 구조 (`Map<sessionId, SessionStore>`) |
| `addSessionHook(...)` | 세션 훅 추가 |
| `removeSessionHook(...)` | 세션 훅 제거 (`once` 훅 성공 시) |

### [src/utils/frontmatterParser.ts](../../src/utils/frontmatterParser.ts)

| 심볼 | 역할 |
|---|---|
| `FRONTMATTER_REGEX` = `/^---\s*\n([\s\S]*?)---\s*\n?/` | Frontmatter 매칭 정규식 |
| `parseFrontmatter(content, path)` | `{ frontmatter, content }` 반환 |

### [src/skills/bundledSkills.ts](../../src/skills/bundledSkills.ts)

| 함수 | 역할 |
|---|---|
| `registerBundledSkill(def)` | 내장 skill 을 레지스트리에 등록 |
| `getBundledSkills()` | 등록된 내장 skill 목록 반환 |
| `BundledSkillDefinition` (타입) | name/description/whenToUse/allowedTools/hooks/context/agent/files/getPromptForCommand 등 |

내장 skill 구현 예시: [src/skills/bundled/](../../src/skills/bundled/)
- `updateConfig.ts`, `loop.ts`, `scheduleRemoteAgents.ts`, `keybindings.ts`, `claudeApi.ts`, `simplify.ts`

### [src/commands.ts](../../src/commands.ts)

| 함수 | 역할 |
|---|---|
| `getCommands(cwd)` | 모든 command 반환 (entry point) |
| `loadAllCommands(cwd)` | 전체 로딩 오케스트레이션 (memoized) |
| `getSkills(cwd)` | dir + plugin + bundled + builtin plugin skill 통합 |
| `getSkillToolCommands(cwd)` | SkillTool 이 노출할 skill 필터링 |
| `getSlashCommandToolSkills(cwd)` | `/slash` 로 사용자가 직접 호출 가능한 skill 필터링 |
| `clearCommandMemoizationCaches()` | 모든 memoize 캐시 무효화 |

### [src/constants/prompts.ts](../../src/constants/prompts.ts)

| 함수 | 역할 |
|---|---|
| `getSystemPrompt(...)` | 시스템 프롬프트 조립 |
| `getSessionSpecificGuidanceSection(enabledTools, skillToolCommands)` | SkillTool 목록을 시스템 프롬프트 섹션에 주입 |

### [src/types/command.ts](../../src/types/command.ts)

| 타입 | 역할 |
|---|---|
| `Command`, `PromptCommand`, `CommandBase` | Skill 을 포함한 모든 command 의 타입 정의 |

## 6.3 코드 베이스를 읽을 때 추천 순서

1. [src/skills/loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts) — **로딩 파이프라인 전체를 이해**
2. [src/types/command.ts](../../src/types/command.ts) — `Command` 타입 확인
3. [src/tools/SkillTool/SkillTool.ts](../../src/tools/SkillTool/SkillTool.ts) — **호출 지점**
4. [src/utils/processUserInput/processSlashCommand.tsx](../../src/utils/processUserInput/processSlashCommand.tsx) — inline 실행 분기
5. [src/tools/SkillTool/prompt.ts](../../src/tools/SkillTool/prompt.ts) — skill 목록이 시스템 프롬프트에 어떻게 포매팅되는지
6. [src/utils/hooks/registerSkillHooks.ts](../../src/utils/hooks/registerSkillHooks.ts) — 훅 주입
7. [src/skills/bundledSkills.ts](../../src/skills/bundledSkills.ts) — 내장 skill 이 어떻게 같은 파이프라인으로 들어오는지
