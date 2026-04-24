# OpenClaude Skills 처리 메커니즘

`.claude/skills` 디렉토리에 정의된 사용자 Skill이 어떻게 로드되고, 파싱되고, 모델에 노출되며, 실행되는지 정리한 보고서.

## 목차

1. [01-loading.md](01-loading.md) — Skill 로드 경로와 발견 메커니즘
2. [02-format.md](02-format.md) — SKILL.md 파일 포맷과 Frontmatter 필드
3. [03-skill-tool.md](03-skill-tool.md) — `Skill` tool 구현과 호출 파이프라인
4. [04-execution-modes.md](04-execution-modes.md) — Inline vs Forked 실행 모드
5. [05-hooks-and-permissions.md](05-hooks-and-permissions.md) — Hook 등록 및 권한 체크
6. [06-file-map.md](06-file-map.md) — 핵심 파일/함수 빠른 참조 맵

## 한 줄 요약

Skills는 `~/.claude/skills` 및 `<project>/.claude/skills` 아래의 `SKILL.md` 파일을 스캔해 `Command` 객체로 변환한 뒤,
모델에게는 [SkillTool](../../src/tools/SkillTool/SkillTool.ts)의 "사용 가능한 skill 목록"으로 노출되고,
호출되면 frontmatter의 `context` 값에 따라 **inline(현재 대화에 콘텐츠 주입)** 또는 **fork(독립 sub-agent 실행)** 으로 처리된다.

## 전체 흐름 한눈에 보기

```
[디스크]                      [로딩]                      [모델 노출]                [실행]
.claude/skills/<name>/  →  loadSkillsDir   →  getSkillToolCommands  →  SkillTool.call()
  SKILL.md                 (frontmatter       (시스템 프롬프트에          ├─ inline:  메시지 주입
                            + markdown 본문)    skill 목록 주입)          └─ fork:    runAgent()
```
