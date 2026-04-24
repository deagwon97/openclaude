# 커스텀 슬래시 커맨드 처리 (`.claude/commands/`)

openclaude가 사용자 정의 슬래시 커맨드(`.claude/commands/*.md`)를 어떻게 발견하고, 파싱하고, 실행하는지 정리한 문서 모음입니다.

## 목차

1. [01-overview.md](01-overview.md) — 전체 파이프라인 한눈에 보기
2. [02-discovery.md](02-discovery.md) — 커맨드 파일 발견(Discovery) 흐름
3. [03-parsing.md](03-parsing.md) — Frontmatter 파싱 및 메타데이터 정규화
4. [04-registration.md](04-registration.md) — 커맨드 객체 생성과 레지스트리 등록
5. [05-execution.md](05-execution.md) — 입력 라우팅부터 Claude 메시지 전달까지
6. [06-special-syntax.md](06-special-syntax.md) — `$ARGUMENTS`, `!`bash``, `@file`, 조건부 `paths`

## 한 줄 요약

`.claude/commands/*.md` 파일은 시작 시 [src/skills/loadSkillsDir.ts](../../src/skills/loadSkillsDir.ts)가 스캔해서 `type: 'prompt'` 커맨드로 변환한 뒤 [src/commands.ts](../../src/commands.ts)의 레지스트리에 합쳐지고, 사용자가 `/name args`를 입력하면 [src/utils/processUserInput/processSlashCommand.tsx](../../src/utils/processUserInput/processSlashCommand.tsx)가 파싱·치환·훅 등록 후 Claude에게 `isMeta` 메시지로 전달합니다.
