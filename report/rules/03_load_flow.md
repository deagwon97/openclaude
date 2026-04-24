# 03. 로딩 흐름 — Eager vs Dynamic

`.claude/rules/*.md`는 두 가지 경로로 로드된다.

- **Eager 경로** — 세션 시작 시 모든 정적 규칙을 한 번에 모아 system prompt에 합친다.
- **Dynamic 경로** — 사용자가 어떤 파일을 읽거나 편집할 때마다 그 파일 경로에 매칭되는 조건부 규칙을 골라 attachment로 끼워 넣는다.

각각의 진입점은 [getMemoryFiles()](../../src/utils/claudemd.ts#L799)와 [getNestedMemoryAttachmentsForFile()](../../src/utils/attachments.ts#L1793)다.

---

## A. Eager 경로 — `getMemoryFiles()`

세션이 시작되면 [src/context.ts:170-172](../../src/context.ts#L170-L172)의 `getUserContext()`가 한 번 호출되고, 그 안에서 `getMemoryFiles()`가 호출된다. 결과는 `memoize()`로 캐싱되어 동일 프로세스 내에서 중복 호출되지 않는다.

### 처리 순서 ([claudemd.ts:799-1083](../../src/utils/claudemd.ts#L799-L1083))

```
getMemoryFiles(forceIncludeExternal=false)
│
├─ 1. Managed
│   ├─ processMemoryFile(/etc/.../CLAUDE.md, 'Managed')
│   └─ processMdRules({                                ← 정적 규칙만
│        rulesDir: getManagedClaudeRulesDir(),
│        type: 'Managed',
│        conditionalRule: false,
│      })
│
├─ 2. User  (userSettings 활성화 시)
│   ├─ processMemoryFile(~/.claude/CLAUDE.md, 'User')
│   └─ processMdRules({
│        rulesDir: getUserClaudeRulesDir(),  // ~/.claude/rules
│        type: 'User',
│        conditionalRule: false,
│      })
│
├─ 3. Project + Local  (root → CWD 순으로 디렉토리 walk)
│   for dir in [root, ..., parent, cwd]:
│     ├─ CLAUDE.md
│     ├─ .claude/CLAUDE.md
│     ├─ .claude/rules/*.md (정적만)            ← processMdRules conditionalRule=false
│     └─ CLAUDE.local.md
│
├─ 4. --add-dir 추가 디렉토리  (env로 게이팅)
│   각 추가 디렉토리에 대해 위 3과 동일 절차
│
├─ 5. AutoMem (memdir) 엔트리포인트  (선택적)
├─ 6. TeamMem 엔트리포인트  (TEAMMEM 기능 플래그가 켜진 경우)
│
└─ InstructionsLoaded hook 발사 (각 파일에 대해 1회)
```

### 핵심 호출 — Project rules

[claudemd.ts:918-928](../../src/utils/claudemd.ts#L918-L928):

```ts
// Try reading .claude/rules/*.md files (Project)
const rulesDir = join(dir, '.claude', 'rules')
result.push(
  ...(await processMdRules({
    rulesDir,
    type: 'Project',
    processedPaths,
    includeExternal,
    conditionalRule: false,   // ← eager 경로는 정적 규칙만 모음
  })),
)
```

### `processMdRules()`의 동작 ([claudemd.ts:706-797](../../src/utils/claudemd.ts#L706-L797))

1. `rulesDir`을 readdir.
2. 각 엔트리에 대해:
   - 디렉토리 → 재귀 호출 (서브디렉토리 지원).
   - `.md` 파일 → `processMemoryFile()`로 파싱.
3. 결과 중 `conditionalRule` 플래그에 따라 필터링:
   - `conditionalRule: false` → `f.globs`가 없는 파일만 (정적).
   - `conditionalRule: true` → `f.globs`가 있는 파일만 (조건부).
4. 심볼릭 링크 사이클은 `visitedDirs` Set으로 차단.
5. ENOENT/EACCES/ENOTDIR는 조용히 빈 배열 반환 — `.claude/rules/`가 없는 프로젝트도 정상.

### 캐싱

`getMemoryFiles`는 `memoize()`로 감싸여 있다 ([claudemd.ts:799](../../src/utils/claudemd.ts#L799)). 캐시는 [resetGetMemoryFilesCache()](../../src/utils/claudemd.ts#L1120-L1130)로만 무효화되며, 주로 `/compact` 같은 컨텍스트 압축 시점에 호출된다. **수동으로 rules 파일을 수정해도 자동 리로드되지 않는다** — 세션 재시작이 필요하다.

---

## B. Dynamic 경로 — `getNestedMemoryAttachmentsForFile()`

사용자가 어떤 파일을 만지는 매 순간(파일 mention, IDE 선택, Read/Edit/Write tool 사용 등) 그 파일 경로 기준으로 추가 메모리를 모은다. 진입점은 [src/utils/attachments.ts:1793](../../src/utils/attachments.ts#L1793).

### 호출 시점

[attachments.ts](../../src/utils/attachments.ts) 안에서 다음 상황마다 호출된다:
- IDE에서 열린 파일 (`getOpenedFileFromIDE`, [attachments.ts:1879](../../src/utils/attachments.ts#L1879)).
- `@`로 멘션된 파일 (`processAtMentionedFiles` 경로, [attachments.ts:2184](../../src/utils/attachments.ts#L2184)).
- `getAttachmentMessages()` ([attachments.ts:2957](../../src/utils/attachments.ts#L2957))는 query loop이 매 turn 호출하는 진입점.

### 4단계 처리 ([attachments.ts:1793-1863](../../src/utils/attachments.ts#L1793-L1863))

```
getNestedMemoryAttachmentsForFile(filePath, ctx, appState)
│
├─ Phase 1. Managed/User 조건부 규칙 매칭
│   getManagedAndUserConditionalRules(filePath, processedPaths)
│     ├─ processConditionedMdRules(/etc/.../rules, 'Managed', filePath)
│     └─ processConditionedMdRules(~/.claude/rules,  'User',   filePath)
│
├─ Phase 2. 처리할 디렉토리 묶음 결정
│   getDirectoriesToProcess(filePath, originalCwd)
│     → { nestedDirs, cwdLevelDirs }
│       nestedDirs   : CWD 아래로 target 파일까지 내려가는 경로
│       cwdLevelDirs : CWD 위로 root까지 올라가는 경로
│
├─ Phase 3. nestedDirs (CWD → target)
│   각 디렉토리마다 getMemoryFilesForNestedDirectory()
│     ├─ CLAUDE.md / .claude/CLAUDE.md
│     ├─ CLAUDE.local.md
│     ├─ .claude/rules/*.md (정적, 아직 안 본 것)
│     └─ .claude/rules/*.md (조건부, paths 매칭)
│
└─ Phase 4. cwdLevelDirs (root → CWD)
    각 디렉토리마다 getConditionalRulesForCwdLevelDirectory()
      └─ .claude/rules/*.md (조건부만, paths 매칭)
```

> **왜 phase 3과 4가 다른가?** Phase 3의 nested 디렉토리는 CWD 아래라서 eager 패스에서 한 번도 본 적이 없는 새 영역 — 그래서 정적 규칙도 함께 끌어온다. Phase 4의 CWD 위 디렉토리는 eager 패스에서 정적 규칙을 이미 봤으므로, 조건부 규칙만 새로 매칭하면 된다.

### 중복 차단

- `toolUseContext.loadedNestedMemoryPaths` (세션 전체 Set) — 한 번 주입한 메모리 파일은 다시 주입하지 않음. ([attachments.ts:1723](../../src/utils/attachments.ts#L1723))
- `toolUseContext.readFileState` (100-entry LRU) — Read/Edit 시점의 파일 상태 캐시. LRU에서 빠지면 재주입될 수 있어, `loadedNestedMemoryPaths`가 보조 가드 역할.

### 결과

선택된 파일들은 [memoryFilesToAttachments()](../../src/utils/attachments.ts#L1711)에서 다음 형태의 attachment로 변환되어 다음 user 메시지에 첨부된다.

```ts
{
  type: 'nested_memory',
  path: memoryFile.path,
  content: memoryFile,
  displayPath: relative(getCwd(), memoryFile.path),
}
```

---

## 두 경로의 관계 한눈에

| | Eager | Dynamic |
|--|-------|---------|
| 진입점 | `getMemoryFiles()` | `getNestedMemoryAttachmentsForFile()` |
| 호출 시점 | 세션 시작 시 1회 (memoize) | 매 turn / 매 파일 터치 |
| 대상 | **정적 규칙만** | **조건부 규칙 + nested 디렉토리의 모든 메모리** |
| 주입 위치 | system prompt (`claudeMd` 필드) | user 메시지에 `nested_memory` attachment |
| 캐싱 | `memoize` 1회 | `loadedNestedMemoryPaths` Set 으로 dedup |
