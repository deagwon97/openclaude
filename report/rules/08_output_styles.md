# 08. Output Styles — 응답 스타일 커스터마이징

`.claude/output-styles/*.md` 파일은 **에이전트의 응답 스타일** 을 바꾸는 설정이다. `.claude/rules` 와 마찬가지로 markdown + frontmatter 형식이고 디렉토리 스캔으로 로드되지만, 적용 지점과 의미가 다르다:

| 항목 | `rules/*.md` | `output-styles/*.md` |
|------|------------|---------------------|
| 적용 범위 | 조건부 (frontmatter `paths`) 또는 항상 | **항상** (활성화된 1개만) |
| 주입 위치 | system prompt 본체 + `nested_memory` attachment | system prompt의 identity 섹션 + `output_style` attachment |
| 활성화 | 디렉토리에 있으면 로드 | `settings.outputStyle` 에 **이름으로 지정** 해야 활성 |
| 개수 | 여러 개 동시 적용 | **1개만** 활성 (또는 'default' = 없음) |

## 1. 파일 형식

```markdown
---
name: MyStyle
description: 한 줄 설명
keep-coding-instructions: true
---

본문이 그대로 system prompt의 "Output Style" 섹션이 된다.
```

- `name` (optional): 프런트매터 미지정 시 파일명에서 자동 추론 (확장자 제외)
- `description`: `/output-style` 메뉴에서 보이는 설명
- `keep-coding-instructions` (boolean): `true` 면 기본 코딩 안내 프롬프트를 **유지**, `false`/미지정이면 코딩 섹션을 통째로 대체
- `force-for-plugin` (plugin 전용): 플러그인 출력 스타일을 자동 활성화

## 2. 내장 스타일

[src/constants/outputStyles.ts](../../src/constants/outputStyles.ts) 에 하드코딩:

| 이름 | 설명 |
|------|------|
| `default` | `null` — 특수값, 커스텀 스타일 없음 |
| `Explanatory` | 구현 선택과 코드베이스 패턴을 교육적으로 설명 |
| `Learning` | 사용자에게 짧은 코드 작성 과제를 요청 |

`Explanatory` 와 `Learning` 모두 공통 `EXPLANATORY_FEATURE_PROMPT` 를 공유한다 — "Insight" 박스 포맷팅 규칙.

## 3. 로드 파이프라인

```
.claude/output-styles/*.md  (project, user, managed)
plugin output styles        (로드된 플러그인)
          ↓
getOutputStyleDirStyles(cwd)     ← loadMarkdownFilesForSubdir 'output-styles'
          ↓
getAllOutputStyles(cwd)          ← 내장 + 플러그인 + 커스텀 병합
          ↓  우선순위: built-in < plugin < user < project < managed
getOutputStyleConfig()           ← settings.outputStyle 로 1개 선택
          ↓
          시스템 프롬프트 + attachment 주입
```

### 핵심 파일

| 역할 | 파일 |
|------|------|
| 디렉토리 스캔 | [src/outputStyles/loadOutputStylesDir.ts](../../src/outputStyles/loadOutputStylesDir.ts) |
| 내장 정의 + 병합 | [src/constants/outputStyles.ts](../../src/constants/outputStyles.ts) |
| 시스템 프롬프트 주입 | [src/constants/prompts.ts](../../src/constants/prompts.ts) `getOutputStyleSection` |
| Attachment 생성 | [src/utils/attachments.ts](../../src/utils/attachments.ts) `getOutputStyleAttachment` |
| 플러그인 스타일 로드 | [src/utils/plugins/loadPluginOutputStyles.ts](../../src/utils/plugins/loadPluginOutputStyles.ts) |

## 4. 디렉토리 스캔 — `getOutputStyleDirStyles`

[src/outputStyles/loadOutputStylesDir.ts](../../src/outputStyles/loadOutputStylesDir.ts)

```typescript
export const getOutputStyleDirStyles = memoize(
  async (cwd: string): Promise<OutputStyleConfig[]> => {
    const markdownFiles = await loadMarkdownFilesForSubdir('output-styles', cwd)
    // ...각 파일의 frontmatter 파싱, styleName=파일명, prompt=content
  }
)
```

