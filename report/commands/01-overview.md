# 01 · 전체 파이프라인 개요

## 한눈에 보는 흐름

```
[시작 시: 커맨드 로드]
  .claude/commands/*.md  (project / user / policy 세 계층)
    ↓ scan
  markdownConfigLoader.loadMarkdownFilesForSubdir('commands', cwd)
    ↓ parse
  frontmatterParser.parseFrontmatter()        ← YAML frontmatter 추출
  loadSkillsDir.parseSkillFrontmatterFields() ← 필드 정규화
    ↓ build
  loadSkillsDir.createSkillCommand()          ← type: 'prompt' 커맨드 객체
    ↓ register (memoized)
  commands.ts.loadAllCommands()               ← 빌트인 + 커스텀 통합 레지스트리

[런타임: 사용자 입력 "/mycommand arg1 arg2"]
  REPL 입력
    ↓
  handlePromptSubmit.handlePromptSubmit()
    ↓
  processUserInput.processUserInputBase()
    ↓
  processSlashCommand.processSlashCommand()
    ├─ slashCommandParsing.parseSlashCommand()    ← {commandName, args}
    ├─ 커맨드 조회 (commands 레지스트리)
    └─ getMessagesForSlashCommand()  ← type에 따라 분기
         ├─ 'local'     → mod.call(args, ctx)
         ├─ 'local-jsx' → React UI + onDone 콜백
         └─ 'prompt'    → getMessagesForPromptSlashCommand()
              ├─ command.getPromptForCommand(args, ctx)
              │    ├─ substituteArguments()         ← $ARGUMENTS, $1 등
              │    ├─ executeShellCommandsInPrompt() ← !`cmd` 실행
              │    └─ 환경 변수 (${CLAUDE_SKILL_DIR}) 치환
              ├─ registerSkillHooks()                ← hooks frontmatter
              └─ messages = [
                   <command-message>,
                   isMeta: skill 본문,
                   @mention 첨부,
                   command_permissions (allowedTools, model)
                 ]
    ↓
  Claude API 호출 (query.ts)
```

## 두 개의 큰 단계

### 1. 로드 단계 (시작 시 · 메모이즈됨)
파일시스템에서 마크다운을 읽어 메모리의 커맨드 객체로 만들어 둡니다. `loadAllCommands`는 `cwd`별로 memoize되므로, 같은 디렉토리에서는 한 번만 스캔합니다.

### 2. 실행 단계 (사용자 입력마다)
입력이 `/`로 시작하면 커맨드 레지스트리를 조회하여 타입에 따라 분기합니다. 커스텀 `.md` 커맨드는 거의 항상 `type: 'prompt'`로, 본문을 변수 치환·셸 실행 처리한 뒤 Claude에게 메시지로 전달됩니다.

## 핵심 개념

| 개념 | 설명 |
|------|------|
| **Skill = Command** | 내부적으로 `.claude/commands/`와 `.claude/skills/`는 같은 로더(`loadSkillsDir.ts`)를 공유. 커맨드는 `loadedFrom: 'commands_DEPRECATED'`로 마킹됨 |
| **type: 'prompt'** | 커스텀 `.md` 커맨드의 기본 타입. 본문이 Claude 메시지로 변환됨 |
| **isMeta 메시지** | 사용자 UI에는 감춰지고 모델에게만 보이는 메시지. 커맨드 본문이 이 형태로 전달됨 |
| **inline vs fork** | `context: fork` frontmatter가 있으면 서브에이전트에서 실행, 없으면 현재 대화에 인라인 주입 |
| **조건부 활성화** | `paths:` frontmatter가 있으면 해당 경로가 터치될 때만 동적 활성화 |

## 관련 문서

- 다음: [02-discovery.md](02-discovery.md) — 파일을 어디서 어떻게 찾는지
