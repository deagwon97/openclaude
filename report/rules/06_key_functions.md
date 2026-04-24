# 06. 핵심 함수 / 파일 / 라인 레퍼런스

## 메인 진입점 (위→아래 = 호출 방향)

| 함수 | 위치 | 역할 |
|------|------|------|
| `getUserContext()` | [src/context.ts:155](../../src/context.ts#L155) | system prompt에 들어갈 `claudeMd` 문자열 만들기 — 세션 1회 |
| `getMemoryFiles()` | [src/utils/claudemd.ts:799](../../src/utils/claudemd.ts#L799) | 모든 스코프의 메모리 + 정적 rules eager 로드 (memoize) |
| `getClaudeMds()` | [src/utils/claudemd.ts:1162](../../src/utils/claudemd.ts#L1162) | `MemoryFileInfo[]`을 system prompt용 큰 문자열로 직렬화 |
| `getNestedMemoryAttachmentsForFile()` | [src/utils/attachments.ts:1793](../../src/utils/attachments.ts#L1793) | 특정 파일에 매칭되는 메모리를 attachment로 변환 — 매 turn |
| `getAttachmentMessages()` | [src/utils/attachments.ts:2957](../../src/utils/attachments.ts#L2957) | query loop이 매 turn 호출하는 attachment 변환 진입점 |
| `memoryFilesToAttachments()` | [src/utils/attachments.ts:1711](../../src/utils/attachments.ts#L1711) | `MemoryFileInfo` → `nested_memory` attachment, dedup 처리 |

## 디렉토리 스캔 / 파일 파싱

| 함수 | 위치 | 역할 |
|------|------|------|
| `processMdRules()` | [src/utils/claudemd.ts:706](../../src/utils/claudemd.ts#L706) | `.claude/rules/`를 재귀 스캔, conditionalRule 플래그로 정적/조건부 분리 |
| `processMemoryFile()` | [src/utils/claudemd.ts:627](../../src/utils/claudemd.ts#L627) | 단일 파일 읽기 + `@include` 재귀 처리 |
| `parseFrontmatterPaths()` | [src/utils/claudemd.ts:254](../../src/utils/claudemd.ts#L254) | frontmatter에서 `paths` 추출, `/**` 정리, match-all 감지 |
| `parseFrontmatter()` | [src/utils/frontmatterParser.ts](../../src/utils/frontmatterParser.ts) | `---` 구분 frontmatter 블록을 YAML로 파싱 |
| `splitPathInFrontmatter()` | [src/utils/frontmatterParser.ts](../../src/utils/frontmatterParser.ts) | `paths` 값을 문자열/배열 양쪽에서 받아 패턴 리스트로 |
| `stripHtmlComments()` | [src/utils/claudemd.ts:292](../../src/utils/claudemd.ts#L292) | 블록 레벨 `<!-- ... -->` 제거 (코드 블록은 보존) |

## 조건부 매칭

| 함수 | 위치 | 역할 |
|------|------|------|
| `processConditionedMdRules()` | [src/utils/claudemd.ts:1363](../../src/utils/claudemd.ts#L1363) | `paths` glob을 targetPath에 매칭, scope별 baseDir 사용 |
| `getManagedAndUserConditionalRules()` | [src/utils/claudemd.ts:1214](../../src/utils/claudemd.ts#L1214) | dynamic Phase 1 — Managed/User 조건부 |
| `getMemoryFilesForNestedDirectory()` | [src/utils/claudemd.ts:1258](../../src/utils/claudemd.ts#L1258) | dynamic Phase 3 — nested 디렉토리의 CLAUDE.md + 정적/조건부 |
| `getConditionalRulesForCwdLevelDirectory()` | [src/utils/claudemd.ts:1338](../../src/utils/claudemd.ts#L1338) | dynamic Phase 4 — CWD 위 디렉토리의 조건부만 |

## 경로 헬퍼

| 함수 | 위치 | 역할 |
|------|------|------|
| `getManagedClaudeRulesDir()` | [src/utils/config.ts](../../src/utils/config.ts) | Managed 정책 rules 디렉토리 (예: `/etc/claude-code/rules/`) |
| `getUserClaudeRulesDir()` | [src/utils/config.ts](../../src/utils/config.ts) | User rules 디렉토리 (`~/.claude/rules/`) |
| `getMemoryPath('Managed' \| 'User')` | [src/utils/config.ts](../../src/utils/config.ts) | 단일 CLAUDE.md 파일 경로 |
| `isMemoryFilePath()` | [src/utils/claudemd.ts:1444](../../src/utils/claudemd.ts#L1444) | 임의 경로가 메모리 파일인지 판정 (CLAUDE.md, CLAUDE.local.md, `.claude/rules/*.md`) |

## 캐싱 / 무효화

| 함수 | 위치 | 역할 |
|------|------|------|
| `getMemoryFiles` (lodash `memoize`) | [src/utils/claudemd.ts:799](../../src/utils/claudemd.ts#L799) | 세션당 1회만 디스크 스캔 |
| `resetGetMemoryFilesCache()` | [src/utils/claudemd.ts:1120](../../src/utils/claudemd.ts#L1120) | `/compact` 시점 등에서 캐시 비우고 다음 호출에 다시 hook 발사 |
| `loadedNestedMemoryPaths` (Set) | `toolUseContext` 필드 | dynamic 경로의 세션 전체 dedup |
| `readFileState` (LRU 100) | `toolUseContext` 필드 | Edit/Write 안전 검사용 파일 상태 캐시 |

## 옵저버빌리티 / Hook

| 함수 | 위치 | 역할 |
|------|------|------|
| `executeInstructionsLoadedHooks()` | [src/utils/hooks.ts](../../src/utils/hooks.ts) | 메모리/rules 파일 로드 시 `InstructionsLoaded` hook 발사 |
| `hasInstructionsLoadedHook()` | [src/utils/hooks.ts](../../src/utils/hooks.ts) | hook 등록 여부 — 미등록 시 dispatch 스킵 |

`InstructionsLoadReason` enum:
- `'session_start'` — eager 경로 첫 로드.
- `'compact'` — 컨텍스트 압축 후 재로드.
- `'path_glob_match'` — 조건부 규칙이 paths에 매칭됨.
- `'nested_traversal'` — nested 디렉토리 walk에서 발견.
- `'include'` — 다른 메모리 파일이 `@`로 포함.

## 상수

| 상수 | 값 | 위치 |
|------|----|------|
| `MEMORY_INSTRUCTION_PROMPT` | `"Codebase and user instructions are shown below..."` 시작 문장 | [claudemd.ts:89](../../src/utils/claudemd.ts#L89) |
| `MAX_MEMORY_CHARACTER_COUNT` | `40000` | [claudemd.ts:92](../../src/utils/claudemd.ts#L92) |
| `TEXT_FILE_EXTENSIONS` | `.md`, `.txt`, `.json`, `.yaml`, ... 약 180개 | [claudemd.ts:96](../../src/utils/claudemd.ts#L96) |

## 비활성화 스위치

| 스위치 | 위치 |
|--------|------|
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env | [context.ts:165-167](../../src/context.ts#L165-L167) |
| `--bare` 플래그 | `isBareMode()` |
| `claudeMdExcludes` settings | [src/utils/settings/types.ts](../../src/utils/settings/types.ts) |
| `isSettingSourceEnabled('userSettings' \| 'projectSettings' \| 'localSettings')` | scope별 활성화 게이트 |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` env | `--add-dir` 디렉토리에서도 메모리 읽을지 |
