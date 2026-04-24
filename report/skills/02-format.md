# 02. SKILL.md 파일 포맷과 Frontmatter

## 2.1 기본 구조

Skill 파일은 YAML frontmatter + Markdown 본문 형태이다.

```markdown
---
description: Commit changes with conventional commit format
when_to_use: 사용자가 "commit", "커밋" 등을 요청할 때
allowed-tools: Bash, Read
argument-hint: "-m '<message>'"
user-invocable: true
model: sonnet
context: inline
---

# Commit Skill

현재 변경사항을 분석해서 conventional commit 형식으로 커밋해.

## Steps
1. `git status` 로 변경사항 확인
2. `git diff --cached` 로 스테이지된 내용 검토
3. ...
```

Frontmatter 파싱은 [frontmatterParser.ts](../../src/utils/frontmatterParser.ts) 의 `parseFrontmatter()` 가 담당하며, 매칭 정규식은:

```ts
export const FRONTMATTER_REGEX = /^---\s*\n([\s\S]*?)---\s*\n?/
```

필드별 의미 해석은 [loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts) 의 `parseSkillFrontmatterFields()` 에서 이루어진다.

## 2.2 Frontmatter 필드 정의

| 필드 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| `description` | string | *(필수)* | Skill 의 한 줄 설명. 모델이 skill 선택 시 참고 |
| `when_to_use` | string | - | 이 skill 을 언제 사용해야 하는지 |
| `allowed-tools` | string \| string[] | `[]` | Skill 실행 중 허용되는 tool 목록 |
| `argument-hint` | string | - | 사용자에게 표시될 인자 힌트 (slash command UI 용) |
| `user-invocable` | boolean | `false` | `/skill-name` 으로 사용자가 직접 호출 가능한지 |
| `disable-model-invocation` | boolean | `false` | 모델이 SkillTool 로 호출할 수 없도록 차단 |
| `model` | string | *(상속)* | 이 skill 실행 시 사용할 모델 오버라이드 (e.g. `haiku`, `opus`) |
| `context` | `"inline"` \| `"fork"` | `"inline"` | 실행 모드 — [04](04-execution-modes.md) 참조 |
| `agent` | string | - | `context: fork` 일 때 사용할 서브에이전트 타입 |
| `effort` | `"low" \| "medium" \| "high" \| "max"` 또는 정수 | - | Forked 에이전트의 thinking effort |
| `hooks` | HooksSettings | - | Skill 실행 중 자동 등록될 훅 — [05](05-hooks-and-permissions.md) |
| `paths` | string \| string[] | - | Conditional skill 활성화 조건 (glob) |
| `shell` | `"bash"` \| `"powershell"` | `"bash"` | Markdown 내 `` !`cmd` `` 블록 실행 셸 |
| `version` | string | - | Skill 버전 정보 |

## 2.3 Markdown 본문에서 쓸 수 있는 치환자

`createSkillCommand()` 가 생성하는 `getPromptForCommand()` 클로저는 본문을 그대로 넘기지 않고 다음 치환을 수행한다.

- `$ARGUMENTS` — 사용자가 slash command 뒤에 붙인 전체 인자 문자열
- `$1`, `$2`, ... — 공백 기준으로 분할된 위치 인자
- `${CLAUDE_SKILL_DIR}` — 이 skill의 디렉토리 절대경로 (참조 파일 로드용)
- `${CLAUDE_SESSION_ID}` — 현재 세션 ID
- `` !`<command>` `` — `shell` 필드가 지정한 셸로 즉시 실행하고 stdout 을 본문에 삽입

이 덕분에 skill 안에서 helper 스크립트를 `${CLAUDE_SKILL_DIR}/helper.py` 형태로 참조할 수 있다.

## 2.4 `Command` 객체로의 변환

파싱 완료 후 최종적으로 [types/command.ts](../../src/types/command.ts) 의 `PromptCommand` 형태가 된다.

```ts
type PromptCommand = CommandBase & {
  type: 'prompt'
  name: string                          // 디렉토리 이름 = skill 이름
  description: string
  source: 'builtin' | 'plugin' | 'plugin:bundled' | 'bundled'
  loadedFrom: 'skills' | 'commands_DEPRECATED' | 'plugin' | 'managed' | 'bundled' | 'mcp'

  skillRoot?: string                    // SKILL.md 가 있는 디렉토리
  context?: 'inline' | 'fork'
  agent?: string
  hooks?: HooksSettings
  effort?: EffortValue
  paths?: string[]
  argNames?: string[]

  userInvocable: boolean
  hasUserSpecifiedDescription: boolean
  disableModelInvocation?: boolean

  // 호출 시점에 markdown 본문을 렌더해서 반환
  getPromptForCommand(
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>
}
```

중요한 점: **본문 파싱은 로드 시점이 아니라 호출 시점에 이루어진다** (`getPromptForCommand` 클로저 안). 이 덕분에 인자 치환이나 `!` `` 셸 주입이 호출 컨텍스트에 맞게 동작한다.
