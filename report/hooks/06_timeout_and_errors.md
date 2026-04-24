# 06. 타임아웃 · 에러 · 비동기 Hook

## 1. 타임아웃 위계

Hook 타입과 이벤트에 따라 기본 타임아웃이 다르다.

| 대상 | 기본값 | 오버라이드 |
|------|--------|-----------|
| command hook | `TOOL_HOOK_EXECUTION_TIMEOUT_MS` = 10 분 | `hook.timeout` (초) |
| prompt hook | 30 초 | `hook.timeout` (초) |
| agent hook | 60 초 | `hook.timeout` (초) |
| http hook | 10 분 | `hook.timeout` (초) |
| **SessionEnd** (특수) | `SESSION_END_HOOK_TIMEOUT_MS_DEFAULT` = 1500 ms | 환경변수 `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` |

`SessionEnd`만 유독 짧은 이유는 종료 지연을 사용자가 체감하기 때문이다. 긴 정리 작업은 `async` hook으로 백그라운드로 돌려야 한다.

## 2. AbortSignal 합성: `createCombinedAbortSignal()`

hook 실행은 **두 가지** 이유로 중단될 수 있다.

1. 부모 컨텍스트 취소 (사용자가 Ctrl+C, 상위 턴 취소 등)
2. 자체 타임아웃

이를 하나의 `AbortSignal`로 합치는 헬퍼가 `createCombinedAbortSignal(parentSignal, { timeoutMs })`이다.

```typescript
const { signal, cleanup } = createCombinedAbortSignal(parentSignal, {
  timeoutMs: commandTimeoutMs,
})

try {
  const result = await spawnWithSignal(cmd, { signal })
  if (result.aborted) {
    // 부모 or 타임아웃에 의해 취소됨
  }
} finally {
  cleanup()   // 타이머 해제
}
```

- 부모가 먼저 abort되면 즉시 자식 프로세스 종료
- 타임아웃이 먼저면 프로세스를 kill하고 stderr에 "Command timed out" 기록
- 어느 쪽이든 cleanup()에서 타이머/리스너 정리

## 3. 에러 분류

Hook 실행 중 발생한 예외는 이렇게 분류된다.

| 상황 | 처리 |
|------|------|
| Trust 검증 실패 | 조용히 스킵, `diag_log`에만 기록 |
| Plugin 디렉토리 없음 | `non_blocking_error`로 변환, 사용자에게만 표시 |
| shell spawn 실패 (ENOENT 등) | `non_blocking_error`, errno 메시지 수록 |
| EPIPE (stdin 조기 종료) | `non_blocking_error` |
| AbortError (취소) | `cancelled` outcome |
| 타임아웃 | kill + stderr "Command timed out" + `non_blocking_error` (exit code ≠ 2인 경우) |
| Exit 2 | `blocking` outcome, stderr을 모델에 표시 |
| JSON 파싱 실패 | plain text 폴백, `outcome`은 exit code 기준 |
| JSON 스키마 불일치 | validation error 메시지 주입 |
| HTTP allowlist 실패 | 조용히 `{ok:false}`로 처리 |
| HTTP 네트워크 에러 | `non_blocking_error`, statusCode/메시지 수록 |

`blockingError`는 **첫 번째**로 발견된 것만 유지되고, 그 이후에 blocking으로 끝난 hook들은 `additionalContexts`로 흡수된다.

## 4. 비동기 Hook

### 4.1 `async: true`

Hook이 실행 즉시 결과를 반환하지 않고 백그라운드로 넘어간다.

```
1. spawn 직후 첫 stdout 라인 파싱 → {"async": true, "processId": "..."}
2. shellCommand.background(processId) 호출
3. AsyncHookRegistry에 pending 등록
4. executeHooks() 는 해당 hook에 대해 즉시 success 반환
5. 실제 작업은 백그라운드에서 계속
6. 완료 시 결과는 버려지거나 (asyncRewake 없음) 다음 턴 주입 큐로 전달
```

파일: [src/utils/hooks/AsyncHookRegistry.ts](../../src/utils/hooks/AsyncHookRegistry.ts)

### 4.2 `asyncRewake: true`

백그라운드 실행하되 **exit code 2**가 나오면 모델을 다시 깨운다.

```typescript
void shellCommand.result.then(async (result) => {
  const stderr = shellCommand.taskOutput.getStderr()
  shellCommand.cleanup()

  if (result.code === 2) {
    enqueuePendingNotification({
      value: wrapInSystemReminder(`Stop hook blocking error: ${stderr}`),
      mode: 'task-notification',
    })
  }
})
```

용도: Stop hook에서 "턴이 끝났지만 이 조건이 충족되지 않았으니 한 번 더 실행해" 같은 재기동 루프.

## 5. 진단 로깅

일회성 이벤트(`SessionStart`, `SessionEnd`, `Setup`)는 `diag_log`에 spawn/완료 이벤트가 남는다.

```json
{
  "hook_spawn_started":   { "hook_event_name", "index" },
  "hook_spawn_completed": { "hook_event_name", "index",
                             "duration_ms", "exit_code", "aborted" }
}
```

`CLAUDE_CODE_DEBUG_VERBOSE` 환경변수를 켜면:

- 후보 hook 선정 결과
- matcher / if 조건 매칭 상태
- command 실행과 타임아웃 설정
- JSON 파싱 성공/실패

가 debug log로 출력된다.

## 6. 성능 이슈와 완화책

| 이슈 | 완화책 |
|------|--------|
| 이벤트마다 설정 파일 파싱 | **스냅샷** (`captureHooksConfigSnapshot`) 한 번만 파싱 |
| tree-sitter 파싱 비용 | `prepareIfConditionMatcher`로 이벤트당 1회 준비 |
| SDK 내부 콜백 hook 오버헤드 | **Fast path** – span/abort/JSON 처리 스킵 (6µs → 1.8µs) |
| 여러 hook 직렬 실행 지연 | 병렬 spawn |
| 잘못된 hook이 프로세스를 오래 붙잡음 | 타입별 기본 타임아웃 + `async` 플래그 |
| hook이 stdout을 폭주시킴 | TaskOutput 스트리밍 수집 + `suppressOutput` 지원 |

## 7. 정리: "언제 무엇을 쓸까"

| 목적 | 추천 타입 / 옵션 |
|------|------------------|
| 간단한 차단 규칙 (e.g., `rm -rf` 금지) | `command` + `if` + exit 2 |
| LLM 기반 간단 검증 | `prompt` |
| 테스트/lint 등 긴 작업 | `command` + `async: true` |
| Stop에서 "조건 충족 안 되면 재실행" | `command` + `asyncRewake: true` |
| 외부 감사 시스템 | `http` + allowlist URL |
| 여러 tool을 호출해 상세 검증 | `agent` |
| Tool 입력 정규화 | `command` (PreToolUse) + `updatedInput` |
| 응답 컨텍스트 추가 | `command`/`prompt` (PostToolUse) + `additionalContext` |
