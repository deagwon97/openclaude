# 01. 메시지/대화 컨텍스트 관리

## 메시지 타입 구조

파일: `src/utils/messages.ts` (~5300줄)

```
Message
├── UserMessage         사용자 입력 또는 tool_result 응답
├── AssistantMessage    Claude API 응답 (BetaMessage 래핑)
├── SystemMessage       내부 시스템 이벤트 (compact boundary, 에러 등)
├── AttachmentMessage   메모리, 스킬, MCP 지침 등 부속 정보
└── ProgressMessage     Tool 실행 진행 상황 (렌더링 전용, API 미전송)
```

### UserMessage

```typescript
type UserMessage = {
  type: 'user'
  uuid: string               // 고유 ID
  parentUuid: string | null  // 이전 메시지 링크 (chain 구조)
  timestamp: string

  content: string | ContentBlockParam[]  // 텍스트 또는 멀티모달
  isMeta?: boolean           // 내부 처리용 (API 미전송)
  isVirtual?: boolean        // REPL 전용 (API 미전송)

  // Tool result 포함 시
  toolUseResult?: ToolResultBlockParam
  sourceToolAssistantUUID?: string  // 대응하는 AssistantMessage UUID
}
```

### AssistantMessage

```typescript
type AssistantMessage = {
  type: 'assistant'
  uuid: string
  parentUuid: string | null
  timestamp: string

  message: BetaMessage       // Anthropic SDK의 원본 응답
  requestId?: string         // 디버깅/추적용
  usage?: {
    input_tokens: number
    output_tokens: number
    cache_read_input_tokens?: number    // 캐시 히트
    cache_creation_input_tokens?: number // 캐시 쓰기
  }
  stop_reason?: string
  apiError?: string
}
```

## 메시지 체인 구조

메시지들은 `uuid` + `parentUuid` 링크로 연결됩니다:

```
UserMessage (uuid: "aaa", parentUuid: null)
    │
    ▼
AssistantMessage (uuid: "bbb", parentUuid: "aaa")
    │
    ▼
UserMessage (uuid: "ccc", parentUuid: "bbb")   ← tool_result 포함 가능
    │
    ▼
AssistantMessage (uuid: "ddd", parentUuid: "ccc")
```

세션 저장/로드 시 이 링크로 대화 흐름을 재구성합니다.

## 메시지 생성

```typescript
// 사용자 메시지 생성
createUserMessage(content, options) → UserMessage
  // UUID 자동 할당
  // timestamp = Date.now()
  // content: string 또는 ContentBlockParam[] (이미지, 문서 포함 가능)

// 어시스턴트 메시지 생성
createAssistantMessage(betaMessage, options) → AssistantMessage
  // API 응답을 래핑
  // usage 정보 보존
```

## API 전송 전 정규화

API 호출 전 `normalizeMessagesForAPI()` 를 통해 메시지를 필터링합니다:

| 처리 | 내용 |
|------|------|
| 가상(virtual) 메시지 제거 | REPL 내부용 메시지는 API에 미전송 |
| isMeta 메시지 제거 | 내부 이벤트 메시지 제거 |
| Tool result 짝 확인 | `ensureToolResultPairing()` 호출 |
| 이미지/문서 블록 정렬 | `ContentBlockParam` 순서 정규화 |

### ensureToolResultPairing()

모든 `tool_use` 블록에 대응하는 `tool_result`가 있는지 검증합니다.
누락된 경우 합성(synthetic) tool_result를 삽입:

```typescript
const SYNTHETIC_TOOL_RESULT_PLACEHOLDER =
  '[Tool result missing due to internal error]'
```
