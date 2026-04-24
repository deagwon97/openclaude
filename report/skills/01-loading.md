# 01. Skill 로드 경로와 발견 메커니즘

## 1.1 Skill이 로드되는 디렉토리와 우선순위

Skill은 여러 경로에서 수집되며, 이름이 충돌하면 아래 순서로 우선권을 가진다 (높음 → 낮음).

| 우선순위 | 소스 | 경로 | 코드 |
|---|---|---|---|
| 1 | **Policy (관리 정책)** | `<MANAGED_FILE_PATH>/.claude/skills` | [loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts) |
| 2 | **User (사용자 홈)** | `~/.claude/skills` | 〃 |
| 3 | **Project** | `<CWD>/.claude/skills` — CWD 에서 `$HOME` 까지 재귀 탐색 | 〃 |
| 4 | **Additional** | `--add-dir` 로 지정된 경로의 `.claude/skills` | 〃 |
| 5 | **Legacy** | `.claude/commands/` (구식, 호환 유지) | `loadSkillsFromCommandsDir()` |
| 6 | **Bundled** | 코드에 내장된 built-in skill | [bundledSkills.ts](../../src/skills/bundledSkills.ts) |
| 7 | **Plugin** | 설치된 플러그인이 제공한 skill | `getPluginSkills()` |

진입점은 [loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts)의 `getSkillDirCommands(cwd)` 로, `memoize` 되어 있어 세션당 한 번만 실행된다.

```ts
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    const userSkillsDir    = join(getClaudeConfigHomeDir(), 'skills')
    const managedSkillsDir = join(getManagedFilePath(), '.claude', 'skills')
    const projectSkillsDirs = getProjectDirsUpToHome('skills', cwd)
    const additionalDirs    = getAdditionalDirectoriesForClaudeMd()
    // → loadSkillsFromSkillsDir(...) 를 각 경로에 대해 호출
  },
)
```

## 1.2 디렉토리 구조

Skill은 **중첩 디렉토리 + SKILL.md** 형태이다. 루트 `.md` 파일은 무시된다.

```
.claude/skills/
├── commit/
│   └── SKILL.md          ← 필수
├── review-pr/
│   ├── SKILL.md
│   └── helpers/
│       └── template.md   ← 선택적 참조 파일
└── category/
    └── another-skill/
        └── SKILL.md      ← 깊은 경로도 허용
```

- 파일명은 대소문자 구분 없이 `SKILL.md`, `Skill.md`, `skill.md` 모두 인식된다.
- 디렉토리 이름이 Skill 이름이 된다 (예: `commit/SKILL.md` → `/commit`).
- 심볼릭 링크는 따라가되, `realpath()` 로 순환 참조를 감지한다.

디렉토리 스캔은 `findSkillMarkdownFiles(basePath)` 가 수행한다.

## 1.3 로드 파이프라인

한 파일이 최종 `Command` 객체가 되기까지의 단계:

```
SKILL.md 발견
   ↓
readFile()
   ↓
parseFrontmatter(content)              ← src/utils/frontmatterParser.ts
   ↓
parseSkillFrontmatterFields(...)       ← src/skills/loadSkillsDir.ts
   ↓
createSkillCommand({...})              ← Command 객체 + getPromptForCommand() 클로저
   ↓
중복 제거 (realpath 기반)
   ↓
unconditional vs conditional 분기 (paths 필드 유무)
   ↓
최종 Command[] 반환
```

`createSkillCommand()` 는 `Command` 객체에 `getPromptForCommand(args, context)` 클로저를 심는다. 이 클로저는 호출 시점에 markdown 본문을 읽고:

- `$ARGUMENTS`, `$1`, `$2` 와 같은 인자 치환
- `${CLAUDE_SKILL_DIR}`, `${CLAUDE_SESSION_ID}` 치환
- Shell command 주입 (` !`cmd` ` 블록)
를 처리한 뒤 `[{ type: 'text', text: finalContent }]` 를 반환한다.

## 1.4 중복 제거

동일한 skill 파일이 여러 경로로 접근될 수 있으므로 (심볼릭 링크, project/user 양쪽에 존재 등) `realpath()` 기반 파일 아이덴티티 맵으로 중복을 제거한다. 소스별 우선순위는 §1.1 표를 따른다.

## 1.5 동적 Skill 발견 (Dynamic Skills)

정적 로드 외에, **파일 Read/Edit/Write 시점에** 해당 파일 근처의 `.claude/skills` 를 발견해 런타임에 추가하는 메커니즘이 있다.

- `discoverSkillDirsForPaths(filePaths, cwd)` — 파일 경로 → CWD 까지 상향식으로 탐색하여 새 `.claude/skills` 를 찾는다.
- `addSkillDirectories(dirs)` — 발견된 디렉토리에서 skill 을 로드해 `dynamicSkills` 맵에 병합한다. 이름 충돌 시 **더 깊은 경로가 우선**.
- `.gitignore` 에 포함된 경로는 제외된다 (보안).

이 덕분에 모노레포의 서브패키지에 있는 skill을, 해당 패키지의 파일을 만지는 순간 자동으로 끌어올 수 있다.

## 1.6 조건부 Skill 활성화 (Conditional Skills)

Frontmatter에 `paths` 필드가 있는 skill 은 로드 시 바로 노출되지 않고 `conditionalSkills` 맵에 보관된다. 이후 작업 중인 파일이 `paths` 의 glob 패턴과 매칭되면 활성화된다.

```yaml
---
description: TypeScript refactor helper
paths:
  - src/**/*.ts
  - lib/**/*.tsx
---
```

위 skill 은 TypeScript 파일을 만지기 전에는 모델에게 보이지 않는다.

## 1.7 캐싱

로드 결과는 아래 함수들이 `memoize` 로 감싸고 있다:

- `getSkillDirCommands(cwd)`
- `loadAllCommands(cwd)` — [commands.ts](../../src/commands.ts)
- `getSkillToolCommands(cwd)`
- `getSlashCommandToolSkills(cwd)`

캐시를 초기화할 때는 `clearCommandMemoizationCaches()` 를 호출한다 (설정 변경, skill 파일 수정 감지 등).
