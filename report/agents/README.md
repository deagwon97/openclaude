# 커스텀 에이전트 (Custom Agents) 런타임 분석

`.claude/agents/` 폴더에 작성한 사용자 정의 에이전트가 OpenClaude 런타임에서 어떻게 로드·등록·실행되는지 정리한 문서입니다.

## 문서 목록

| 파일 | 주제 |
|------|------|
| [01_definition.md](01_definition.md) | 에이전트 정의 형식 (frontmatter, 본문) |
| [02_loading.md](02_loading.md) | 디렉토리 스캔, 파싱, 우선순위, 캐싱 |
| [03_registration.md](03_registration.md) | AgentTool 등록 및 system prompt 주입 |
| [04_execution.md](04_execution.md) | `runAgent()` 실행 루프, 서브 컨텍스트 구성 |
| [05_permissions.md](05_permissions.md) | 도구 필터링과 차단 규칙 |
| [06_async_and_isolation.md](06_async_and_isolation.md) | 백그라운드 실행, worktree 격리, 결과 반환 |
| [07_coordinator_mode.md](07_coordinator_mode.md) | Coordinator 모드 — 메인 에이전트를 워커 조율자로 전환 |

---

## 핵심 파일 한눈에 보기

| 역할 | 파일 |
|------|------|
| 에이전트 로드/파싱 | [src/tools/AgentTool/loadAgentsDir.ts](../../src/tools/AgentTool/loadAgentsDir.ts) |
| 마크다운 디렉토리 스캔 | [src/utils/markdownConfigLoader.ts](../../src/utils/markdownConfigLoader.ts) |
| AgentTool 정의 | [src/tools/AgentTool/AgentTool.tsx](../../src/tools/AgentTool/AgentTool.tsx) |
| 서브에이전트 실행 엔진 | [src/tools/AgentTool/runAgent.ts](../../src/tools/AgentTool/runAgent.ts) |
| 도구 필터링 유틸 | [src/tools/AgentTool/agentToolUtils.ts](../../src/tools/AgentTool/agentToolUtils.ts) |
| 차단 도구 상수 | [src/constants/tools.ts](../../src/constants/tools.ts) |
| System prompt 생성 | [src/tools/AgentTool/prompt.ts](../../src/tools/AgentTool/prompt.ts) |
| 내장 에이전트 | [src/tools/AgentTool/builtInAgents.ts](../../src/tools/AgentTool/builtInAgents.ts) |
| Fork 서브에이전트 | [src/tools/AgentTool/forkSubagent.ts](../../src/tools/AgentTool/forkSubagent.ts) |

---

## 전체 흐름 요약

```
.claude/agents/*.md
        ↓
[스캔]  loadMarkdownFilesForSubdir()              — markdownConfigLoader.ts
        ↓
[파싱]  parseAgentFromMarkdown()                  — loadAgentsDir.ts
        ↓
[병합]  getActiveAgentsFromList()                 — 우선순위별 override
        ↓
[캐시]  getAgentDefinitionsWithOverrides()        — memoize(by cwd)
        ↓
[등록]  AgentTool.prompt() 에서 필터링 후 노출    — AgentTool.tsx
        ↓
        LLM이 Agent tool 호출 (subagent_type 지정)
        ↓
[실행]  AgentTool.call() → runAgent()             — AgentTool.tsx, runAgent.ts
        ├─ 도구 필터 (ALL/CUSTOM/ASYNC DISALLOWED)
        ├─ 시스템 프롬프트 구성 (본문 + env details)
        ├─ MCP 서버 초기화
        ├─ 독립 Subagent Context 생성
        ├─ query() 루프 진입
        └─ sidechain transcript 기록
        ↓
[반환]  yield된 메시지 → 부모에게 최종 텍스트 반환
        또는 백그라운드 태스크 등록 (비동기 경로)
```

---

## 관련 문서

- 에이전트 도구 개요: [../tools/03_agent_task.md](../tools/03_agent_task.md)
- 쿼리 루프: [../query/query-loop.md](../query/query-loop.md)
- 시스템 프롬프트: [../query/system-prompt.md](../query/system-prompt.md)
