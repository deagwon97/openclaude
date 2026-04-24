# 컨텍스트 압축 (Compact)

핵심 파일:
- [src/services/compact/compact.ts](../../src/services/compact/compact.ts)
- [src/services/compact/prompt.ts](../../src/services/compact/prompt.ts)
- [src/utils/messages.ts](../../src/utils/messages.ts) (4535줄~)

---

## 목적

Claude API의 컨텍스트 윈도우(기본 200k 토큰)가 가득 찼을 때  
오래된 대화를 요약(summary)으로 교체하여 대화를 이어가는 메커니즘.

---

## 압축 트리거

| 종류 | 조건 |
|------|------|
| 자동(auto) | API에서 `PROMPT_TOO_LONG_ERROR_MESSAGE` 에러 반환 시 |
| 수동(manual) | 사용자가 `/compact` 명령 실행 시 |

---

## 압축 흐름

```
토큰 초과 감지 (compact.ts)
  ↓
getPromptTooLongTokenGap() — 부족한 토큰 양 계산
  ↓
stripImagesFromMessages() — 압축 전 이미지 제거
  ↓
Claude API 호출 (compact prompt 사용)
  → BASE_COMPACT_PROMPT 또는 PARTIAL_COMPACT_PROMPT
  → <analysis> 블록: 모델의 중간 사고 (최종 결과에서 제거됨)
  → <summary> 블록: 최종 요약 (컨텍스트에 유지됨)
  ↓
createCompactBoundaryMessage() — 경계 마커 삽입
  ↓
[경계 마커] + [요약 메시지] + [경계 이후 최근 메시지]
로 새 컨텍스트 구성
```

---

## 이미지 처리

```typescript
// src/services/compact/compact.ts:144-199
export function stripImagesFromMessages(messages: Message[]): Message[] {
  // image 블록 → { type: 'text', text: '[image]' } 마커로 교체
  // document 블록 → { type: 'text', text: '[document]' } 마커로 교체
  // tool_result 내부 이미지도 동일하게 처리
}
```

압축 API 호출 시 모델에게 이미지를 보내지 않는다.  
이미지가 있던 자리는 `[image]` 텍스트 마커로 남는다.

---

## 경계 마커

### SystemCompactBoundaryMessage

```typescript
// src/utils/messages.ts:4535-4560
{
  type: 'system',
  subtype: 'compact_boundary',
  content: 'Conversation compacted',
  isMeta: false,
  timestamp: ISO 문자열,
  uuid: UUID,
  level: 'info',
  compactMetadata: {
    trigger: 'manual' | 'auto',    // 압축 유발 원인
    preTokens: number,             // 압축 전 토큰 수
    userContext?: string,          // 사용자 정의 컨텍스트 (수동 시)
    messagesSummarized?: number,   // 요약된 메시지 개수
  },
  logicalParentUuid?: UUID,        // 논리적 체인 연결
}
```

### MicrocompactBoundaryMessage

이미지/첨부파일만 선택적으로 제거할 때 사용하는 경량 압축.

```typescript
// src/utils/messages.ts:4562-4588
{
  ...
  compactMetadata: {
    trigger: 'auto',
    preTokens: number,
    tokensSaved: number,            // 절약된 토큰
    compactedToolIds: string[],     // 제거된 tool 호출 ID
    clearedAttachmentUUIDs: string[], // 제거된 첨부 UUID
  }
}
```

---

## 경계 탐색 함수

```typescript
// src/utils/messages.ts
isCompactBoundaryMessage(msg)              // 타입 가드
findLastCompactBoundaryIndex(messages)     // 마지막 경계 인덱스
getMessagesAfterCompactBoundary(messages)  // 경계 이후 메시지만 추출
```

이 함수들은 API에 보낼 메시지를 구성할 때 호출된다.  
이전 압축 경계 이전 메시지는 API 요청에서 제외된다.

---

## 압축 프롬프트 구조

### BASE_COMPACT_PROMPT (prompt.ts:61-143줄)

처음부터 현재까지 전체 대화를 요약. 포함 섹션:

1. 원래 요청/의도
2. 탐색한 기술 개념
3. 관련 파일/코드 범위
4. 발견한 에러와 해결책
5. 문제 해결 과정
6. 주요 사용자 메시지
7. 대기 중인 작업
8. 현재 작업 상태
9. 다음 단계

### PARTIAL_COMPACT_PROMPT (prompt.ts:145줄~)

이전 압축 경계 이후 최근 메시지만 요약.  
이전 컨텍스트(요약)는 보존된 채로 새 요약을 앞에 붙인다.

---

## 압축 후 복원 상수

```typescript
// src/services/compact/compact.ts:121-130
POST_COMPACT_MAX_FILES_TO_RESTORE = 5       // 복원할 최대 파일 수
POST_COMPACT_TOKEN_BUDGET = 50_000          // 복원에 쓸 최대 토큰
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000    // 파일당 최대 토큰
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000   // 스킬당 최대 토큰
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000   // 스킬 복원 총 예산
MAX_COMPACT_STREAMING_RETRIES = 2           // 압축 실패 시 재시도
```

---

## Context Collapse (marble-origami)

더 세밀한 범위별 압축 메커니즘. 특정 메시지 범위를 선택적으로 축소한다.

```typescript
// src/types/logs.ts:255-295
ContextCollapseCommitEntry {
  type: 'marble-origami-commit',
  collapseId: string,              // 16자리 식별자
  summaryUuid: string,             // 플레이스홀더 UUID
  summaryContent: string,          // <collapsed> XML
  summary: string,                 // 평문 요약
  firstArchivedUuid: string,       // 압축 범위 시작
  lastArchivedUuid: string,        // 압축 범위 끝
}

ContextCollapseSnapshotEntry {
  staged: [{ startUuid, endUuid, summary, risk, stagedAt }],
  armed: boolean,                  // 다음 스폰 시 트리거 여부
  lastSpawnTokens: number,
}
```
