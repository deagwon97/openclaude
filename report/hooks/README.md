# OpenClaude Hooks 시스템 분석

## 개요

OpenClaude의 hooks 시스템은 `settings.json`에 등록한 사용자 정의 커맨드를 **라이프사이클 이벤트**에 연결해 실행하는 확장 메커니즘이다. 총 **27개 이벤트**와 **4가지 hook 타입**(command / prompt / agent / http)을 지원한다.

| 특성 | 설명 |
|------|------|
| 실행 모델 | Async generator 스트리밍 + 병렬 실행 |
| 매칭 | `matcher`(문자열/파이프/정규식) + `if`(권한 규칙 문법) |
| 결과 | exit code + JSON 출력으로 모델 제어 (allow/deny, input 수정, context 주입) |
| 안전장치 | workspace trust 검증, URL allowlist, CRLF 주입 방어, 타임아웃 |
| 확장 | 비동기 hook(`async`, `asyncRewake`), 플러그인/스킬 네임스페이싱 |

## 목차

| 파일 | 내용 |
|------|------|
| [01_events_and_types.md](01_events_and_types.md) | 27개 hook 이벤트 목록과 호출 위치, 4가지 hook 타입 스키마 |
| [02_loading_and_matching.md](02_loading_and_matching.md) | settings 로드, 스냅샷, matcher/if 필터링, 중복 제거 |
| [03_execution_engine.md](03_execution_engine.md) | `executeHooks()` 파이프라인과 병렬 실행 흐름 |
| [04_hook_type_executors.md](04_hook_type_executors.md) | command / prompt / agent / http 각 executor 상세 |
| [05_result_processing.md](05_result_processing.md) | JSON 출력 스키마, exit code 해석, 결과 주입 경로 |
| [06_timeout_and_errors.md](06_timeout_and_errors.md) | 타임아웃 위계, AbortSignal, 에러 분류, 비동기 재기동 |

## 핵심 파일 지도

```
src/
├── schemas/
│   └── hooks.ts                         # Hook Zod 스키마 (command/prompt/agent/http)
├── entrypoints/sdk/
│   ├── coreTypes.ts                     # HOOK_EVENTS 상수 (27개)
│   └── coreSchemas.ts                   # Hook 입력/출력 JSON 스키마
├── utils/
│   ├── hooks.ts                         # [메인 엔진] executeHooks, execCommandHook, processHookJSONOutput
│   └── hooks/
│       ├── hooksSettings.ts             # getAllHooks / getHooksForEvent
│       ├── hooksConfigSnapshot.ts       # 앱 시작 시 설정 스냅샷
│       ├── hooksConfigManager.ts        # 이벤트 메타데이터(설명/matcher 필드)
│       ├── hookEvents.ts                # SDK용 span/progress 이벤트 방출
│       ├── execPromptHook.ts            # LLM 기반 prompt hook
│       ├── execAgentHook.ts             # 멀티턴 agent hook
│       ├── execHttpHook.ts              # HTTP POST hook + SSRF 가드
│       ├── AsyncHookRegistry.ts         # 비동기 hook 추적
│       ├── sessionHooks.ts              # 세션 범위 임시 hook
│       ├── ssrfGuard.ts                 # URL allowlist 검증
│       └── postSamplingHooks.ts         # 샘플링 후 hook 훅
├── services/tools/
│   └── toolHooks.ts                     # runPreToolUseHooks / runPostToolUseHooks / Failure
└── utils/processUserInput/
    └── processUserInput.ts              # UserPromptSubmit hook 호출 지점
```

## 3줄 요약

- 설정은 앱 시작 시 **스냅샷**으로 고정되고, 이벤트 발생 시 matcher/if로 필터링 후 **병렬 실행**된다.
- 각 hook은 **exit code(0/2/기타)** 와 **선택적 JSON 출력**으로 결과를 돌려주며, `hookSpecificOutput`을 통해 tool input 수정·context 주입·권한 결정을 제어한다.
- Workspace trust, URL allowlist, 타임아웃, AbortSignal 조합으로 신뢰 경계 밖 코드의 RCE를 방어한다.
