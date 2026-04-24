# 06. 컨텍스트 주입 (`systemContext` / `userContext`)

시스템 프롬프트 자체의 구조는 [05_system_prompt.md](05_system_prompt.md) 에서 다뤘습니다. 이 문서는 **매 turn 변할 수 있는 동적 컨텍스트**가 언제, 어디에, 어떤 형식으로 들어가는지를 정리합니다.

## 관련 파일

- [src/context.ts](../../src/context.ts) — `getSystemContext()`, `getUserContext()` 수집
- [src/utils/api.ts](../../src/utils/api.ts) — `appendSystemContext`, `prependUserContext`
- [src/query.ts](../../src/query.ts) — 루프 안에서 두 함수를 부르는 지점

---

## 두 종류의 컨텍스트

OpenClaude 는 동적 컨텍스트를 두 채널로 분리합니다.

| 채널 | 최종 위치 | 예시 |
|------|-----------|------|
| `systemContext` | **system 프롬프트 배열 끝** | `gitStatus`, `currentDate`, `platform`, `cwd` |
| `userContext` | **첫 user 메시지 바로 앞**에 `<system-reminder>` 블록 | `CLAUDE.md`, `sessionMemory`, MCP resource hints |

두 채널을 나누는 이유:

- **캐시 친화성** — `systemContext` 는 이미 동적 섹션 뒤쪽에 들어가므로 cacheScope: null 영역에 그대로 합쳐지면 된다.
- **모델 포커싱** — `userContext` 는 user 메시지 앞에 `<system-reminder>` 로 들어가므로, "지금 이 대화의 지시"로 인식된다. CLAUDE.md 의 프로젝트 지침이 여기 속한다.

---

## 수집 단계 (`src/context.ts`)

수집 함수는 memoize 되어 있어서 같은 세션 안에서는 반복 호출되더라도 캐시된 결과를 돌려줍니다.

```typescript
getSystemContext() → memoized
// 수집 내용 (예)
//   currentBranch, defaultBranch, gitUser
//   gitStatus (최대 2K chars)
//   최근 5개 commit: "hash message"
//   cacheBreaker?: "[CACHE_BREAKER: ...]" (캐시 무효화용)

getUserContext() → memoized
// 수집 내용
//   "CLAUDE.md" → 프로젝트 지침 전체
//   "Today's date is YYYY-MM-DD."
//   sessionMemory summary (있는 경우)
```

`cacheBreaker` 는 사용자가 의도적으로 시스템 프롬프트 캐시를 무효화하고 싶을 때 주입되는 훅입니다 (예: prompt 변경 실험).

---

## 주입 지점 1 — `appendSystemContext`

시스템 프롬프트가 조립된 직후, `queryLoop` 의 2단계에서 바로 호출됩니다.

```typescript
// src/query.ts
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

구현은 아주 단순합니다. 배열 끝에 `key: value` 를 줄바꿈으로 이은 한 덩어리를 append 할 뿐입니다.

```typescript
// src/utils/api.ts
export function appendSystemContext(
  systemPrompt: SystemPrompt,
  context: { [k: string]: string },
): string[] {
  return [
    ...systemPrompt,
    Object.entries(context)
      .map(([key, value]) => `${key}: ${value}`)
      .join('\n'),
  ].filter(Boolean)
}
```

결과적으로 API 에 전달되는 `system` 블록의 **맨 마지막**에 다음과 같은 텍스트 덩어리가 붙습니다.

```
currentDate: 2026-04-12
gitStatus: On branch main
...
platform: linux
osVersion: Linux 6.8.0-49-generic
cwd: /data/ai-test/openclaude
```

경계 마커 이후에 들어가므로 이 블록은 `cacheScope: null` 쪽에 합쳐지고 캐시되지 않습니다.

---

## 주입 지점 2 — `prependUserContext`

`userContext` 는 **API 호출 직전**에 첫 user 메시지 앞에 삽입됩니다. 시스템 프롬프트가 아니라 메시지 배열에 들어가는 것이 핵심.

```typescript
// src/query.ts (callModel 호출부)
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  ...
})) { ... }
```

구현:

```typescript
// src/utils/api.ts
export function prependUserContext(
  messages: Message[],
  context: { [k: string]: string },
): Message[] {
  if (process.env.NODE_ENV === 'test') return messages
  if (Object.entries(context).length === 0) return messages

  return [
    createUserMessage({
      content:
        `<system-reminder>\n` +
        `As you answer the user's questions, you can use the following context:\n` +
        Object.entries(context)
          .map(([key, value]) => `# ${key}\n${value}`)
          .join('\n') +
        `\n\n      IMPORTANT: this context may or may not be relevant to your tasks. ` +
        `You should not respond to this context unless it is highly relevant to your task.\n` +
        `</system-reminder>\n`,
      isMeta: true,
    }),
    ...messages,
  ]
}
```

실제로 모델이 보게 되는 첫 user 메시지는 이런 모양이 됩니다.

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# CLAUDE.md
(...프로젝트 지침 전문...)
# currentDate
Today's date is 2026-04-12.

      IMPORTANT: this context may or may not be relevant to your tasks. You should not respond to this context unless it is highly relevant to your task.
</system-reminder>
```

- **`isMeta: true`** — UI 가 이 메시지를 대화 로그에 표시하지 않는다는 플래그. 모델에겐 보이지만 사용자에겐 보이지 않는다.
- **테스트 환경에서는 생략** — `NODE_ENV === 'test'` 일 때는 주입 자체를 스킵해 스냅샷 테스트를 단순화한다.

---

## 두 채널의 가시적 차이

API 페이로드를 의사 표현하면 대략 다음과 같습니다.

```jsonc
{
  "system": [
    { "type": "text", "text": "(intro + system + doingTasks + ...)", "cache_control": {...} },
    { "type": "text", "text": "(memory + env + mcp + ...)                 \n currentDate: 2026-04-12\n gitStatus: ..." }
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "<system-reminder>...CLAUDE.md...</system-reminder>" }
      ]
    },
    /* ...실제 대화 메시지들... */
  ]
}
```

요점:

1. **정적 system 블록** ← 프롬프트 캐시 적용
2. **동적 system 블록** ← 매 요청 새로 전송, `systemContext` 가 여기 합류
3. **첫 user 메시지 앞** ← `userContext` (`<system-reminder>`, `isMeta`)

---

## 왜 CLAUDE.md 가 system 쪽이 아니라 user 쪽에 있는가

직관적으로는 시스템 프롬프트 안에 넣고 싶지만, 의도적으로 user 메시지 쪽에 있습니다.

- **변경이 잦다** — 프로젝트 진행 중 CLAUDE.md 는 수시로 편집된다. 캐시 대상에 넣으면 캐시 무효화가 잦아진다.
- **프로젝트 로컬 맥락** — 프로젝트 전환 시마다 user context 전체가 바뀌므로, 메시지 쪽에서 처리하는 편이 세션/메시지 수준의 격리와 잘 맞는다.
- **"규칙"이 아니라 "지금 이 대화의 배경"** — `<system-reminder>` 로 모델에게 "참고용 컨텍스트이며 관련 없으면 언급하지 말라"고 명시적으로 지시할 수 있다.

---

## 다음 문서

- [07_autocompact.md](07_autocompact.md) — 컨텍스트가 커졌을 때 AutoCompact 가 루프에 어떻게 개입하는가
