# Context 관리 방식 분석

OpenClaude CLI의 context 관리 방식을 5개 영역으로 분류하여 정리합니다.

## 목차

| 파일 | 내용 |
|------|------|
| [01_message_history.md](01_message_history.md) | 대화 히스토리 저장 구조 및 메시지 타입 |
| [02_context_compression.md](02_context_compression.md) | 자동 컨텍스트 압축(Auto-Compact) 프로세스 |
| [03_tool_context.md](03_tool_context.md) | Tool 실행 결과의 컨텍스트 포함 방식 |
| [04_session_state.md](04_session_state.md) | 세션/글로벌 상태 관리 및 저장 |
| [05_system_prompt.md](05_system_prompt.md) | 시스템 프롬프트 조립 및 컨텍스트 주입 |

## 전체 흐름 요약

```
사용자 입력
    │
    ▼
[1] createUserMessage()          src/utils/messages.ts
    메시지 생성 (UUID, timestamp)
    │
    ▼
[2] 토큰 계산 → Auto-Compact 판단  src/services/compact/autoCompact.ts
    ├─ 임계값 초과: compactConversation()
    └─ 통과: 그대로 진행
    │
    ▼
[3] normalizeMessagesForAPI()    src/utils/messages.ts
    가상 메시지 제거 / tool_result 짝 맞추기
    │
    ▼
[4] Context 주입                 src/utils/api.ts
    ├─ prependUserContext()  → CLAUDE.md, 날짜
    └─ appendSystemContext() → Git 상태
    │
    ▼
[5] Claude API 호출              src/query.ts
    messages + systemPrompt + tools
    │
    ▼
[6] 응답 처리
    ├─ createAssistantMessage()
    └─ tool_use → runTools() → tool_result
    │
    ▼
[7] 세션 저장                    src/utils/sessionStorage.ts
    JSONL 파일에 메시지 append
```

## 핵심 파일 목록

| 파일 | 역할 |
|------|------|
| `src/utils/messages.ts` | 메시지 타입 정의 및 생성/정규화 함수 (~5300줄) |
| `src/context.ts` | Git/날짜 컨텍스트 수집 |
| `src/utils/api.ts` | 시스템 프롬프트 조립, 컨텍스트 주입 |
| `src/services/compact/compact.ts` | 컨텍스트 압축 실행 |
| `src/services/compact/autoCompact.ts` | 자동 압축 트리거 조건 |
| `src/utils/tokens.ts` | 토큰 카운팅 |
| `src/utils/sessionStorage.ts` | JSONL 세션 저장/로드 |
| `src/bootstrap/state.ts` | 글로벌 상태 타입 정의 |
| `src/query.ts` | Claude API 호출 루프 |
