# 07. 비슷한 개념과의 차이

`.claude/` 안에는 `rules/` 외에도 비슷해 보이는 디렉토리가 여럿 있다. 모두 마크다운 + frontmatter 조합이지만 역할이 서로 다르다.

## 한눈에 비교

| 개념 | 위치 | 한 줄 정의 | 실행 가능? | `paths` frontmatter |
|------|------|-----------|----------|-------------------|
| **CLAUDE.md** | `<repo>/CLAUDE.md`, `~/.claude/CLAUDE.md` | 디렉토리당 1개, 무조건 system prompt에 포함되는 명령문 | ✕ | 무시 |
| **.claude/rules/*.md** | `<scope>/.claude/rules/` | 여러 파일로 분할 가능, `paths`로 적용 범위 좁히기 | ✕ | **핵심 기능** |
| **CLAUDE.local.md** | `<repo>/CLAUDE.local.md` | gitignored 개인 명령문 | ✕ | 무시 |
| **.claude/skills/*.md** | `<scope>/.claude/skills/` | 모델이 호출 가능한 "기술" 정의, 실행 컨텍스트 포함 | ○ | 사용 가능 |
| **.claude/commands/*.md** | `<scope>/.claude/commands/` | 사용자가 `/` 슬래시로 호출하는 매크로 | ○ | 사용 가능 |
| **.claude/agents/** | `<scope>/.claude/agents/` | 서브에이전트 정의 | ○ (Agent tool) | — |
| **AutoMem (memdir)** | `~/.claude/projects/<hash>/memory/` | 대화 간 지속되는 자동 메모리 | ✕ | — |
| **TeamMem** (TEAMMEM 기능 플래그) | 조직 동기화 경로 | 팀 공유 메모리 | ✕ | — |

## 실행 가능 vs 명령문

이 분류가 가장 중요하다.

- **명령문(Instructions)** — `CLAUDE.md`, `.claude/rules/*.md`, `CLAUDE.local.md`. 텍스트일 뿐이고 모델 컨텍스트에 들어가서 모델 행동을 가이드한다. `getMemoryFiles()`가 모은다.
- **실행 가능(Executable)** — `skills/`, `commands/`, `agents/`. 메타데이터(이름, 설명, 도구 권한)를 통해 모델이나 사용자가 명시적으로 호출하는 단위. 별도 로더가 처리한다.

`.claude/rules`는 **순수 명령문**이다. 무엇을 "실행"하지 않고, 그저 모델이 읽을 텍스트를 제공한다.

## 같은 frontmatter, 다른 의미

`paths` 필드는 [frontmatterParser.ts](../../src/utils/frontmatterParser.ts)에 한 번만 정의되고 rules / skills / commands가 모두 같은 파서를 쓴다. 의미는 비슷하다 — "이 파일이 특정 경로 패턴을 다룰 때만 활성화된다". 다른 점:

- **rules의 paths** → 매칭되면 그 파일에 대한 user 메시지에 명령문 본문이 attachment로 붙는다.
- **skills의 paths** → 매칭되면 skill 이 활성화 풀에 들어가 모델이 호출할 수 있게 된다.
- **commands의 paths** → 슬래시 명령어 자동 완성 후보에 포함된다.

## CLAUDE.md vs `.claude/rules/*.md` 선택 가이드

| 상황 | 추천 |
|------|------|
| 모든 작업에 항상 적용되는 짧은 컨벤션 | CLAUDE.md |
| 큰 프로젝트인데 명령문이 길어지고 주제가 섞여 있음 | `.claude/rules/{topic}.md`로 분할 |
| 특정 디렉토리/파일 유형에만 해당하는 규칙 | `.claude/rules/foo.md` + `paths: "..."` |
| 개인용 (체크인 안 함) | `CLAUDE.local.md` |
| 조직 정책 (강제) | Managed `/etc/claude-code/...` |

CLAUDE.md를 두면서 `.claude/rules/`도 같이 둘 수 있다 — 둘은 보완적이다.

## 같이 보면 좋은 메모리 시스템

- **AutoMem (memdir)** — 대화 간 영속되는 자동 학습 메모리. `getMemoryFiles()`의 마지막에 entrypoint가 한 번 합쳐진다 ([claudemd.ts:988-1001](../../src/utils/claudemd.ts#L988-L1001)). rules와 달리 모델이 도구로 직접 쓰고/지운다.
- **TeamMem** — `feature('TEAMMEM')` 기능 플래그로 게이팅. 조직 차원의 동기화 메모리. 같은 자리에서 합쳐지지만 출력 시 `<team-memory-content source="shared">` 태그로 구분 ([claudemd.ts:1189-1192](../../src/utils/claudemd.ts#L1189-L1192)).

이 두 메모리는 `InstructionsLoaded` hook에서 의도적으로 제외된다 ([claudemd.ts:1086-1094](../../src/utils/claudemd.ts#L1086-L1094)) — "instructions"가 아닌 별도 메모리 시스템으로 분류되어 있다.
