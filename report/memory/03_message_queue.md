# 메시지 큐 및 우선순위 관리

핵심 파일:
- [src/utils/messageQueueManager.ts](../../src/utils/messageQueueManager.ts)
- [src/context/QueuedMessageContext.tsx](../../src/context/QueuedMessageContext.tsx)
- [src/types/textInputTypes.ts](../../src/types/textInputTypes.ts)

---

## 개요

사용자 입력 또는 외부 이벤트(브리지, 서브에이전트 등)로 발생한 명령들을  
우선순위 기반 큐에서 순서대로 처리한다.  
React의 `useSyncExternalStore`와 통합되어 UI와 상태가 동기화된다.

---

## 큐 데이터 구조

### QueuedCommand

```typescript
// src/types/textInputTypes.ts:301-360
type QueuedCommand = {
  value: string | Array<ContentBlockParam>  // 입력 값
  mode: PromptInputMode                     // bash / prompt / permission / notification
  priority?: QueuePriority                  // 우선순위
  uuid?: UUID
  orphanedPermission?: OrphanedPermission
  pastedContents?: Record<number, PastedContent>  // 붙여넣은 콘텐츠
  preExpansionValue?: string                // 슬래시 명령 확장 전 원본
  skipSlashCommands?: boolean               // 슬래시 명령 무시
  bridgeOrigin?: boolean                    // 브리지(원격 제어)에서 온 명령
  isMeta?: boolean                          // 메타 메시지 여부
  origin?: MessageOrigin                    // 출처 (외부 시스템)
  workload?: string                         // 워크로드 태그
  agentId?: AgentId                         // 서브에이전트 ID
}
```

### 내부 큐 상태

```typescript
// src/utils/messageQueueManager.ts:53-60
const commandQueue: QueuedCommand[] = []           // 실제 큐 (mutable)
let snapshot: readonly QueuedCommand[] = Object.freeze([])  // 불변 스냅샷 (React용)
const queueChanged = createSignal()                // 구독자 알림 신호
```

---

## 우선순위 시스템

```typescript
// src/types/textInputTypes.ts:278-296
type QueuePriority = 'now' | 'next' | 'later'
```

| 우선순위 | 숫자값 | 의미 |
|---------|--------|------|
| `'now'` | 0 | 즉시 중단 후 전송. 현재 tool 실행 중단 + ESC 후 전송 |
| `'next'` | 1 | 현재 tool 완료 후, tool 결과 반환 전에 전송 |
| `'later'` | 2 | 현재 턴 완전히 종료 후 새 쿼리로 처리 |

같은 우선순위 내에서는 FIFO 순서를 따른다.

---

## 큐 연산

### 추가 (Enqueue)

```typescript
// messageQueueManager.ts:201줄 — 기본 우선순위 'next'
export function enqueue(command: QueuedCommand): void {
  commandQueue.push({ ...command, priority: command.priority ?? 'next' })
  notifySubscribers()
}

// messageQueueManager.ts:142-149줄 — 기본 우선순위 'later'
export function enqueuePendingNotification(command: QueuedCommand): void {
  commandQueue.push({ ...command, priority: command.priority ?? 'later' })
  notifySubscribers()
}
```

### 제거 (Dequeue)

```typescript
// messageQueueManager.ts:167-193줄
export function dequeue(filter?: (cmd: QueuedCommand) => boolean): QueuedCommand | undefined {
  // 큐 전체 순회하여 최고 우선순위 항목 탐색
  let bestIdx = -1
  let bestPriority = Infinity
  for (let i = 0; i < commandQueue.length; i++) {
    const cmd = commandQueue[i]!
    if (filter && !filter(cmd)) continue
    const priority = PRIORITY_ORDER[cmd.priority ?? 'next']  // 숫자 변환
    if (priority < bestPriority) {
      bestIdx = i
      bestPriority = priority
    }
  }
  if (bestIdx === -1) return undefined
  const [dequeued] = commandQueue.splice(bestIdx, 1)
  notifySubscribers()
  return dequeued
}
```

`filter` 함수를 넘기면 조건에 맞는 항목 중 최고 우선순위를 꺼낸다.

---

## React 통합

```typescript
// messageQueueManager.ts
export const subscribeToCommandQueue = queueChanged.subscribe
export function getCommandQueueSnapshot(): readonly QueuedCommand[] {
  return snapshot
}
```

컴포넌트에서 `useSyncExternalStore(subscribeToCommandQueue, getCommandQueueSnapshot)`로  
큐 상태를 구독하면, 큐 변경 시 자동으로 리렌더링된다.

---

## 큐 연산 로깅

```typescript
// messageQueueManager.ts:28-38
type QueueOperationMessage = {
  type: 'queue-operation',
  operation: 'enqueue' | 'dequeue' | 'remove',
  timestamp: ISO 문자열,
  sessionId: string,
  content?: string,    // 문자열 명령인 경우만 기록
}
```

모든 enqueue/dequeue/remove 연산은 세션 스토리지에 기록된다.  
디버깅 및 감사(audit) 용도.

---

## 큐 사용 흐름 예시

```
사용자가 tool 실행 중에 새 메시지 입력
  → enqueue({ value: '..', priority: 'later' })
  
현재 tool 완료
  → dequeue() 호출 → priority='later' 항목 꺼냄
  → 새 턴으로 처리

ESC 누름 + 새 메시지 입력
  → enqueue({ value: '..', priority: 'now' })
  → 현재 tool 실행 중단
  → dequeue() → priority='now' 항목 먼저 꺼냄
```
