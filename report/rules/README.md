# `.claude/rules` 처리 메커니즘 분석

OpenClaude CLI가 사용자가 작성한 `.claude/rules/*.md` 파일을 어떻게 발견·로드·필터링하고, 어떤 경로로 시스템 프롬프트나 메시지에 주입하는지를 정리한 문서입니다. CLAUDE.md 계열 메모리 파일과의 관계도 함께 다룹니다.

## 문서 목록 (읽는 순서)

| # | 파일 | 주제 |
|---|------|------|
| 01 | [01_overview.md](01_overview.md) | `.claude/rules`란 무엇이며 CLAUDE.md와 어떻게 다른가 |
| 02 | [02_file_format.md](02_file_format.md) | 파일 형식, frontmatter `paths`, `@include` 지시문 |
| 03 | [03_load_flow.md](03_load_flow.md) | 세션 시작 시 eager 로드 vs 파일 단위 dynamic 로드 |
| 04 | [04_injection.md](04_injection.md) | system prompt 주입 vs `nested_memory` attachment 주입 |
| 05 | [05_conditional_rules.md](05_conditional_rules.md) | 조건부 규칙의 glob 매칭과 baseDir 결정 규칙 |
| 06 | [06_key_functions.md](06_key_functions.md) | 핵심 함수 / 파일 / 라인 번호 레퍼런스 |
| 07 | [07_related_concepts.md](07_related_concepts.md) | CLAUDE.md, skills, commands, AutoMem과의 차이 |
| 08 | [08_output_styles.md](08_output_styles.md) | `.claude/output-styles/*.md` — 응답 스타일 커스터마이징 |

---

## 핵심 파일 한눈에 보기

| 역할 | 파일 |
|------|------|
| 메모리·rules 파일 발견·파싱 | [src/utils/claudemd.ts](../../src/utils/claudemd.ts) |
| Frontmatter (`paths` 등) 파서 | [src/utils/frontmatterParser.ts](../../src/utils/frontmatterParser.ts) |
| Managed/User rules 디렉토리 경로 | [src/utils/config.ts](../../src/utils/config.ts) |
| `nested_memory` attachment 생성 | [src/utils/attachments.ts](../../src/utils/attachments.ts) |
| system prompt에 `claudeMd` 주입 | [src/context.ts](../../src/context.ts) |
| `InstructionsLoaded` hook 디스패치 | [src/utils/hooks.ts](../../src/utils/hooks.ts) |

---

## 한 문장 요약

`.claude/rules/*.md`는 frontmatter `paths`로 적용 범위를 좁힐 수 있는 세분화된 CLAUDE.md다. 정적 규칙은 세션 시작 시 [getMemoryFiles()](../../src/utils/claudemd.ts#L799)가 한 번 모아 system prompt에 합치고, 조건부 규칙은 파일을 만질 때마다 [getNestedMemoryAttachmentsForFile()](../../src/utils/attachments.ts#L1793)가 glob 매칭으로 골라 `nested_memory` attachment로 끼워 넣는다.