- `rules/` 와 **동일한** `loadMarkdownFilesForSubdir` 헬퍼를 재사용 (`.claude/output-styles`, `~/.claude/output-styles`, managed 경로 모두 스캔)
- 각 파일의 `source` 필드로 `projectSettings | userSettings | policySettings` 구분
- `memoize` 로 cwd 단위 캐시 — `clearOutputStyleCaches()` 로 무효화

## 5. 우선순위 병합 — `getAllOutputStyles`

[src/constants/outputStyles.ts:137-175](../../src/constants/outputStyles.ts#L137-L175)

```
built-in (Explanatory, Learning)
  ← plugin styles
  ← user styles   (~/.claude/output-styles)
  ← project styles (.claude/output-styles)
  ← managed styles (policy)
```

낮은 우선순위부터 덮어쓰는 방식. 같은 `name` 이면 higher-priority 가 이긴다.

> ⚠ 이것은 `rules/` 와 **반대** 방향이다. rules 는 적용 `paths` 로 scope 이 나뉘어 병합 개념이 다름.

## 6. 활성 스타일 선택 — `getOutputStyleConfig`

[src/constants/outputStyles.ts:181-211](../../src/constants/outputStyles.ts#L181-L211)

결정 순서:

1. **Forced plugin style**: 플러그인이 `forceForPlugin: true` 인 스타일을 내놓으면 무조건 이것 사용 (여러 개면 첫 번째 + 경고 로그)
2. **settings.outputStyle**: 사용자가 명시한 이름의 스타일
3. `null` 반환 = `default` = 커스텀 스타일 없음

## 7. System Prompt 주입 — `getOutputStyleSection`

[src/constants/prompts.ts:152-157](../../src/constants/prompts.ts#L152-L157)

```typescript
function getOutputStyleSection(
  outputStyleConfig: OutputStyleConfig | null,
): string | null {
  if (outputStyleConfig === null) return null
  return `# Output Style: ${outputStyleConfig.name}
${outputStyleConfig.prompt}`
}
```

시스템 프롬프트 상단 identity 문구도 영향받음:

```typescript
`You are an interactive agent that helps users
${outputStyleConfig !== null
  ? 'according to your "Output Style" below, which describes how you should respond to user queries.'
  : 'with software engineering tasks.'}`
```

스타일이 활성이면 "Output Style 에 따라 응답하라" 로, 없으면 기본 문구로.

### keepCodingInstructions

`keep-coding-instructions: false` (또는 미지정) 일 때 하네스는 기본 "Doing tasks" / "Tone and style" 섹션을 **드롭**한다. 스타일이 완전히 대체하기 원할 때 씀 (예: 비기술 모드).

`true` 면 기본 지침은 남기고 Output Style 섹션을 추가 — 일반적인 Explanatory / Learning 패턴.

## 8. Attachment 주입 — `getOutputStyleAttachment`

[src/utils/attachments.ts:1598-1613](../../src/utils/attachments.ts#L1598-L1613)

첫 사용자 턴에 다음 형태의 system reminder 가 추가된다:

```
{name} output style is active. Remember to follow the specific guidelines for this style.
```

`default` 면 생략. 긴 대화에서 모델이 스타일 적용을 잊지 않도록 **리마인더** 역할.

## 9. `/output-style` 커맨드와의 관계

`/output-style` 커맨드로 스타일을 바꾸면:
1. `settings.outputStyle` 을 업데이트 → settings.json 저장
2. `clearAllOutputStylesCache()` 호출 → 다음 쿼리부터 반영
3. `hasCustomOutputStyle()` 로 footer 에 표시

## 10. rules/ vs output-styles/ 요약

```
rules/*.md           ←→   output-styles/*.md
───────────────              ────────────────
여러 개 동시 적용          1개만 활성 (settings)
조건부 paths             항상 적용
nested_memory            output_style (attachment)
system prompt 본문       Output Style 섹션 + identity 교체
독립                     keepCodingInstructions 로
                         기본 지침 제거 가능
```

두 시스템 모두 `loadMarkdownFilesForSubdir` 를 공유하지만, **활성화 모델과 주입 위치가 다르다**. rules 는 "추가" 개념, output-styles 는 "선택" 개념.
