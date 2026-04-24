# 06 · 특수 문법 (Arguments · Shell · Mention · Paths)

커스텀 커맨드의 본문은 호출 시점에 여러 가지 특수 문법이 치환·실행됩니다. 이 문서는 각 문법과 그 처리 파일을 정리합니다.

## 1. Arguments 치환

**파일**: [src/utils/argumentSubstitution.ts](../../src/utils/argumentSubstitution.ts) · `substituteArguments()`

### 지원 패턴

| 패턴 | 의미 |
|------|------|
| `$ARGUMENTS` | 입력 인수 전체 문자열 |
| `$ARGUMENTS[0]`, `$ARGUMENTS[1]` | 인덱스 기반 접근 |
| `$0`, `$1`, `$2` | 위치 인수 단축 표기 |
| `$argname` | 명명된 인수 — frontmatter `arguments: env action`처럼 선언 시 |

### 예시

**커맨드 파일**:
```markdown
---
arguments: "env action"
---
Deploy to $env with action $action.
Full args: $ARGUMENTS
First: $0 / $ARGUMENTS[0]
```

**호출**: `/deploy staging rollback`

**치환 결과**:
```
Deploy to staging with action rollback.
Full args: staging rollback
First: staging / staging
```

### 명명된 인수 파싱

frontmatter `arguments: env action`은 공백으로 분리해 `argNames: ['env', 'action']`이 됩니다. 입력된 `args` 문자열도 공백 단위로 잘라 `env=staging`, `action=rollback`으로 매핑됩니다.

## 2. Shell 명령 실행 (`!`cmd``)

**파일**: [src/utils/promptShellExecution.ts](../../src/utils/promptShellExecution.ts) · `executeShellCommandsInPrompt()`

### 지원 패턴

```markdown
인라인:
Current branch: !`git branch --show-current`

블록:
```!
git status
git diff --stat
```
```

### 실행 흐름

1. 본문에서 `!`cmd`` / ```!\n...``` 패턴을 정규식으로 추출
2. frontmatter의 `shell: bash | powershell`에 따라 `BashTool` 또는 `PowerShellTool` 선택
3. `hasPermissionsToUseTool()`로 권한 확인 — 통과해야 실행
4. `shellTool.call({ command }, context)`로 실행
5. 실행 결과를 원래 패턴 자리에 **치환**
6. 치환된 본문이 Claude에게 전달됨

### 보안 제약

- **MCP 스킬은 shell 실행 금지**: 원격에서 온 스킬은 신뢰할 수 없으므로 `!`cmd`` 패턴을 실행하지 않음
- **권한 컨텍스트 준수**: 현재 세션의 도구 권한 규칙을 따름
- **실패 시 `MalformedCommandError`**: 실행 실패하면 스킬 전체가 에러

## 3. 환경 변수

커스텀 커맨드 본문에서 사용 가능한 내장 변수:

| 변수 | 의미 |
|------|------|
| `${CLAUDE_SKILL_DIR}` | 이 커맨드 파일이 있는 디렉토리 절대경로 |
| `${CLAUDE_SESSION_ID}` | 현재 세션 UUID |

주로 `${CLAUDE_SKILL_DIR}/templates/foo.txt`처럼 커맨드 옆의 보조 파일을 참조할 때 사용됩니다.

## 4. 파일 Mention (`@file`)

본문의 `@README.md`, `@src/foo.ts` 같은 mention은 실행 시 **첨부 메시지**로 변환되어 Claude에게 같이 전달됩니다. 처리 경로는 일반 사용자 입력의 mention 처리와 공유됩니다 — 특수한 커맨드 전용 문법이 아니라 일반 prompt의 mention 파이프라인을 재사용합니다.

결과 메시지 구조:
```
[치환된 본문]               ← isMeta user 메시지
[@README.md 내용]           ← attachment 메시지
[@src/foo.ts 내용]          ← attachment 메시지
```

## 5. 조건부 활성화 (`paths:` frontmatter)

**파일**: [src/skills/loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts)

### 목적

매번 활성화되어 컨텍스트를 잡아먹지 않고, **관련 파일이 터치될 때만** 모델이 이 커맨드를 볼 수 있게 하는 기능입니다.

### 동작

```markdown
---
paths: "src/**/*.ts,**/*.md"
---
```

1. **로드 시점**: 일반 레지스트리에 들어가지 않고 `conditionalSkills` 맵에 따로 보관
2. **파일 터치 시**: `discoverSkillDirsForPaths(touchedPaths)` 호출
   - 각 conditional skill의 `paths` glob과 비교
   - 매칭되면 `activatedConditionalSkillNames` 세트에 추가
3. **다음 쿼리**: `getDynamicSkills()`가 activated 목록을 반환
   - 시스템 프롬프트에 이 커맨드들의 `description` / `whenToUse`가 포함됨
   - 모델이 해당 커맨드를 선택·호출할 수 있게 됨

### 사용자 호출 vs 모델 호출

- 사용자가 `/`로 직접 호출하는 경우에도 conditional skill은 조회 가능 (`userInvocable`)
- 모델이 자동으로 선택하려면 조건부 활성화된 상태여야 함 (`disableModelInvocation`이 false인 경우)

## 치환 순서 요약

`getPromptForCommand(args, ctx)` 내부 처리 순서:

```
1. 원본 본문 로드
2. substituteArguments()          — $ARGUMENTS, $1, $argname
3. 환경 변수 치환                 — ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
4. executeShellCommandsInPrompt() — !`cmd` 실행 결과 삽입
5. (@mention은 상위 메시지 조립 단계에서 attachment로 변환)
6. ContentBlockParam[] 반환
```

## 관련 문서

- 이전: [05-execution.md](05-execution.md)
- 처음으로: [README.md](README.md)
