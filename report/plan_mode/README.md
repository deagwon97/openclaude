# Plan Mode 내부 구현 분석

Plan Mode 는 "먼저 조사하고 설계한 뒤, 사용자 승인을 받고 실행" 이라는 2단계 워크플로우를 하네스 수준에서 강제하는 모드다. 단일 도구가 아니라 **권한 모드 + 2개 도구 + 파일 시스템 + attachment 주입** 이 맞물려 동작한다.

## 문서 목록

| 파일 | 주제 |
|------|------|
| [01_overview.md](01_overview.md) | 전체 흐름: 진입·탐색·승인·실행 |
| [02_enter_exit_tools.md](02_enter_exit_tools.md) | `EnterPlanMode` / `ExitPlanMode` 도구 구현 |
| [03_permissions.md](03_permissions.md) | `prePlanMode` 스태시, dangerous rule strip, auto 모드 상호작용 |
| [04_plan_file_and_slug.md](04_plan_file_and_slug.md) | 플랜 파일 저장 위치, slug 생성, 세션 재개 |
| [05_teammate_approval.md](05_teammate_approval.md) | 팀 리더 승인 경로 (`plan_mode_required` teammate) |
| [06_interview_phase.md](06_interview_phase.md) | Plan Mode V2 인터뷰 단계 + Pewter Ledger 실험 |

---

## 핵심 파일

| 역할 | 파일 |
|------|------|
| 권한 모드 타입 & 심볼 | [src/utils/permissions/PermissionMode.ts](../../src/utils/permissions/PermissionMode.ts) |
| Enter 도구 | [src/tools/EnterPlanModeTool/EnterPlanModeTool.ts](../../src/tools/EnterPlanModeTool/EnterPlanModeTool.ts) |
| Enter 도구 프롬프트 | [src/tools/EnterPlanModeTool/prompt.ts](../../src/tools/EnterPlanModeTool/prompt.ts) |
| Exit 도구 | [src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts](../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts) |
| 권한 전환 로직 | [src/utils/permissions/permissionSetup.ts](../../src/utils/permissions/permissionSetup.ts) `prepareContextForPlanMode` |
| 플랜 파일 I/O | [src/utils/plans.ts](../../src/utils/plans.ts) |
| V2 인터뷰 게이트 | [src/utils/planModeV2.ts](../../src/utils/planModeV2.ts) |
| Plan mode exit attachment | [src/bootstrap/state.ts](../../src/bootstrap/state.ts) `handlePlanModeTransition` |
| `/plan` 슬래시 커맨드 | [src/commands/plan/plan.tsx](../../src/commands/plan/plan.tsx) |

---

## 전체 흐름 요약

```
┌─────────────────────────────────────────────────────────────────┐
│  진입 경로                                                         │
│  ┌──────────────┐   ┌───────────────┐   ┌────────────────────┐  │
│  │ Shift+Tab    │   │ /plan command │   │ EnterPlanMode tool │  │
│  │ 모드 토글      │   │               │   │ (LLM이 호출)        │  │
│  └──────┬───────┘   └───────┬───────┘   └─────────┬──────────┘  │
└─────────┼──────────────────┼──────────────────────┼─────────────┘
          ▼                  ▼                      ▼
   prepareContextForPlanMode(context)
   - prePlanMode = currentMode (복원용)
   - auto 모드라면 특수 처리 (transcript classifier)
   - mode = 'plan'
          │
          ▼
   handlePlanModeTransition(from, to)
   - to='plan' → needsPlanModeExitAttachment = false
          │
          ▼
   ┌────────────────────────────────────────────────┐
   │ PLAN MODE 세션                                   │
   │  - Read/Glob/Grep/Agent 만 허용 (쓰기 차단)       │
   │  - LLM 이 플랜 파일에 작성                        │
   │  - AskUserQuestion 으로 사용자에게 질문 가능       │
   │  - 플랜 슬러그 파일 생성·유지                      │
   └─────────────────┬──────────────────────────────┘
                     ▼
             ExitPlanMode tool 호출
             - 플랜 승인 대화상자
             - allowedPrompts 수집 (e.g. "run tests")
                     │
                     ▼
             Exit 처리:
             - mode = prePlanMode (복원)
             - strippedDangerousRules 복원
             - needsPlanModeExitAttachment = true
             - 다음 쿼리에 plan_mode_exit attachment 주입
             - initialMessage + allowedPrompts 설정
                     │
                     ▼
             IMPLEMENTATION 세션
             (복원된 모드로 실제 쓰기 수행)
```

---

## 관련 문서

- 권한 시스템 (향후): `../permissions/`
- 커맨드 시스템: [../commands/](../commands/)
- 에이전트 & 팀 리더: [../agents/07_coordinator_mode.md](../agents/07_coordinator_mode.md)
- 상태 관리: [../state/04_on_change_side_effects.md](../state/04_on_change_side_effects.md)
