# 04 · 등록 (Command Object Creation & Registry)

파싱된 frontmatter + 본문을 기반으로 **커맨드 객체**를 생성하고 전역 레지스트리에 등록하는 단계입니다.

## 커맨드 객체 생성

**파일**: [src/skills/loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts) · `createSkillCommand()`

```ts
{
  type: 'prompt',                    // 커스텀 .md 커맨드의 기본 타입
  name: skillName,                   // 파일명 (확장자 제외)
  description,
  hasUserSpecifiedDescription,
  allowedTools,
  argNames,                          // 명명된 인수
  argumentHint,
  whenToUse,
  version,
  model,
  disableModelInvocation,
  userInvocable,
  context,                           // 'fork' | undefined
  agent,
  effort,
  paths,                             // 조건부 활성화
  contentLength,
  isHidden,
  progressMessage,
  source: 'projectSettings' | 'userSettings' | 'policySettings',
  loadedFrom: 'commands_DEPRECATED' | 'skills',
  hooks,
  skillRoot,

  // 핵심: 실행 시 프롬프트 본문을 생성하는 지연 함수
  async getPromptForCommand(args, context): Promise<ContentBlockParam[]> { ... }
}
```

### `getPromptForCommand`의 지연 평가

가장 중요한 점은 **본문 치환이 등록 시점이 아니라 호출 시점에 일어난다**는 것입니다. 이 함수는 호출될 때마다:

1. 원본 본문을 가져옴
2. `substituteArguments()`로 `$ARGUMENTS`, `$1`, `$argname` 치환
3. 환경 변수 치환: `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}`
4. `executeShellCommandsInPrompt()`로 `!`cmd`` 블록 실행 및 출력 삽입
5. 최종 `ContentBlockParam[]`을 반환

자세한 치환 규칙은 [06-special-syntax.md](06-special-syntax.md) 참조.

## 레지스트리 등록

**파일**: [src/commands.ts](../../src/commands.ts)

```ts
loadAllCommands = memoize(async (cwd) => {
  const builtins = [...]              // 하드코딩된 빌트인 커맨드 import
  const skillDirCmds = await getSkillDirCommands(cwd)   // .claude/commands/
  const skillToolCmds = await getSkillToolCommands(cwd) // SkillTool용 스킬
  const mcpCmds = ...                  // MCP 제공 커맨드
  const pluginCmds = ...               // 플러그인 커맨드

  return dedupeByName([
    ...builtins,
    ...skillDirCmds,
    ...skillToolCmds,
    ...mcpCmds,
    ...pluginCmds,
  ])
})
```

- `cwd`별로 memoize → 같은 디렉토리에서 호출하면 캐시 사용
- 이름 충돌 시 로드 순서에 따라 선-로드가 우선 (빌트인 > 커스텀)

## 빌트인 vs 커스텀 커맨드 비교

| 항목 | 빌트인 | 커스텀 (`.claude/commands/*.md`) |
|------|--------|----------------------------------|
| 로드 방식 | 하드코딩 `import` | 디스크 스캔 (`loadSkillsDir`) |
| 타입 | `'local'` \| `'local-jsx'` \| `'prompt'` | 주로 `'prompt'` |
| source | `'builtin'` | `'projectSettings'` \| `'userSettings'` \| `'policySettings'` |
| 실행 | JS 함수 직접 호출 | 본문을 Claude에게 메시지로 전달 |
| 즉시 실행 | `immediate` 플래그로 API 호출 없이 처리 가능 | 거의 항상 Claude 호출 필요 |
| 라이프사이클 | 앱 시작 시 1회 | `cwd`별 memoize, 설정 변경 시 무효화 |

## 캐시 무효화

```ts
// src/commands.ts
clearCommandMemoizationCaches()   // memoize 캐시만 초기화
clearCommandsCache()              // 플러그인까지 포함한 전체 초기화
```

무효화가 필요한 경우:
- `.claude/commands/` 파일 수정/추가
- 플러그인 로드/언로드
- 설정(profile) 변경

## 조건부 활성화 (`paths:` frontmatter)

`paths:` 필드가 있는 커맨드는 **정상 등록되지 않고** `conditionalSkills` 맵에 따로 보관됩니다.

```ts
// src/skills/loadSkillsDir.ts
conditionalSkills.set(name, command)
```

- 시작 시: 일반 레지스트리에는 포함 안 됨
- 런타임: `discoverSkillDirsForPaths(touchedPaths)`가 호출될 때마다 glob 매칭
- 매칭 성공 → `activatedConditionalSkillNames` 세트에 추가
- 다음 쿼리의 `getDynamicSkills()` 호출에서 반환되어 모델이 볼 수 있게 됨

자세한 내용은 [06-special-syntax.md](06-special-syntax.md) 참조.

## 관련 문서

- 이전: [03-parsing.md](03-parsing.md)
- 다음: [05-execution.md](05-execution.md) — 사용자 입력부터 Claude 메시지까지
