# 04. 주입 위치 — System Prompt vs Attachment

`.claude/rules/*.md`의 내용은 두 군데로 들어간다. 어느 쪽에 들어가는지는 **정적이냐 조건부냐**, 그리고 **eager 경로냐 dynamic 경로냐**로 갈린다.

## A. System Prompt에 합쳐지기 — `claudeMd` 필드

### 진입점

[src/context.ts:155-189](../../src/context.ts#L155-L189)의 `getUserContext()`가 세션 시작 시 호출되어 다음을 만든다.

```ts
const claudeMd = shouldDisableClaudeMd
  ? null
  : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))
...
return {
  ...(claudeMd && { claudeMd }),
  currentDate: `Today's date is ${getLocalISODate()}.`,
}
```

`getMemoryFiles()`가 모은 모든 파일(정적 rules 포함)을 [getClaudeMds()](../../src/utils/claudemd.ts#L1162)가 하나의 큰 문자열로 직렬화한다.

### 직렬화 형식 ([claudemd.ts:1162-1204](../../src/utils/claudemd.ts#L1162-L1204))

```
Codebase and user instructions are shown below. Be sure to adhere to these
instructions. IMPORTANT: These instructions OVERRIDE any default behavior and
you MUST follow them exactly as written.

Contents of /path/to/file1 (project instructions, checked into the codebase):

<file 1 내용>

Contents of /path/to/file2 (user's private global instructions for all projects):

<file 2 내용>

...
```

각 파일마다 type별로 다른 설명 꼬리표가 붙는다:

| type | 꼬리표 |
|------|--------|
| Project | `(project instructions, checked into the codebase)` |
| Local | `(user's private project instructions, not checked in)` |
| User / Managed | `(user's private global instructions for all projects)` |
| AutoMem | `(user's auto-memory, persists across conversations)` |
| TeamMem | `(shared team memory, synced across the organization)` |

### 그 다음 어디로 가나

`claudeMd`는 `getUserContext()`가 반환하는 객체의 한 필드일 뿐이다. 이 객체는 query loop이 system prompt를 조립할 때 [src/utils/api.ts](../../src/utils/api.ts)의 `prependUserContext` 단계에서 사용된다 — 자세한 흐름은 [report/query/06_context_injection.md](../query/06_context_injection.md)를 참고.

### 비활성화

- `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` — 완전 비활성화.
- `--bare` + `--add-dir` 없음 — auto-discovery 끄기.
- `claudeMdExcludes` settings — 특정 파일 제외.

---

## B. `nested_memory` Attachment로 메시지에 끼워 넣기

조건부 규칙과 nested 디렉토리의 메모리는 system prompt가 아니라 **각 user 메시지에 첨부**된다.

### 변환

[memoryFilesToAttachments()](../../src/utils/attachments.ts#L1711)가 선택된 메모리 파일을 다음 형태로 변환한다.

```ts
{
  type: 'nested_memory',
  path: '/abs/path/to/.claude/rules/typescript.md',
  content: memoryFile,                  // MemoryFileInfo 객체 전체
  displayPath: 'src/.claude/rules/typescript.md',
}
```

이 attachment는 query loop이 매 turn마다 [getAttachmentMessages()](../../src/utils/attachments.ts#L2957)를 호출해 user 메시지 콘텐츠로 변환된다.

### 왜 attachment로 분리하나

- **관련 파일을 만질 때만** 모델 컨텍스트에 등장 → 토큰 절약.
- 동일 세션에서 같은 파일을 반복해서 주입하지 않도록 [loadedNestedMemoryPaths](../../src/utils/attachments.ts#L1723) Set으로 추적.
- `readFileState`에도 동시에 등록되어 ([attachments.ts:1743](../../src/utils/attachments.ts#L1743)) 후속 Edit/Write가 "이 파일은 이미 본 상태"로 인식 — 실수로 메모리 파일을 덮어쓰는 사고 방지.

### `contentDiffersFromDisk` 케이스

memory 파일은 모델에 보낼 때 여러 변형이 가해진다 (HTML 주석 제거, frontmatter 스트립, MEMORY.md 200줄 절단 등). 이런 경우 `contentDiffersFromDisk: true` 플래그가 붙고, `readFileState`에는 **원본 디스크 바이트**가 `isPartialView: true`와 함께 저장된다 ([attachments.ts:1743-1751](../../src/utils/attachments.ts#L1743-L1751)). Edit/Write tool은 이 플래그를 보고 "다시 Read부터 하라"고 사용자에게 요구한다.

---

## C. 두 주입 경로 종합

```
┌─────────────────────────────────────────────────────────────────┐
│  세션 시작                                                        │
│    getUserContext()                                              │
│      └─ getMemoryFiles()         [memoize, 1회]                  │
│           └─ 정적 rules 모음 → getClaudeMds() → 큰 문자열         │
│                                                                  │
│  query loop / 매 turn / 매 파일 터치                              │
│    getAttachmentMessages()                                       │
│      └─ getNestedMemoryAttachmentsForFile(filePath)              │
│           ├─ 조건부 rules: paths가 filePath에 매칭되는 것만        │
│           └─ nested 디렉토리: CWD → target 경로의 모든 메모리       │
└─────────────────────────────────────────────────────────────────┘
            │                                  │
            ▼                                  ▼
   ┌─────────────────┐               ┌──────────────────────┐
   │  System Prompt  │               │  User Message        │
   │  claudeMd 블록  │               │  nested_memory       │
   │                 │               │  attachment          │
   │  - Managed      │               │  - 조건부 매칭본       │
   │  - User         │               │  - nested CLAUDE.md  │
   │  - Project 정적 │               │  - nested 정적 rules │
   │  - Local        │               │                      │
   └─────────────────┘               └──────────────────────┘
```
