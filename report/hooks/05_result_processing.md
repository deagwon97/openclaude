# 05. 결과 처리

## 1. 두 가지 결과 채널

모든 hook은 두 채널로 결과를 돌려준다.

1. **Exit code** (또는 HTTP status) — blocking 여부 결정
2. **JSON 출력** (선택) — 세부 결정과 payload 수정

두 채널을 조합하면 동일한 의미를 여러 방식으로 표현할 수 있다. 예: "tool 차단"은 `exit 2 + stderr 메시지`로도, `exit 0 + JSON {"decision": "block"}`으로도 가능하다.

## 2. Exit code 해석 표

| Exit code | 분류 | 동작 |
|-----------|------|------|
| `0` | success | JSON 파싱 시도, 실패하면 stdout은 무시, suppress |
| `2` | **explicit blocking** | stderr을 모델에 표시, `blockingError` 생성 |
| 그 외 | non-blocking error | stderr은 사용자에게만 표시, 모델은 영향 없음 |

핵심 구현: `processHookJSONOutput()` + exit code 분기 (in [src/utils/hooks.ts](../../src/utils/hooks.ts)).

## 3. JSON 출력 공통 스키마

```typescript
type HookJSONOutput = {
  continue?: boolean               // false → 전체 대화 계속 금지
  stopReason?: string              // continue:false 시 사유
  decision?: 'approve' | 'block'   // 구형식 권한 결정 (PreToolUse 호환용)
  reason?: string                  // decision 관련 사유
  suppressOutput?: boolean         // stdout/stderr 표시 안 함
  systemMessage?: string           // 사용자에게 시스템 메시지
  permissionDecision?: 'allow' | 'deny' | 'ask'
  hookSpecificOutput?: HookSpecificOutput
}
```

정의 위치: [src/entrypoints/sdk/coreSchemas.ts](../../src/entrypoints/sdk/coreSchemas.ts).

## 4. 이벤트별 `hookSpecificOutput`

### 4.1 PreToolUse

```typescript
{
  hookEventName: 'PreToolUse',
  permissionDecision?: 'allow' | 'deny' | 'ask',
  permissionDecisionReason?: string,
  updatedInput?: Record<string, unknown>   // tool 호출 시 사용할 새 input
}
```

**효과**
- `permissionDecision: 'deny'` → tool call 차단, reason을 stderr로 모델에 전달
- `permissionDecision: 'allow'` → 권한 대화를 건너뛰고 바로 실행 (automated approval)
- `updatedInput` → 이후 실행에 사용될 tool input이 교체됨 (e.g., 경로 정규화, secret redaction)

### 4.2 PostToolUse

```typescript
{
  hookEventName: 'PostToolUse',
  additionalContext?: string,          // 모델에 hook_additional_context attachment로 전달
  updatedMCPToolOutput?: unknown       // MCP tool 응답을 덮어쓰기
}
```

**효과**
- `additionalContext` → 다음 턴 시스템 메시지에 주입 (검증 결과, 린트 로그 등)
- `updatedMCPToolOutput` → `isMcpTool(tool)` 확인 후에만 적용. 일반 내장 tool에는 적용 불가

### 4.3 UserPromptSubmit

```typescript
{
  hookEventName: 'UserPromptSubmit',
  additionalContext?: string           // 원 프롬프트에 덧붙여 모델에 전달
}
```

차단이 필요하면 공통 필드(`decision: 'block'` 또는 exit 2)를 쓴다.

### 4.4 PermissionRequest

```typescript
{
  hookEventName: 'PermissionRequest',
  decision: {
    behavior: 'allow' | 'deny',
    updatedInput?: Record<string, unknown>
  }
}
```

권한 대화가 표시되기 직전에 hook이 결정을 대신 내려주는 용도.

### 4.5 Elicitation (MCP)

```typescript
{
  hookEventName: 'Elicitation',
  action: 'accept' | 'decline' | 'cancel',
  content?: unknown                    // accept 시 사용자가 제공한 값
}
```

MCP 서버가 구조화 입력을 요구할 때 hook이 자동 응답하도록 허용.

## 5. 결과 주입 경로

```
hook 결과
   ├── blockingError
   │      ├── PreToolUse → tool 호출 즉시 실패, hook_blocking_error attachment를 모델에 전달
   │      ├── UserPromptSubmit → 원 프롬프트 폐기, 사용자에게 stderr 표시
   │      └── Stop → 모델 응답 재실행 유도
   │
   ├── additionalContext / additionalContexts[]
   │      └── hook_additional_context attachment → 모델 메시지에 추가
   │
   ├── updatedInput (PreToolUse)
   │      └── tool executor가 사용 (toolHooks.ts)
   │
   ├── updatedMCPToolOutput (PostToolUse)
   │      └── MCP tool 응답 교체
   │
   ├── permissionBehavior
   │      └── 권한 시스템에 주입 (allow/deny/ask)
   │
   ├── preventContinuation / stopReason
   │      └── 현재 턴 종료, 모델이 다음 호출을 하지 않음
   │
   └── systemMessage / systemMessages[]
          └── 사용자 화면에 시스템 공지로 표시
```

## 6. Attachment 종류

Hook 결과를 메시지로 포장할 때 사용되는 attachment 태그:

| Attachment | 사용처 |
|------------|--------|
| `hook_success` | 정상 완료된 hook의 JSON/plain 출력 |
| `hook_blocking_error` | exit 2 또는 decision:block |
| `hook_additional_context` | additionalContext 주입 |
| `hook_non_blocking_error` | 기타 비정상 종료 (사용자 전용) |
| `hook_cancelled` | AbortSignal/타임아웃으로 취소됨 |

## 7. 다중 hook 집계 규칙

여러 hook이 한 이벤트에 걸린 경우:

```
blockingError       ← 첫 번째로 발견된 것만 유지
additionalContexts  ← 모두 배열로 누적
updatedInput        ← 마지막에 반환된 값으로 덮어쓰기 (주의: 순서 의존)
updatedMCPToolOutput← 마지막에 반환된 값으로 덮어쓰기
permissionBehavior  ← deny 우선, 다음 allow, 그 외 ask
preventContinuation ← OR 집계
systemMessages      ← 모두 누적
```

`updatedInput`/`updatedMCPToolOutput`이 여러 hook에서 나오면 순서 의존성이 발생하므로, 현실적으로는 같은 matcher에 하나의 수정 hook만 걸어두는 게 안전하다.

## 8. 예시: PreToolUse에서 Bash 입력 재작성

```bash
#!/usr/bin/env bash
# 입력 JSON을 읽어 command에서 'sudo' 제거
input="$(cat)"
cmd=$(jq -r '.tool_input.command' <<< "$input")
rewritten=${cmd/sudo /}
jq -n --arg cmd "$rewritten" '{
  hookSpecificOutput: {
    hookEventName: "PreToolUse",
    updatedInput: { command: $cmd }
  }
}'
```

이 hook은 `Bash` tool 호출의 `command` 필드를 안전한 형태로 교체해 모델이 모르게 sudo를 벗겨버린다. Tool executor는 updatedInput으로 실제 실행을 진행한다.
