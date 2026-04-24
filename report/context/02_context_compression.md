# 02. 컨텍스트 압축 (Auto-Compact)

## 관련 파일

- `src/services/compact/compact.ts` — 압축 실행 로직
- `src/services/compact/autoCompact.ts` — 자동 압축 트리거 조건
- `src/utils/tokens.ts` — 토큰 카운팅

## 자동 압축 활성화 조건

```typescript
// autoCompact.ts
isAutoCompactEnabled():
  - DISABLE_COMPACT env      → 모든 압축 비활성화
  - DISABLE_AUTO_COMPACT env → 자동만 비활성화 (수동 /compact 유지)
  - autoCompactEnabled 설정  → 사용자 설정값
```

### 토큰 임계값

```
모델 컨텍스트 윈도우
    │
    ├── ~85% → Auto-Compact 트리거 (getAutoCompactThreshold)
    │
    ├── WARNING_THRESHOLD  → 경고 메시지 표시
    │
    └── ERROR_THRESHOLD    → 입력 블로킹

AUTOCOMPACT_BUFFER_TOKENS = 13,000   (여유분)
MANUAL_COMPACT_BUFFER_TOKENS         (수동 시 더 공격적)
```

## 압축 프로세스

### 1단계: 이미지 제거

```typescript
stripImagesFromMessages(messages)
// image/document 블록을 "[image]" 텍스트 마커로 교체
// 목적: 요약 API 호출 시 토큰 절약
```

### 2단계: 요약 API 호출

가장 오래된 메시지들을 선택하여 Claude API로 요약 요청:

```
선택된 오래된 메시지 묶음
    │
    ▼
요약 프롬프트 + 메시지 전송 → Claude API
    │
    ▼
summaryMessages: UserMessage[]  (AI 생성 요약)
```

### 3단계: 압축 결과 구조

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage      // "[Conversation compacted]" 경계 표시
  summaryMessages: UserMessage[]     // AI가 생성한 요약 메시지
  attachments: AttachmentMessage[]   // 스킬, 메모리 등 재주입 항목
  hookResults: HookResultMessage[]   // Hook 실행 결과
  messagesToKeep?: Message[]         // 유지할 최근 메시지들
}
```

### 4단계: 메시지 재조립

```typescript
buildPostCompactMessages():
// 최종 메시지 배열 순서:
[
  boundaryMarker,    // 압축 경계
  summaryMessages,   // 요약 내용
  messagesToKeep,    // 최근 메시지 (유지)
  attachments,       // 스킬/메모리 재주입
  hookResults        // Hook 결과
]
```

**재주입 제한:**
- 파일 재주입: 최대 5개 파일, 파일당 최대 5K 토큰, 총 50K 토큰 예산
- 스킬 재주입: 최대 25K 토큰 예산

## Prompt-Too-Long 복구

압축 자체가 PTL(Prompt-Too-Long) 오류를 유발하는 극단적 상황 처리:

```typescript
truncateHeadForPTLRetry(messages)
// 가장 오래된 API 라운드 그룹의 20%를 삭제 후 재시도
```

## 토큰 추적

```typescript
// tokens.ts
tokenCountWithEstimation(messages)    // 메시지 배열 토큰 수 추정
finalContextTokensFromLastResponse()  // API 응답의 usage에서 실제 토큰 수 추출

// usage 분리 추적
cache_read_input_tokens       // 프롬프트 캐시 히트 (비용 절감)
cache_creation_input_tokens   // 캐시 쓰기
```

## 수동 압축

사용자가 `/compact` 명령으로 직접 트리거 가능.
자동 압축과 동일한 프로세스를 사용하지만 더 공격적인 토큰 임계값 적용.
