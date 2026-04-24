# Query 생성·처리 및 System Prompt 구조 분석

OpenClaude CLI가 사용자 입력을 받아 Claude API로 쿼리를 만들고, 응답 스트림을 tool 호출과 함께 처리하는 전체 파이프라인을 정리한 문서입니다. 시스템 프롬프트가 어떤 블록들로 조립되는지도 함께 다룹니다.

## 문서 목록 (읽는 순서)

| # | 파일 | 주제 |
|---|------|------|
| 01 | [01_entry_and_query_params.md](01_entry_and_query_params.md) | `query()` 진입점, `QueryParams`, `QuerySource`, 호출자 |
| 02 | [02_query_loop.md](02_query_loop.md) | `queryLoop()` while 루프 단계별 분해와 `State` 전이 |
| 03 | [03_api_layer.md](03_api_layer.md) | Streaming / Non-streaming API 호출, 재시도와 폴백 |
| 04 | [04_tool_execution.md](04_tool_execution.md) | `StreamingToolExecutor`의 병렬 실행과 결과 반환 |
| 05 | [05_system_prompt.md](05_system_prompt.md) | **시스템 프롬프트 내부 구조** — 정적/동적 블록, 캐시 경계 |
| 06 | [06_context_injection.md](06_context_injection.md) | `prependUserContext` / `appendSystemContext` 주입 |
| 07 | [07_autocompact.md](07_autocompact.md) | AutoCompact / Reactive Compact의 루프 개입 |

---

## 핵심 파일 한눈에 보기

| 역할 | 파일 |
|------|------|
| 쿼리 엔트리 & 루프 | [src/query.ts](../../src/query.ts) |
| API 호출·스트리밍 | [src/services/api/claude.ts](../../src/services/api/claude.ts) |
| 시스템 프롬프트 조립 | [src/constants/prompts.ts](../../src/constants/prompts.ts) |
| 프롬프트 캐시 분할·컨텍스트 주입 | [src/utils/api.ts](../../src/utils/api.ts) |
| Git/날짜/환경 컨텍스트 수집 | [src/context.ts](../../src/context.ts) |
| Tool 병렬 실행 엔진 | [src/services/tools/StreamingToolExecutor.ts](../../src/services/tools/StreamingToolExecutor.ts) |
| AutoCompact 트리거·실행 | [src/services/compact/autoCompact.ts](../../src/services/compact/autoCompact.ts) |

---

## 전체 흐름 요약

```
REPL / runAgent
      │  (QueryParams 구성)
      ▼
query()                              src/query.ts
      │
      ▼
queryLoop()  ── while(true) ─────────────────────────────┐
  │                                                      │
  │ 1. 메시지 정규화 (budget / snip / microcompact)      │
  │ 2. 시스템 프롬프트 조립 (appendSystemContext)        │
  │ 3. AutoCompact 체크 ──► 필요 시 continue             │
  │ 4. callModel() 스트리밍 ──► yield StreamEvent / msg │
  │ 5. tool_use 감지 ──► StreamingToolExecutor.addTool() │
  │ 6. tool 결과 yield + toolResults 누적                │
  │ 7. 에러 복구 (413 / max_output / fallback)           │
  │ 8. needsFollowUp ? continue : return { done }        │
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

---

## 관련 문서

- 컨텍스트 전반: [../context/README.md](../context/README.md)
- Tool 카탈로그: [../tools/README.md](../tools/README.md)
- 에이전트 실행: [../agents/README.md](../agents/README.md)