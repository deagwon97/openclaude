# 대화 히스토리 관리

핵심 파일: [src/history.ts](../../src/history.ts)

## 개요

히스토리는 `~/.claude/history.jsonl`에 저장되는 **전역 공유 파일**이다.  
대화 내용 자체가 아니라 "어떤 프롬프트를 입력했는가"를 저장하며,  
Ctrl+R 검색과 ↑ 화살표 탐색에 사용된다.

---

## 데이터 구조

### LogEntry

```typescript
// src/history.ts:219-225
type LogEntry = {
  display: string                                      // 표시 텍스트
  pastedContents: Record<number, StoredPastedContent>  // 붙여넣은 콘텐츠
  timestamp: number                                    // Unix 타임스탬프
  project: string                                      // 프로젝트 경로
  sessionId?: string                                   // 세션 ID
}
```

### StoredPastedContent

```typescript
// src/history.ts:25-32
type StoredPastedContent = {
  id: number
  type: 'text' | 'image'
  content?: string        // 1KB 이하 콘텐츠는 인라인 저장
  contentHash?: string    // 1KB 초과 콘텐츠는 해시 참조 (별도 파일)
  mediaType?: string
  filename?: string
}
```

---

## 쓰기 흐름

### Pending Buffer 시스템

```typescript
// src/history.ts:281-289
const pendingEntries: LogEntry[] = []         // 아직 디스크에 쓰지 않은 항목들
let isWriting = false
let currentFlushPromise: Promise<void> | null = null
let lastAddedEntry: LogEntry | null = null    // 마지막 항목 (undo 용)
const skippedTimestamps = new Set<number>()   // 이미 플러시됐지만 제거할 항목
```

메모리 버퍼에 쌓은 뒤, 비동기로 디스크에 플러시하는 방식이다.

### 플러시 방식

| 함수 | 특징 |
|------|------|
| `immediateFlushHistory()` (292줄) | 즉시 디스크 쓰기 (JSONL append) |
| `flushPromptHistory()` (329줄) | 비동기, 최대 5회 재시도, lock 기반 |

- **Lock 설정**: `stale: 10000ms`, 재시도 3회
- **파일 권한**: `0o600` (소유자만 읽기/쓰기)
- **형식**: JSONL (줄마다 JSON 객체)

### 엔트리 추가

```
addToPromptHistory() 호출
  ↓
콘텐츠 크기 판단
  - 텍스트 > 1KB → 별도 파일 저장 + contentHash 참조
  - 텍스트 ≤ 1KB → content에 인라인 저장
  - 이미지 → 이미지 캐시에서 별도 관리
  ↓
pendingEntries 배열에 push
  ↓
flushPromptHistory() 스케줄링 (fire-and-forget)
```

### 엔트리 제거 (Undo)

```typescript
// src/history.ts:453-464
export function removeLastFromHistory(): void {
  // 빠른 경로: 버퍼에 있으면 pop
  if (pendingEntries.length > 0) {
    pendingEntries.pop()
    return
  }
  // 느린 경로: 이미 플러시됐으면 스킵 세트에 타임스탬프 추가
  if (lastAddedEntry) {
    skippedTimestamps.add(lastAddedEntry.timestamp)
  }
}
```

---

## 읽기 흐름

### 기본 역순 읽기 (`makeLogEntryReader`, 106줄)

1. 메모리 버퍼(pendingEntries) 먼저 역순으로 순회
2. 디스크 파일을 청크 단위로 역순 읽기
3. `skippedTimestamps`에 있는 항목은 건너뜀

성능 최적화: 가장 최근 히스토리부터 접근하므로 전체 파일을 읽지 않는다.

### 용도별 읽기

| 함수 | 용도 | 특징 |
|------|------|------|
| `getTimestampedHistory()` (162줄) | Ctrl+R 검색 | 최신순, 중복 제거, 최대 100개, lazy 콘텐츠 로드 |
| `getHistory()` (190줄) | ↑ 화살표 탐색 | 현재 세션 항목 우선, 타 세션 후순 |

---

## 동시성 처리

여러 Claude 인스턴스가 동시에 실행될 수 있으므로:
- **File lock**으로 쓰기 경합 방지
- 읽기 시에는 lock 없이 메모리 버퍼 먼저 조회
- `currentFlushPromise`로 중복 플러시 방지
