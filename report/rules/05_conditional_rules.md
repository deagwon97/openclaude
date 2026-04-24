# 05. 조건부 규칙의 Glob 매칭

조건부 규칙(`paths` frontmatter가 있는 파일)이 어떤 타깃 파일에 적용되는지 결정하는 로직은 [processConditionedMdRules()](../../src/utils/claudemd.ts#L1363)에 모여 있다.

## 함수 시그니처

```ts
// claudemd.ts:1363
export async function processConditionedMdRules(
  targetPath: string,        // 매칭 대상 (예: 사용자가 편집하는 src/api/foo.ts)
  rulesDir: string,          // 스캔할 .claude/rules 디렉토리
  type: MemoryType,          // 'Managed' | 'User' | 'Project'
  processedPaths: Set<string>,
  includeExternal: boolean,
): Promise<MemoryFileInfo[]>
```

## 알고리즘 ([claudemd.ts:1369-1405](../../src/utils/claudemd.ts#L1369-L1405))

```ts
// 1. rulesDir 안에서 paths frontmatter가 있는 파일만 모은다
const conditionedRuleMdFiles = await processMdRules({
  rulesDir,
  type,
  processedPaths,
  includeExternal,
  conditionalRule: true,    // ← f.globs가 있는 파일만 통과
})

// 2. 각 파일의 globs 패턴이 targetPath에 매칭되는지 검사
return conditionedRuleMdFiles.filter(file => {
  if (!file.globs || file.globs.length === 0) return false

  // baseDir 결정: scope에 따라 다름
  const baseDir =
    type === 'Project'
      ? dirname(dirname(rulesDir))   // .claude의 부모 디렉토리
      : getOriginalCwd()             // Managed/User는 원래 CWD

  const relativePath = isAbsolute(targetPath)
    ? relative(baseDir, targetPath)
    : targetPath

  // 안전 가드: 빈 문자열, 상위 탈출, 절대 경로 모두 거부
  if (!relativePath ||
      relativePath.startsWith('..') ||
      isAbsolute(relativePath)) {
    return false
  }

  // .gitignore 문법으로 매칭
  return ignore().add(file.globs).ignores(relativePath)
})
```

## 핵심 디테일 — `baseDir`이 왜 scope마다 다른가

`paths` 패턴은 **baseDir 기준 상대 경로**로 해석된다. baseDir이 어디냐에 따라 같은 패턴의 의미가 달라진다.

### Project 규칙
`baseDir = .claude의 부모 디렉토리`

```
my-repo/
├── .claude/
│   └── rules/
│       └── api.md          paths: "src/api/**"
└── src/
    └── api/
        └── handler.ts       ← 매칭됨 (relative: "src/api/handler.ts")
```

서브디렉토리에 `.claude/rules`가 또 있으면, **그 서브디렉토리의 부모**가 baseDir이 된다 — 즉 nested된 rules는 그 서브트리 안에서만 효과적이다.

```
my-repo/
└── packages/
    └── frontend/
        ├── .claude/
        │   └── rules/
        │       └── react.md     paths: "src/**/*.tsx"
        └── src/
            └── App.tsx          ← baseDir = packages/frontend
                                   relative: "src/App.tsx" → 매칭
```

이 baseDir 결정 코드 한 줄이 핵심:
```ts
type === 'Project' ? dirname(dirname(rulesDir)) : getOriginalCwd()
```

### Managed / User 규칙
`baseDir = getOriginalCwd()` — 사용자가 `openclaude`를 실행한 디렉토리.

`~/.claude/rules/personal.md`에서 `paths: "src/**/*.ts"`라고 쓰면, **CWD 기준으로** `src/**/*.ts`에 매칭된다. 여러 프로젝트를 오가도 일관된 의미를 가진다.

## 안전 가드의 이유

```ts
if (!relativePath || relativePath.startsWith('..') || isAbsolute(relativePath)) {
  return false
}
```

세 가지를 막는다:
1. **빈 문자열** — `ignore` 라이브러리가 throw.
2. **`..`로 시작** — baseDir 밖의 파일. 어차피 baseDir 기준 패턴과 매칭될 일이 없다.
3. **절대 경로** — Windows에서 cross-drive `relative()`가 절대 경로를 반환할 수 있다. `ignore`가 throw하고 매칭도 의미 없다.

## 매칭 라이브러리

```ts
return ignore().add(file.globs).ignores(relativePath)
```

[`ignore`](https://www.npmjs.com/package/ignore) 패키지는 `.gitignore` 문법을 그대로 구현한다. 따라서:
- `**/*.ts` — 모든 .ts 파일
- `src/` — `src` 디렉토리 통째로
- `!exclude.ts` — 부정 패턴
- `path/{a,b}.ts` — 중괄호 확장

`ignores(path)`는 패턴 매칭 결과를 boolean으로 반환한다. 이름이 `ignores`라 헷갈리지만, 여기서는 "매칭된다 = true"의 의미로 쓴다.

## 누가 호출하나

dynamic 경로의 3개 함수가 결국 모두 `processConditionedMdRules()`에 도달한다.

| 호출자 | 위치 | 용도 |
|--------|------|------|
| [getManagedAndUserConditionalRules()](../../src/utils/claudemd.ts#L1214) | claudemd.ts:1214 | Phase 1 — Managed/User 조건부 매칭 |
| [getMemoryFilesForNestedDirectory()](../../src/utils/claudemd.ts#L1258) | claudemd.ts:1258 | Phase 3 — nested 디렉토리의 조건부 규칙 |
| [getConditionalRulesForCwdLevelDirectory()](../../src/utils/claudemd.ts#L1338) | claudemd.ts:1338 | Phase 4 — CWD 위 디렉토리의 조건부 규칙 |

이 셋이 [getNestedMemoryAttachmentsForFile()](../../src/utils/attachments.ts#L1793)의 Phase 1/3/4를 구성한다 ([report/rules/03_load_flow.md](03_load_flow.md) §B 참고).
