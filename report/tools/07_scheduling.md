# 스케줄링 도구 (Scheduling Tools)

프롬프트나 에이전트를 예약 실행하는 도구들입니다.

---

## CronCreate (CronCreateTool)

**소스**: [src/tools/ScheduleCronTool/CronCreateTool.ts](../../src/tools/ScheduleCronTool/CronCreateTool.ts)

### 설명
크론 표현식(5-field cron schedule)으로 프롬프트를 반복 또는 일회성으로 예약 실행하는 도구.

### 활성화 조건
- `feature('AGENT_TRIGGERS')` 활성화 시

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `cron` | string | 필수 | 5-field cron 표현식 (예: `*/5 * * * *` = 5분마다) |
| `prompt` | string | 필수 | 각 실행 시 전달할 프롬프트 |
| `recurring` | boolean | 선택 | `true` (기본) = 반복 실행, `false` = 1회 후 삭제 |
| `durable` | boolean | 선택 | `true` = 재시작 후에도 유지 (`.claude/scheduled_tasks.json`에 저장), `false` (기본) = 세션 종료 시 삭제 |

### 크론 표현식 형식
```
"M H DoM Mon DoW"
예시:
  */5 * * * *      → 5분마다
  30 14 28 2 *     → 2월 28일 오후 2시 30분
  0 9 * * 1        → 매주 월요일 오전 9시
```

### 제한
- 최대 50개 크론 작업 등록 가능

### 반환값
```typescript
{
  id: string,            // 생성된 크론 작업 ID
  humanSchedule: string, // 사람이 읽기 쉬운 스케줄 설명
  recurring: boolean,
  durable?: boolean
}
```

---

## CronDelete (CronDeleteTool)

**소스**: [src/tools/ScheduleCronTool/CronDeleteTool.ts](../../src/tools/ScheduleCronTool/CronDeleteTool.ts)

### 설명
예약된 크론 작업을 삭제하는 도구.

### 활성화 조건
- `feature('AGENT_TRIGGERS')` 활성화 시

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `id` | string | 필수 | 삭제할 크론 작업 ID |

---

## CronList (CronListTool)

**소스**: [src/tools/ScheduleCronTool/CronListTool.ts](../../src/tools/ScheduleCronTool/CronListTool.ts)

### 설명
등록된 모든 크론 작업 목록을 조회하는 도구.

### 활성화 조건
- `feature('AGENT_TRIGGERS')` 활성화 시

### 입력 파라미터
없음

### 반환값
각 크론 작업의 ID, 스케줄, 프롬프트, recurring/durable 상태, 다음 실행 시각

---

## RemoteTrigger (RemoteTriggerTool)

**소스**: [src/tools/RemoteTriggerTool/RemoteTriggerTool.ts](../../src/tools/RemoteTriggerTool/RemoteTriggerTool.ts)

### 설명
claude.ai 서비스에서 관리되는 원격 에이전트 트리거(triggers)를 생성, 조회, 업데이트, 실행하는 도구.

### 활성화 조건
- `feature('AGENT_TRIGGERS_REMOTE')` 활성화 시
- `getFeatureValue('tengu_surreal_dali')` GrowthBook 피처 플래그가 `true`
- `isPolicyAllowed('allow_remote_sessions')` 정책 허용 시

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `action` | enum | 필수 | `list` / `get` / `create` / `update` / `run` |
| `trigger_id` | string | 조건부 | `get`, `update`, `run` 시 필수 |
| `body` | object | 조건부 | `create`, `update` 시 JSON 본문 |

### 동작
- claude.ai OAuth 인증 사용
- `ccr-triggers-2026-01-30` beta API 호출
- 원격 에이전트를 즉시 실행(`run`) 또는 스케줄 관리

### 인증 요구사항
- claude.ai OAuth 토큰 필요 (`getClaudeAIOAuthTokens()`)
- 조직 UUID 필요 (`getOrganizationUUID()`)
