# 01. Hook 이벤트와 타입

## 1. 이벤트 목록 (27개)

`HOOK_EVENTS` 상수는 [src/entrypoints/sdk/coreTypes.ts:25](../../src/entrypoints/sdk/coreTypes.ts#L25)에 정의되어 있다.

| 이벤트 | 시점 | 입력 payload 핵심 필드 | 대표 호출 위치 |
|--------|------|----------------------|----------------|
| `PreToolUse` | Tool 실행 직전 | `tool_name`, `tool_input`, `tool_use_id` | [src/services/tools/toolHooks.ts](../../src/services/tools/toolHooks.ts) (`runPreToolUseHooks`) |
| `PostToolUse` | Tool 실행 직후 | `tool_name`, `tool_input`, `tool_response` | [src/services/tools/toolHooks.ts](../../src/services/tools/toolHooks.ts) (`runPostToolUseHooks`) |
| `PostToolUseFailure` | Tool 실행 실패 | `tool_name`, `error`, `is_interrupt` | [src/services/tools/toolHooks.ts](../../src/services/tools/toolHooks.ts) (`runPostToolUseFailureHooks`) |
| `UserPromptSubmit` | 사용자 입력 제출 직전 | `prompt`, `session_id` | [src/utils/processUserInput/processUserInput.ts](../../src/utils/processUserInput/processUserInput.ts) |
| `SessionStart` | 세션 시작 | `source: startup\|resume\|clear\|compact` | [src/utils/hooks.ts](../../src/utils/hooks.ts) |
| `SessionEnd` | 세션 종료 | `reason: clear\|logout\|prompt_input_exit\|…` | 동상 |
| `Stop` | 턴 종료 직전 | 기본 payload만 | 동상 |
| `StopFailure` | API 에러로 턴 종료 | `error`, `error_details` | 동상 |
| `SubagentStart` / `SubagentStop` | 서브에이전트 생애주기 | `agent_id`, `agent_type`, `agent_transcript_path` | 동상 |
| `PreCompact` / `PostCompact` | 컨텍스트 압축 전/후 | `trigger: manual\|auto`, `custom_instructions`, `summary` | 동상 |
| `Notification` | 사용자 알림 | `message`, `notification_type` | 동상 |
| `PermissionRequest` | 권한 대화 표시 | `tool_name`, `tool_input` | 권한 시스템 |
| `PermissionDenied` | 분류기 거부 | `tool_name`, `reason` | 권한 시스템 |
| `Setup` | 저장소 초기화/유지 | `trigger: init\|maintenance` | 동상 |
| `Elicitation` / `ElicitationResult` | MCP 입력 요청/응답 | `mcp_server_name`, `message`, `content` | MCP 통합 |
| `ConfigChange` | 설정 변경 | `source`, `changed_keys` | 설정 시스템 |
| `WorktreeCreate` / `WorktreeRemove` | worktree 생애주기 | `worktree_path` | worktree 서비스 |
| `InstructionsLoaded` | CLAUDE.md/명령 파일 로드 | `instructions_path` | 동상 |
| `CwdChanged` | 작업 디렉토리 변경 | `cwd` | 환경 훅 |
| `FileChanged` | 파일 변경 감지 | `file_path`, `change_type` | [src/utils/hooks/fileChangedWatcher.ts](../../src/utils/hooks/fileChangedWatcher.ts) |
| `TeammateIdle` / `TaskCreated` / `TaskCompleted` | 팀 협업 | 팀 협업 | 팀 협업 시스템 |

> 이벤트 메타데이터(설명·matcher 필드·once-per-session 여부)는 [src/utils/hooks/hooksConfigManager.ts](../../src/utils/hooks/hooksConfigManager.ts)의 `getHookEventMetadata()`에서 조회한다.

## 2. Hook 타입 (4종)

Zod 스키마는 [src/schemas/hooks.ts](../../src/schemas/hooks.ts)에 정의되어 있다. 모든 타입은 공통 필드 `matcher`, `if`, `timeout`, `async`, `asyncRewake`를 선택적으로 가질 수 있다.

### 2.1 command (가장 기본)

```json
{
  "type": "command",
  "command": "scripts/check.sh",
  "shell": "bash",          // 기본 bash, Windows에서 powershell 선택 가능
  "timeout": 30,             // 초 단위
  "if": "Bash(git *)"
}
```

- `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA` 환경변수 주입
- stdin으로 hook input JSON 전달, stdout/stderr을 캡처
- Windows는 POSIX 경로 변환(`toHookPath`) 수행

### 2.2 prompt (LLM 기반 가벼운 검증)

```json
{
  "type": "prompt",
  "prompt": "이 tool call이 secrets를 유출할 가능성이 있으면 block 하라.",
  "model": "claude-opus-4",
  "timeout": 30
}
```

- [src/utils/hooks/execPromptHook.ts](../../src/utils/hooks/execPromptHook.ts)
- LLM에 단일 쿼리, 반환 스키마는 `{ ok: boolean, reason?: string }`

### 2.3 agent (멀티턴 검증)

```json
{
  "type": "agent",
  "agentType": "verifier",
  "prompt": "트랜스크립트를 검토해 잘못된 주장을 찾아라.",
  "timeout": 60
}
```

- [src/utils/hooks/execAgentHook.ts](../../src/utils/hooks/execAgentHook.ts)
- `hook-agent-*` 에이전트 생성, `SyntheticOutputTool`로 구조화 출력 강제
- 트랜스크립트 읽기 권한 부여

### 2.4 http (외부 서비스 위임)

```json
{
  "type": "http",
  "url": "https://internal.example/hook",
  "headers": { "X-Auth": "$HOOK_TOKEN" },
  "timeout": 600
}
```

- [src/utils/hooks/execHttpHook.ts](../../src/utils/hooks/execHttpHook.ts) + [ssrfGuard.ts](../../src/utils/hooks/ssrfGuard.ts)
- URL allowlist 검증(`allowedHttpHookUrls`), `$VAR_NAME` 보간은 `allowedEnvVars`만
- 헤더 CRLF 주입 방어
- POST 본문은 hook 입력 JSON, 응답은 일반 hook 출력 스키마와 동일

## 3. settings.json 구조 예시

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "scripts/guard.sh", "if": "Bash(rm *)" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "prompt", "prompt": "수정된 파일이 린트를 통과하는지 확인" }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "scripts/redact-secrets.sh" }
        ]
      }
    ]
  }
}
```

각 이벤트는 `{matcher, hooks[]}` 배열을 가지며, 한 matcher 엔트리 안의 hook들은 병렬 실행된다.
