# 에이전트 정의 형식 (Definition Format)

사용자는 다음 세 위치에 마크다운 또는 JSON 파일로 커스텀 에이전트를 정의할 수 있습니다.

| 범위 | 경로 | 우선순위 |
|------|------|---------|
| managed (관리자 정책) | `{managedPath}/.claude/agents/*.md` | 가장 높음 |
| flag (CLI 플래그) | 런타임 주입 | ↑ |
| project | `<git root 이내>/.claude/agents/*.md` | ↑ |
| user | `~/.claude/agents/*.md` | ↑ |
| plugin | 플러그인이 주입 | ↑ |
| built-in | [src/tools/AgentTool/built-in/](../../src/tools/AgentTool/built-in/) | 가장 낮음 |

> 같은 `name`이 여러 범위에 존재하면 더 높은 우선순위가 이전 정의를 덮어씁니다. 자세한 병합 로직은 [02_loading.md](02_loading.md) 참조.

핵심 파일:
- [src/tools/AgentTool/loadAgentsDir.ts](../../src/tools/AgentTool/loadAgentsDir.ts)
- [src/utils/markdownConfigLoader.ts](../../src/utils/markdownConfigLoader.ts)

---

## 1. 마크다운 형식

가장 일반적인 방식입니다. 파일명에서 `name`을 얻고 frontmatter로 메타데이터, 본문으로 시스템 프롬프트를 지정합니다.

```markdown
---
name: search-specialist                # 파일명에서도 추출 가능
description: "Web search specialist for finding recent docs"
tools: [WebSearch, WebFetch, Read, Glob, Grep]
disallowedTools: [Bash]
model: sonnet                           # inherit | opus | sonnet | haiku
permissionMode: plan                    # plan | acceptEdits | bypassPermissions
maxTurns: 20
memory: project                         # user | project | local
background: false
isolation: worktree                     # worktree | remote
color: cyan
---

You are a web research specialist. Your job is to ...
(본문 = 에이전트의 시스템 프롬프트)
```

파싱은 [parseAgentFromMarkdown()](../../src/tools/AgentTool/loadAgentsDir.ts) 에서 수행됩니다.

---

## 2. Frontmatter 필드 레퍼런스

스키마는 [loadAgentsDir.ts](../../src/tools/AgentTool/loadAgentsDir.ts)의 `AgentJsonSchema` (Zod)에 정의되어 있습니다.

| 필드 | 타입 | 설명 |
|------|------|------|
| `name` | string | 에이전트 타입(=`subagent_type`). 파일명에서도 유도 가능 |
| `description` | string | 언제 사용할지 — LLM이 에이전트 선택 시 참조 (`whenToUse`) |
| `tools` | `string[]` \| `'*'` \| `undefined` | 허용할 도구. 미지정/`*` → 필터링된 모든 도구, `[]` → 없음 |
| `disallowedTools` | `string[]` | 명시적 차단 도구 |
| `prompt` | string | JSON 에이전트용 시스템 프롬프트 (마크다운은 본문 사용) |
| `model` | `'inherit'` \| `'sonnet'` \| `'opus'` \| `'haiku'` \| string | 모델 오버라이드. `inherit` → 부모 모델 상속 |
| `effort` | `'low'` \| `'medium'` \| `'high'` \| number | 사고 예산 힌트 |
| `permissionMode` | `'plan'` \| `'acceptEdits'` \| `'bypassPermissions'` | 권한 모드 오버라이드 |
| `mcpServers` | `AgentMcpServerSpec[]` | 에이전트 전용 MCP 서버 스펙 |
| `hooks` | `HooksSettings` | 세션 훅 (start/stop 등) |
| `maxTurns` | number | 최대 대화 턴 수 (양의 정수) |
| `skills` | `string[]` | 에이전트 시작 시 사전 로드할 스킬/슬래시 커맨드 |
| `initialPrompt` | string | 첫 턴 프롬프트 접두사 |
| `memory` | `'user'` \| `'project'` \| `'local'` | 영속 메모리 범위. 지정 시 Read/Write/Edit 자동 주입 |
| `background` | boolean | `true` → 항상 백그라운드로 실행 |
| `isolation` | `'worktree'` \| `'remote'` | 격리 모드. [06_async_and_isolation.md](06_async_and_isolation.md) 참조 |
| `color` | string | UI 라벨 색상 ([agentColorManager.ts](../../src/tools/AgentTool/agentColorManager.ts)) |

### 2.1 `tools` 필드 해석

[parseAgentToolsFromFrontmatter()](../../src/utils/markdownConfigLoader.ts) 기준:

| 값 | 의미 |
|----|------|
| 미정의 | 필터링된 모든 도구 허용 (기본값) |
| `'*'` 또는 `['*']` | 모든 도구 허용 |
| `[]` (빈 배열) | 도구 없음 (순수 추론 전용) |
| `['WebSearch', 'Read']` | 해당 도구만 화이트리스트 |

> 이 단계의 필터링은 "에이전트 정의가 요청하는" 허용 목록이며, 이후 [05_permissions.md](05_permissions.md)의 글로벌 차단 집합(`ALL_AGENT_DISALLOWED_TOOLS` 등)이 추가로 적용됩니다.

### 2.2 `memory` 필드와 자동 도구 주입

`memory`가 지정되면 에이전트는 해당 범위에서 영속 메모리 파일에 접근할 수 있어야 하므로, FileRead/FileWrite/FileEdit가 자동으로 `tools` 목록에 합쳐집니다. 관련 코드는 [agentMemory.ts](../../src/tools/AgentTool/agentMemory.ts)를 참고하세요.

---

## 3. 본문 (System Prompt)

마크다운 본문은 전부 시스템 프롬프트로 사용됩니다. 실제 런타임에서는 본문만으로 끝나지 않고, [04_execution.md](04_execution.md)의 `getAgentSystemPrompt()`가 본문에 환경 정보(현재 작업 디렉토리, 사용 가능한 도구 목록 등)를 덧붙여 완성된 프롬프트를 만듭니다.

---

## 4. JSON 에이전트 (대체 형식)

마크다운 대신 JSON을 사용하는 경로도 지원합니다. 파싱은 [parseAgentFromJson()](../../src/tools/AgentTool/loadAgentsDir.ts)에서 수행되며, 필드 의미는 위 표와 동일합니다. 차이는 시스템 프롬프트를 본문이 아닌 `prompt` 필드에 넣는다는 점입니다.

```json
{
  "name": "safe-reviewer",
  "description": "Reviews diffs without running code",
  "tools": ["Read", "Grep", "Glob"],
  "model": "inherit",
  "prompt": "You are a senior reviewer. Focus on..."
}
```

---

## 5. 예시: 최소한의 커스텀 에이전트

```markdown
---
name: doc-finder
description: "Finds relevant documentation files in the repo"
tools: [Glob, Grep, Read]
---

You help the user locate documentation. Search the repository for markdown
files and report the three most relevant with file paths and one-line summaries.
```

이 파일을 `.claude/agents/doc-finder.md`에 저장하면, 다음 세션부터 LLM은
`Agent` 도구를 `subagent_type="doc-finder"`로 호출할 수 있습니다.
