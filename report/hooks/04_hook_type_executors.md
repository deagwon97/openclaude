# 04. Hook 타입별 Executor

네 가지 hook 타입은 각각 별도의 executor로 실행된다.

## 1. command hook — `execCommandHook()`

구현: [src/utils/hooks.ts](../../src/utils/hooks.ts) (`execCommandHook`).

### 1.1 실행 단계

```
1. Shell 선택
   ├── hook.shell === 'powershell' → PowerShell
   └── 기본 → bash (POSIX)
       └── Windows는 toHookPath()로 경로 변환

2. 환경변수 주입
   ├── CLAUDE_PROJECT_DIR   = 현재 repo 루트
   ├── CLAUDE_PLUGIN_ROOT   = hook의 소속 플러그인 디렉토리
   ├── CLAUDE_PLUGIN_DATA   = 플러그인 데이터 디렉토리
   └── (부모 프로세스 env 그대로 상속)

3. Spawn + stdin 전달
   └── child.stdin.write(JSON.stringify(hookInput))

4. Stream capture (TaskOutput)
   ├── stdout, stderr 누적
   └── 첫 stdout 라인 파싱 → {"async": true} 감지

5. 종료 대기 (combined abortSignal + timeout)

6. 결과 해석
   ├── JSON 파싱 시도 → processHookJSONOutput()
   └── 실패 시 plain text로 폴백
```

### 1.2 stdin 요청 (interactive prompt)

Hook이 stdout에 `{"prompt": "..."}` JSON 라인을 출력하면 호스트가 사용자에게 입력을 요청하고 그 응답을 hook의 stdin으로 되돌려보낸다. 대화형 확인을 요구하는 hook을 지원하기 위한 기능이다.

### 1.3 비동기 플래그

```json
{ "type": "command", "command": "long-lint.sh", "async": true }
```

- 첫 stdout 라인에 `{"async": true, "processId": "..."}` 패턴이 감지되면 즉시 백그라운드로 이전
- `AsyncHookRegistry`에 등록되어 이후 결과는 푸시 알림으로 처리

## 2. prompt hook — `execPromptHook()`

구현: [src/utils/hooks/execPromptHook.ts](../../src/utils/hooks/execPromptHook.ts)

```
1. LLM 선택 (hook.model || 기본: claude-opus-4)
2. 시스템 프롬프트: "너는 hook 검증자. JSON으로만 응답하라."
3. 사용자 프롬프트: hook.prompt + hook 입력 payload
4. 응답 스키마:
   {
     "ok": boolean,
     "reason"?: string
   }
5. ok === false → blockingError 생성, reason을 stderr로 전달
6. ok === true  → plainText 결과로 통과
```

- 빠른 검증 용도 (수십 줄 이내 규칙)
- 기본 타임아웃 30초

## 3. agent hook — `execAgentHook()`

구현: [src/utils/hooks/execAgentHook.ts](../../src/utils/hooks/execAgentHook.ts)

```
1. hook.agentType에 해당하는 agent 정의 조회
2. agentId = "hook-agent-{uuid}"
3. 전용 에이전트 부팅
   ├── tools 풀: 에이전트 정의에 맞춰 제한
   ├── transcriptPath 접근 권한 부여
   └── SyntheticOutputTool(StructuredOutputTool) 강제
4. 멀티턴 실행 (hook.prompt를 초기 메시지로)
5. 종료 조건
   ├── SyntheticOutputTool 호출 → structured result 반환
   └── 타임아웃 or AbortSignal
6. 결과 파싱 → processHookJSONOutput()
```

- `prompt` hook보다 강력한 검증(여러 tool을 호출해 실제 파일/테스트를 확인)
- 기본 타임아웃 60초
- 에이전트 자체에서도 추가 hook이 걸릴 수 있음 → 재귀 방지를 위해 subagent 컨텍스트 플래그 체크

## 4. http hook — `execHttpHook()`

구현: [src/utils/hooks/execHttpHook.ts](../../src/utils/hooks/execHttpHook.ts) + [ssrfGuard.ts](../../src/utils/hooks/ssrfGuard.ts)

```
1. URL allowlist 검증 (ssrfGuard.ts)
   ├── policy.allowedHttpHookUrls 과 매칭
   └── 실패 → 조용히 {"ok": false} 반환

2. 헤더 처리
   ├── "$VAR_NAME" 보간은 policy.allowedEnvVars 에 포함된 것만
   ├── CRLF 주입 차단 (header value sanitize)
   └── Content-Type: application/json 강제

3. POST /
   └── body = hook 입력 JSON

4. 응답
   ├── JSON 파싱 → processHookJSONOutput()
   └── 실패 시 raw text를 stderr로 사용
```

- 기본 타임아웃 **10분** (외부 서비스 지연 고려)
- 응답 스키마는 command hook의 JSON 출력과 동일

## 5. 공통 후처리: `processHookJSONOutput()`

네 executor 모두 JSON 출력이 있으면 동일한 `processHookJSONOutput()`에 투입된다. 이 함수는 다음 필드를 해석해 `HookResult`로 변환한다.

```
{
  continue,          // false → preventContinuation
  stopReason,
  decision,          // "block" → permissionBehavior: 'deny'
  reason,
  suppressOutput,
  systemMessage,
  permissionDecision,
  hookSpecificOutput // 이벤트별 필드
}
```

자세한 해석은 [05_result_processing.md](05_result_processing.md)에서 다룬다.

## 6. 실행자별 비교

| 항목 | command | prompt | agent | http |
|------|---------|--------|-------|------|
| 외부 프로세스 | 예 (shell) | 아니오 (LLM API) | 아니오 (내부 agent) | 아니오 (HTTP) |
| 표현력 | ★★★ (shell 전체) | ★ (단일 응답) | ★★★ (멀티턴+도구) | ★★ (외부 서비스) |
| 지연 | 수 ms~분 | LLM 1콜 | LLM N콜 | 네트워크 RTT |
| 기본 타임아웃 | `TOOL_HOOK_EXECUTION_TIMEOUT_MS` (10분) | 30s | 60s | 10분 |
| 주요 리스크 | RCE | 잘못된 판단 | 비용/지연 | SSRF |
| 주요 안전장치 | trust 검증 | 모델 구속 프롬프트 | 도구 화이트리스트 | URL/env allowlist |
