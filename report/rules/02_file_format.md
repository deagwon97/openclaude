# 02. 파일 형식과 Frontmatter

## 기본 형식

`.claude/rules/*.md` 파일은 표준 마크다운이며, 선택적으로 YAML frontmatter 블록을 가질 수 있다.

```markdown
---
paths: "src/**/*.ts, src/**/*.tsx"
description: "TypeScript 작업 시 적용할 규칙"
---

# TypeScript 스타일 가이드

- 모든 export 함수에 명시적 반환 타입 지정
- ...
```

frontmatter가 없으면 그냥 정적 규칙으로 취급된다.

## Frontmatter 스키마

frontmatter 자체는 [src/utils/frontmatterParser.ts](../../src/utils/frontmatterParser.ts)의 `FrontmatterData` 타입에 정의되어 있다. rules와 직접 관계 있는 필드는 `paths` 하나뿐이다 ([frontmatterParser.ts:48-52](../../src/utils/frontmatterParser.ts#L48-L52)).

```ts
// Glob patterns for file paths this skill applies to. Accepts either a
// comma-separated string or a YAML list of strings.
paths?: string | string[] | null
```

> 같은 `paths` 필드는 skills와 commands에서도 동일한 의미로 쓰인다 — 코드 한 곳에 모든 종류의 frontmatter 필드가 모여 있다.

## `paths` 파싱 규칙

[parseFrontmatterPaths()](../../src/utils/claudemd.ts#L254)가 다음 단계로 처리한다.

1. `paths`가 없으면 → 정적 규칙으로 취급 (`paths: undefined` 반환).
2. 문자열이면 쉼표로, 배열이면 그대로 패턴 리스트로 분리 (`splitPathInFrontmatter`).
3. 각 패턴 끝의 `/**` 접미사 제거 — `ignore` 라이브러리는 디렉토리 매칭 시 자동으로 하위를 포함하므로 중복 제거용.
4. 모든 패턴이 비었거나 전부 `**`이면 → 정적 규칙으로 취급.
5. 그 외에는 `paths`가 들어간 **조건부 규칙**으로 분류.

```ts
// claudemd.ts:265-278
const patterns = splitPathInFrontmatter(frontmatter.paths)
  .map(p => p.endsWith('/**') ? p.slice(0, -3) : p)
  .filter(p => p.length > 0)

if (patterns.length === 0 || patterns.every(p => p === '**')) {
  return { content }   // 정적
}
return { content, paths: patterns }  // 조건부
```

## 패턴 형식

매칭은 [`ignore`](https://www.npmjs.com/package/ignore) 라이브러리로 수행된다 ([claudemd.ts:1404](../../src/utils/claudemd.ts#L1404)). 즉 **`.gitignore`와 동일한 문법**이다.

```yaml
# 단일 패턴 (문자열)
paths: "src/**/*.ts"

# 쉼표 구분 다중 패턴
paths: "src/**/*.ts, src/**/*.tsx, !src/**/*.test.ts"

# YAML 리스트
paths:
  - "src/api/**"
  - "tests/**"
```

자주 쓰는 형태:
- `src/` — `src` 디렉토리 아래 모든 파일
- `**/*.md` — 트리 전체에서 .md 파일
- `path/to/file.{ts,tsx}` — 중괄호 확장
- `!something` — 부정 패턴 (제외)

## `@include` 지시문

rules 파일은 다른 파일의 내용을 끌어올 수 있다 ([claudemd.ts:18-26](../../src/utils/claudemd.ts#L18-L26)).

```markdown
# 메인 규칙

@./shared-rules.md
@~/global-style.md
@/absolute/path/rules.md
```

규칙:
- 텍스트 노드에서만 인식 — 코드 블록/인라인 코드 안에서는 무시.
- `@path` (접두사 없음) = `@./path` (상대 경로).
- 포함된 파일은 **포함하는 파일보다 먼저 별도 엔트리**로 추가된다.
- 순환 참조는 `processedPaths` Set으로 차단.
- 존재하지 않는 파일은 조용히 무시.
- 허용 확장자: `.md`, `.txt`, `.json`, `.yaml`, `.toml`, `.xml`, `.sql` 등 텍스트 계열만 ([TEXT_FILE_EXTENSIONS](../../src/utils/claudemd.ts#L96)에 약 180개).

## HTML 주석 제거

[stripHtmlComments()](../../src/utils/claudemd.ts#L292)가 블록 레벨 `<!-- ... -->` 주석을 제거한다. 코드 블록 안의 주석과 인라인 주석은 보존된다. 저자용 메모를 모델에 보내지 않기 위한 장치.

## 권장 크기

`MAX_MEMORY_CHARACTER_COUNT = 40000` ([claudemd.ts:92](../../src/utils/claudemd.ts#L92)). 권장 한계이며 강제 차단은 아니다.

## `claudeMdExcludes` 설정으로 제외하기

settings.json에서 특정 rules 파일을 로드 대상에서 빼낼 수 있다.

```json
{
  "claudeMdExcludes": [
    "**/some-dir/.claude/rules/**",
    "/abs/path/CLAUDE.md"
  ]
}
```

User/Project/Local 메모리에만 적용된다 — Managed 정책 파일은 항상 로드된다 ([settings/types.ts](../../src/utils/settings/types.ts) 내 `claudeMdExcludes` 정의 참고).
