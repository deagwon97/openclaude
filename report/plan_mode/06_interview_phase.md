# 06. Plan Mode V2 — 인터뷰 페이즈 & Pewter Ledger 실험

Plan Mode 는 두 가지 워크플로우 변종을 가진다. **5-phase** (기본) 과 **Interview phase** (ant 전용 + growthbook gate). 그리고 5-phase 의 Phase 4 는 `tengu_pewter_ledger` 실험으로 4-way A/B 테스트 중이다.

## 1. 두 가지 워크플로우

### 1.1 5-Phase 워크플로우 (기본)
[src/utils/messages.ts:3232-3297](../../src/utils/messages.ts#L3232-L3297) 의 `getPlanModeV2Instructions` 가 시스템 리마인더로 주입.

```
Phase 1: Initial Understanding
  - explore-subagent 으로 병렬 탐색 (최대 getPlanModeV2ExploreAgentCount() = 3)
  - AskUserQuestion 으로 질문 가능

Phase 2: Design
  - plan-subagent 으로 설계 (최대 getPlanModeV2AgentCount() = 1~3)
  - 서브스크립션 등급별 차등 (max-20x/enterprise = 3개 병렬)

Phase 3: Review
  - 에이전트 결과 종합, 사용자 질문

Phase 4: Final Plan (← 여기가 실험 대상)
  - 플랜 파일에 최종 작성

Phase 5: Call ExitPlanMode
  - "Is this plan okay?" 같은 텍스트 질문 금지 — 반드시 ExitPlanMode 사용
```

### 1.2 Interview Phase 워크플로우
`isPlanModeInterviewPhaseEnabled()` 가 true 이면 `getPlanModeInterviewInstructions` 가 대신 주입됨. 반복적(iterative) 플로우로, 에이전트 병렬 탐색 대신 LLM 이 직접 탐색·질문·작성을 섞어서 수행.

## 2. Interview Phase 게이트

[src/utils/planModeV2.ts:50-62](../../src/utils/planModeV2.ts#L50-L62)

```typescript
export function isPlanModeInterviewPhaseEnabled(): boolean {
  // ant 직원은 항상 on
  if (process.env.USER_TYPE === 'ant') return true

  const env = process.env.CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE
  if (isEnvTruthy(env)) return true
  if (isEnvDefinedFalsy(env)) return false

  // 외부 사용자: growthbook gate
  return getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_plan_mode_interview_phase',
    false,
  )
}
```

우선순위:
1. ant 직원 → 항상 on
2. 환경 변수 `CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE=true|false` → 그대로
3. 그 외 → growthbook `tengu_plan_mode_interview_phase` 게이트

## 3. 병렬 에이전트 개수

### 3.1 `getPlanModeV2AgentCount` — Plan agent 병렬도
[src/utils/planModeV2.ts:5-29](../../src/utils/planModeV2.ts#L5-L29)

```typescript
// 환경 변수 오버라이드
CLAUDE_CODE_PLAN_V2_AGENT_COUNT=N  (1 ≤ N ≤ 10)

// 기본 매트릭스
subscriptionType === 'max' && rateLimitTier === 'default_claude_max_20x'  → 3
subscriptionType === 'enterprise' || 'team'                               → 3
그 외                                                                      → 1
```

### 3.2 `getPlanModeV2ExploreAgentCount` — Explore agent 병렬도
```typescript
CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT=N  또는  기본값 3
```

Explore 는 모든 등급에서 3개. 리서치 도중 rate limit 걱정이 덜 해서.

## 4. Phase 4 실험 — `tengu_pewter_ledger`

[src/utils/planModeV2.ts:64-95](../../src/utils/planModeV2.ts#L64-L95) 주석에 이 실험의 **전체 맥락**이 문서화되어 있다.

### 4.1 Arms

| 이름 | 설명 | 지침 강도 |
|------|------|---------|
| `null` (control) | 기본 Phase 4 문구 | 기본 (5-7K char 기대) |
| `'trim'` | Context/Background 섹션 금지 | 중간 |
| `'cut'` | prose 최소화, 파일 경로 중심 | 강함 |
| `'cap'` | **Hard limit: 40 lines** | 최강 |

실제 문구는 [src/utils/messages.ts](../../src/utils/messages.ts) 에 `PLAN_PHASE4_CONTROL/TRIM/CUT/CAP` 상수로 정의.

### 4.2 베이스라인 (주석에서)
```
14일 관측, N=26.3M session, 82% Opus 4.6
p50: 4,906 chars  |  p90: 11,617 chars  |  mean: 6,207 chars
Reject rate: 20% (<2K chars) → 50% (20K+ chars)  ← 단조 증가
```

플랜이 **길수록 거부율이 높아진다**. 너무 길면 사용자가 읽기 싫어한다는 신호.

### 4.3 Primary Metric
```
session 단위 Avg Cost (fact__201omjcij85f)
```

왜 비용인가:
- Opus output 은 input 의 **5배 가격**
- 긴 output 이 session 총 비용을 좌우
- `planLengthChars` 는 cost 의 프록시일 뿐, goal 은 cost 자체

### 4.4 Guardrails
- feedback-bad rate (negative feedback 비율)
- requests/session (너무 얇은 플랜 → 구현 중 반복 이터레이션 증가)
- tool error rate

### 4.5 왜 `cap` arm 이 필요한가
주석에서 핵심 주의: `cap` arm 이 **planLengthChars 를 줄이면서도 session 총 output 은 오히려 늘릴 수 있다** — 짧은 플랜 파일 + write→count→edit 사이클 반복. 그래서 primary metric 을 플랜 길이가 아닌 **session cost** 로 잡음.

## 5. `getPewterLedgerVariant`

```typescript
export function getPewterLedgerVariant(): PewterLedgerVariant {
  const raw = getFeatureValue_CACHED_MAY_BE_STALE<string | null>(
    'tengu_pewter_ledger',
    null,
  )
  if (raw === 'trim' || raw === 'cut' || raw === 'cap') return raw
  return null
}
```

growthbook 값 그대로 매핑, 알 수 없는 값은 control 로 폴백. **5-phase 만 실험 대상**이고 interview-phase 는 reference population 으로 둔다.

## 6. 시스템 리마인더로 주입되는 지점

[src/utils/messages.ts](../../src/utils/messages.ts) 의 `plan_mode` attachment case 가 `getPlanModeV2Instructions` 를 호출. 이 attachment 는 매 쿼리마다 재평가되므로:
- 세션 중간에 설정이 바뀌어도 다음 턴에 반영
- A/B arm 도 growthbook 캐시 TTL 에 따라 업데이트 (`_CACHED_MAY_BE_STALE`)

## 7. 분석 이벤트

| 이벤트 | 내용 |
|--------|------|
| `tengu_plan_exit` | `planLengthChars` 등 플랜 메트릭 (arm assignment 로깅) |
| `tengu_exit_plan_mode_called_outside_plan` | 오용 탐지 |
| `tengu_plan_mode_interview_phase_{on/off}` | interview-phase 활성 여부 추적 |

## 8. 종합

```
┌────────────────────────────────────────────┐
│  Plan Mode 진입                              │
│  (EnterPlanMode / Shift+Tab / /plan)       │
└────────────────┬───────────────────────────┘
                 ▼
        isPlanModeInterviewPhaseEnabled()
         ┌───────────────────┐
         │ true              │ false
         ▼                   ▼
  Interview Phase      5-Phase Workflow
  (iterative)          (parallel agents)
                              │
                              ▼
                  Phase 4: getPewterLedgerVariant()
                     ├─ null  → CONTROL
                     ├─ trim  → TRIM
                     ├─ cut   → CUT
                     └─ cap   → CAP (40-line hard limit)
```

## 9. 관련 파일

| 역할 | 파일 |
|------|------|
| 게이트 함수 & A/B arm | [src/utils/planModeV2.ts](../../src/utils/planModeV2.ts) |
| 시스템 리마인더 본문 | [src/utils/messages.ts](../../src/utils/messages.ts) (`getPlanModeV2Instructions`, `getPlanPhase4Section`) |
| 5-phase 워크플로우 상수 | [src/utils/messages.ts](../../src/utils/messages.ts) (`PLAN_PHASE4_*`) |
| Interview phase 워크플로우 | [src/utils/messages.ts](../../src/utils/messages.ts) (`getPlanModeInterviewInstructions`) |
| Growthbook 클라이언트 | [src/services/analytics/growthbook.ts](../../src/services/analytics/growthbook.ts) |
