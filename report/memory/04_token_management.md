# 토큰 측정 및 컨텍스트 윈도우 관리

핵심 파일:
- [src/utils/tokens.ts](../../src/utils/tokens.ts)
- [src/utils/context.ts](../../src/utils/context.ts)

---

## 토큰 사용량 측정

### getTokenUsage()

```typescript
// src/utils/tokens.ts:7-20
export function getTokenUsage(message: Message): Usage | undefined {
  // assistant 메시지에서 usage 필드 추출
  // SYNTHETIC_MESSAGES(합성 메시지)는 제외
  // SYNTHETIC_MODEL 응답은 제외
}
```

Usage 구조:
```typescript
{
  input_tokens: number,
  output_tokens: number,
  cache_creation_input_tokens: number,   // 새 캐시 생성 토큰
  cache_read_input_tokens: number,       // 캐시 읽기 토큰
}
```

### tokenCountFromLastAPIResponse()

```typescript
// src/utils/tokens.ts:55-66
export function tokenCountFromLastAPIResponse(messages: Message[]): number {
  // 메시지 배열을 역순으로 순회
  // 가장 최근 assistant 메시지의 usage 합산 반환
  // (input + output + cache_creation + cache_read)
}
```

---

## 컨텍스트 윈도우 크기

### 기본값

```typescript
// src/utils/context.ts
MODEL_CONTEXT_WINDOW_DEFAULT = 200_000   // 기본 200k 토큰
COMPACT_MAX_OUTPUT_TOKENS = 20_000       // 압축 요약 최대 출력 토큰
CAPPED_DEFAULT_MAX_TOKENS = 8_000        // 기본 응답 상한
ESCALATED_MAX_TOKENS = 64_000            // 에스컬레이션된 최대 응답
```

### getContextWindowForModel()

```typescript
// src/utils/context.ts:9-118
export function getContextWindowForModel(model: string, betas?: string[]): number
```

모델 ID → 컨텍스트 윈도우 크기 결정. 우선순위:

| 순위 | 소스 |
|------|------|
| 1 | `CLAUDE_CODE_MAX_CONTEXT_TOKENS` 환경변수 (내부 오버라이드) |
| 2 | 모델 ID에 `[1m]` 접미사 → 1,000,000 토큰 |
| 3 | OpenAI 호환 모델 테이블 |
| 4 | 모델 기능 정보의 `max_input_tokens` |
| 5 | `CONTEXT_1M_BETA_HEADER` 활성화 여부 |
| 6 | 기본값 200,000 |

---

## 컨텍스트 사용률 계산

```typescript
// src/utils/context.ts:138-150
export function calculateContextPercentages(
  currentUsage: {
    input_tokens: number,
    cache_creation_input_tokens: number,
    cache_read_input_tokens: number,
  } | null,
  contextWindowSize: number,
): { used: number | null, remaining: number | null }
```

UI 상단 컨텍스트 게이지에 표시되는 값을 계산한다.

---

## 토큰 초과 처리 흐름

```
API 호출
  ↓
PROMPT_TOO_LONG_ERROR 수신
  ↓
getPromptTooLongTokenGap()
  → 부족한 토큰 양 계산
  ↓
압축 여부 결정
  - 자동 압축 활성화 → compact 실행
  - 비활성화 → 에러 메시지 표시
  ↓
[압축 실행 시]
stripImagesFromMessages() → API 호출 (요약 생성)
  ↓
새 컨텍스트 = 경계 마커 + 요약 + 최근 메시지
  ↓
재시도 (최대 MAX_COMPACT_STREAMING_RETRIES = 2회)
```

---

## 캐시 토큰 처리

Anthropic의 prompt caching을 활용한다.

- `cache_creation_input_tokens`: 이번 요청에서 새로 캐시를 만든 토큰
- `cache_read_input_tokens`: 캐시에서 읽은 토큰 (비용 절감)

비용 표시 시 캐시 읽기 토큰은 할인율이 적용된다.
