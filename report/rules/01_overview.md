# 01. `.claude/rules` 개요

## 무엇인가

`.claude/rules/`는 **사용자가 작성한 명령문(instructions) 마크다운 파일들을 모아두는 디렉토리**다. CLAUDE.md와 같은 "프로젝트 명령문" 카테고리에 속하지만, 다음 두 가지 차별점이 있다.

1. **여러 파일로 분할**할 수 있다 — 주제별로 `typescript.md`, `testing.md`, `api/endpoints.md` 같은 식.
2. **frontmatter `paths`로 적용 범위를 좁힐 수 있다** — 특정 glob 패턴에 매칭되는 파일을 다룰 때만 모델에 주입된다.

코드 상의 권위 있는 정의는 [src/utils/claudemd.ts:1-26](../../src/utils/claudemd.ts#L1-L26)의 파일 헤더 주석에 있다.

## 어디에 둘 수 있나

[getMemoryFiles()](../../src/utils/claudemd.ts#L799)가 다음 4개 스코프의 `.claude/rules/`를 차례로 스캔한다.

| 스코프 | 디렉토리 | 누가 관리 | 비고 |
|--------|---------|----------|------|
| Managed | `getManagedClaudeRulesDir()` (예: `/etc/claude-code/rules/`) | 조직 정책 | 항상 로드 — `userSettings` 비활성화와 무관 |
| User | `~/.claude/rules/` | 사용자 본인 | `userSettings`가 켜져 있을 때만 |
| Project | `<repo>/.claude/rules/` | 팀 (체크인됨) | CWD에서 root까지 디렉토리 walk하며 각 단계 스캔 |
| Project (`--add-dir`) | `<additional-dir>/.claude/rules/` | 사용자가 명시한 추가 디렉토리 | `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` env로 게이팅 |

각 스코프에서 `.claude/rules/` 안의 모든 `.md` 파일을 **재귀적으로** 읽는다 ([processMdRules()](../../src/utils/claudemd.ts#L706)). 서브디렉토리로 그룹화해도 모두 발견된다.

## CLAUDE.md와의 핵심 차이

| 항목 | CLAUDE.md | `.claude/rules/*.md` |
|------|-----------|----------------------|
| 파일 수 | 디렉토리당 1개 | 임의 개수 + 서브디렉토리 |
| `paths` frontmatter | 무시 | **핵심 기능** |
| 로드 시점 | 세션 시작 (eager) | 정적 규칙은 eager, 조건부 규칙은 파일 터치 시 dynamic |
| 적용 범위 | 항상 system prompt에 포함 | 정적은 system prompt, 조건부는 매칭된 파일과 함께 attachment로 |

> 같은 디렉토리에 `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`를 함께 둘 수 있고, 이들은 **모두 다 로드된다** ([claudemd.ts:895-928](../../src/utils/claudemd.ts#L895-L928)).

## 정적 규칙 vs 조건부 규칙

- **정적 (unconditional)** — frontmatter에 `paths`가 없거나 `**` 같은 match-all 패턴인 파일. 세션 시작 시 한 번 로드되어 system prompt에 합쳐진다.
- **조건부 (conditional)** — frontmatter에 `paths` 패턴이 있는 파일. 어떤 파일을 읽거나 편집할 때 그 경로가 패턴에 매칭되는 경우에만 동적으로 주입된다.

이 두 가지를 가르는 코드는 [parseFrontmatterPaths()](../../src/utils/claudemd.ts#L254)와 [processMdRules()의 conditionalRule 파라미터](../../src/utils/claudemd.ts#L706-L784)다.

## 우선순위

[claudemd.ts:1-10](../../src/utils/claudemd.ts#L1-L10) 주석에 명시:

```
1. Managed memory   (/etc/claude-code/CLAUDE.md, /etc/.../rules/*.md)
2. User memory      (~/.claude/CLAUDE.md, ~/.claude/rules/*.md)
3. Project memory   (CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md — root → CWD 순)
4. Local memory     (CLAUDE.local.md)
```

**나중에 로드된 파일이 더 높은 우선순위**를 가진다 — 모델은 뒤쪽 파일에 더 주의를 기울인다. 즉 CWD에 가까운 파일이 가장 강한 영향을 준다.
