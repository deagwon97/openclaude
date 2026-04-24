# 02 · 발견 (Discovery)

커스텀 커맨드 파일을 디스크에서 찾아 읽어들이는 단계입니다.

## 검색 대상 경로

openclaude는 **세 계층**에서 `.claude/commands/` 디렉토리를 찾습니다. 같은 이름이 충돌하면 **policy > user > project** 순으로 우선합니다.

| 계층 | 경로 | source 값 |
|------|------|-----------|
| Project | `<cwd>/.claude/commands/` (git root까지 상향 탐색) | `projectSettings` |
| User | `~/.claude/commands/` | `userSettings` |
| Policy | `<managed>/.claude/commands/` | `policySettings` |

## 핵심 파일

### [src/utils/markdownConfigLoader.ts](../../src/utils/markdownConfigLoader.ts)

공용 로더 — `commands/`, `skills/`, `agents/` 모두 이 파일이 처리합니다.

- **`getProjectDirsUpToHome(subdir, cwd)`**
  - `cwd`에서 git root(또는 `$HOME`)까지 올라가며 각 레벨의 `.claude/<subdir>/`를 수집
  - git root를 경계로 삼아 프로젝트 외부 누출 방지
  - 심링크 inode 기반 중복 제거

- **`loadMarkdownFilesForSubdir(subdir, cwd)`**
  - 세 계층(managed/user/project)을 **병렬 스캔**
  - 각 `.md` 파일을 읽어 `{filePath, baseDir, source, frontmatter, content}` 메타데이터로 반환
  - 우선순위: managed → user → project (뒤에 나온 게 앞을 덮지 않음)

### [src/skills/loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts)

커맨드/스킬 전용 상위 로더.

- **`getSkillDirCommands(cwd)`** — `memoize`된 최종 진입점. `loadMarkdownFilesForSubdir('commands', cwd)`를 호출하고 각 파일을 커맨드 객체로 변환.
- 결과는 [src/commands.ts](../../src/commands.ts)의 `loadAllCommands()`에서 빌트인 커맨드와 합쳐집니다.

## 검색 범위의 경계

- **git root까지만**: 부모 디렉토리를 무한 탐색하지 않고 git 경계에서 멈춤 → 레포 외부의 낯선 `.claude/commands/`가 실수로 로드되는 것 방지
- **심링크 중복 제거**: 같은 파일을 심링크로 두 번 로드하지 않음 (inode 비교)
- **확장자**: `.md` 파일만 수집

## 캐싱

```ts
// src/commands.ts
loadAllCommands = memoize(async (cwd) => { ... })
getSkillDirCommands = memoize(async (cwd) => { ... })
```

- `cwd`를 키로 memoize → 같은 디렉토리에서 재스캔 없음
- 플러그인 로드나 설정 변경 시 `clearCommandsCache()` / `clearCommandMemoizationCaches()`로 무효화

## 왜 `skills` 로더가 `commands`도 처리하나?

내부적으로 `commands/`는 `skills/`의 **레거시 이름**으로 취급됩니다. 커맨드 객체에는 `loadedFrom: 'commands_DEPRECATED'` 플래그가 붙어 추후 마이그레이션 가능성을 남겨둡니다. 파싱·등록 로직은 동일하므로 한 벌의 코드로 둘 다 처리합니다.

## 관련 문서

- 이전: [01-overview.md](01-overview.md)
- 다음: [03-parsing.md](03-parsing.md) — 읽어들인 파일을 어떻게 파싱하는지
