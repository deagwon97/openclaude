# 03 · 파싱 (Frontmatter & 본문)

발견된 `.md` 파일에서 YAML frontmatter를 추출하고, 필드를 커맨드 객체에서 쓸 수 있는 형태로 정규화하는 단계입니다.

## 파일 구조

```markdown
---
description: "커맨드 한 줄 설명"
allowed-tools: ["Bash", "Read"]
argument-hint: "[file_path] [env]"
arguments: "file env"          # 명명된 인수 (공백 구분)
model: "sonnet"
when_to_use: "사용 시나리오"
user-invocable: true
disable-model-invocation: false
context: "inline"              # 또는 "fork"
agent: "Bash"                  # context: fork 일 때 사용할 서브에이전트
effort: "low"                  # thinking effort
paths: "src/**/*.ts"           # 조건부 활성화
hooks: { ... }                 # 훅 설정
shell: "bash"                  # 또는 "powershell"
---
# 본문 마크다운
Deploy $file to $env environment.

Current branch: !`git branch --show-current`

See @README.md for context.
```

## 파싱 단계

### Step 1 — YAML Frontmatter 추출

**파일**: [src/utils/frontmatterParser.ts](../../src/utils/frontmatterParser.ts)

- **`parseFrontmatter(raw)`**
  - `---` 경계 기준으로 frontmatter 블록 분리
  - YAML 파싱 시도 → 실패하면 특수 문자를 자동으로 인용(quote)한 뒤 재시도 (사용자 친화성)
  - 반환: `{ frontmatter: FrontmatterData, content: string }`

### Step 2 — 필드 정규화

**파일**: [src/skills/loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts) · `parseSkillFrontmatterFields()`

Raw frontmatter를 커맨드 객체의 표준 필드로 변환합니다. 주요 변환:

| frontmatter 필드 | 정규화 결과 | 비고 |
|------------------|------------|------|
| `description` | `description` | 없으면 본문 첫 줄에서 추론 |
| — | `hasUserSpecifiedDescription` | 명시 여부 플래그 |
| `allowed-tools` | `allowedTools: string[]` | `parseSlashCommandToolsFromFrontmatter()` |
| `argument-hint` | `argumentHint` | UI 힌트 텍스트 |
| `arguments` | `argumentNames: string[]` | 공백 구분해서 배열화 |
| `model` | `model` | `parseUserSpecifiedModel()` — 별칭 해석 |
| `when_to_use` | `whenToUse` | 모델이 스스로 선택할 때 쓰는 힌트 |
| `user-invocable` | `userInvocable` | `false`면 `/`로 호출 불가 |
| `disable-model-invocation` | `disableModelInvocation` | 모델이 자동 선택 금지 |
| `context` | `executionContext: 'inline' \| 'fork'` | — |
| `agent` | `agent` | fork 시 사용할 서브에이전트 타입 |
| `effort` | `effort` | thinking effort 레벨 |
| `hooks` | `hooks` | 실행 시 임시 등록될 훅 |
| `shell` | `shell: 'bash' \| 'powershell'` | `!`cmd`` 실행 환경 |

### Step 3 — 본문 보관

본문(마크다운)은 이 단계에서는 **치환하지 않고** 원본 그대로 보관합니다. `$ARGUMENTS`, `!`cmd``, `@file` 같은 특수 문법은 **실행 시점**(사용자가 호출할 때) 처리됩니다 — 자세한 내용은 [06-special-syntax.md](06-special-syntax.md) 참조.

## 설명(description)의 우선순위

1. frontmatter `description` 필드 → 그대로 사용 (`hasUserSpecifiedDescription = true`)
2. 없으면 → 본문의 첫 의미 있는 줄을 description으로 추론 (`hasUserSpecifiedDescription = false`)

이 플래그는 나중에 UI 표시나 모델이 선택할 때 신뢰도 계산에 쓰입니다.

## 에러 내성

- YAML 파싱 실패 → 특수문자 자동 인용 후 재시도
- frontmatter 자체가 없음 → 빈 객체로 처리, 본문만 사용
- 필수 필드 없음 → 기본값으로 채움 (예: `userInvocable = true`)

결과적으로 **거의 모든 `.md` 파일이 커맨드로 등록**될 수 있도록 관대하게 파싱합니다.

## 관련 문서

- 이전: [02-discovery.md](02-discovery.md)
- 다음: [04-registration.md](04-registration.md) — 파싱된 데이터로 커맨드 객체 만들기
