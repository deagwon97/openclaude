# 계획 & 흐름 제어 도구 (Planning & Flow Control Tools)

작업 계획 수립, 실행 격리, 진행 상황 추적을 위한 도구들입니다.

---

## EnterPlanMode (EnterPlanModeTool)

**소스**: [src/tools/EnterPlanModeTool/EnterPlanModeTool.ts](../../src/tools/EnterPlanModeTool/EnterPlanModeTool.ts)

### 설명
복잡한 작업에서 코드 작성 전에 탐색과 설계를 위한 계획 모드로 진입하는 도구.

### 입력 파라미터
없음 (파라미터 없음)

### 동작
1. 현재 permission mode를 `plan`으로 변경
2. 파일 쓰기/편집 도구 비활성화 (읽기 전용 모드)
3. 에이전트 컨텍스트에서는 사용 불가 (메인 스레드 전용)

### 계획 모드에서 해야 할 일
1. 코드베이스를 탐색하여 기존 패턴 파악
2. 유사한 기능 및 아키텍처 접근 방식 파악
3. 여러 구현 방안과 트레이드오프 검토
4. 필요 시 AskUserQuestion으로 접근 방식 명확화
5. 구체적인 구현 전략 설계
6. ExitPlanMode로 계획 제출 및 승인 요청

### 비활성화 조건
- KAIROS 또는 KAIROS_CHANNELS 기능 활성화 + `--channels` 옵션 사용 시

---

## ExitPlanMode (ExitPlanModeV2Tool)

**소스**: [src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts](../../src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts)

### 설명
계획 모드에서 나와 실행 승인을 요청하는 도구. 작성한 계획을 사용자에게 제시하고 승인을 받아야 실제 코드 작성 단계로 넘어갈 수 있다.

### 동작
1. 계획 내용 취합
2. 사용자에게 계획 제시 및 승인 요청
3. 승인 시 permission mode를 이전 모드로 복원
4. 원격 환경에서는 계획 파일 스냅샷 저장

---

## EnterWorktree (EnterWorktreeTool)

**소스**: [src/tools/EnterWorktreeTool/EnterWorktreeTool.ts](../../src/tools/EnterWorktreeTool/EnterWorktreeTool.ts)

### 설명
격리된 git worktree를 생성하고 세션을 그 안으로 이동시키는 도구. 독립적인 브랜치에서 안전하게 작업할 수 있게 한다.

### 활성화 조건
- `isWorktreeModeEnabled()` 반환값이 `true`인 경우

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `name` | string | 선택 | worktree 이름 (미지정 시 랜덤 생성) |

### 이름 규칙
- 영문자, 숫자, 점, 밑줄, 대시만 허용
- `/`로 구분하는 경로 세그먼트 허용
- 최대 64자

### 동작
1. 현재 git 루트 탐색
2. 새 worktree 브랜치 생성 (`createWorktreeForSession()`)
3. CWD를 worktree 경로로 변경
4. 세션 상태에 worktree 정보 저장

### 반환값
```typescript
{
  worktreePath: string,   // worktree 경로
  worktreeBranch: string, // 생성된 브랜치 이름
  message: string         // 확인 메시지
}
```

---

## ExitWorktree (ExitWorktreeTool)

**소스**: [src/tools/ExitWorktreeTool/ExitWorktreeTool.ts](../../src/tools/ExitWorktreeTool/ExitWorktreeTool.ts)

### 설명
현재 worktree 세션을 종료하고 원래 디렉토리로 돌아오는 도구.

### 활성화 조건
- `isWorktreeModeEnabled()` 반환값이 `true`인 경우

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `action` | enum | 필수 | `"keep"` (worktree 유지) / `"remove"` (worktree 삭제) |
| `discard_changes` | boolean | 선택 | `action="remove"` 시 미커밋 변경사항/커밋이 있으면 반드시 `true` 필요 |

### 동작
- `keep`: worktree와 브랜치를 디스크에 남기고 원래 CWD로 복귀
- `remove`: worktree와 브랜치 삭제 후 원래 CWD로 복귀
  - 미커밋 파일이나 미머지 커밋이 있으면 `discard_changes: true` 없이는 거부

---

## TodoWrite (TodoWriteTool)

**소스**: [src/tools/TodoWriteTool/TodoWriteTool.ts](../../src/tools/TodoWriteTool/TodoWriteTool.ts)

### 설명
현재 세션의 할 일 목록을 업데이트하는 도구. Claude가 작업 진행 상황을 스스로 추적하기 위해 사용한다.

### 활성화 조건
- `isTodoV2Enabled()` 반환값이 `false`인 경우 (TodoV2가 비활성화된 경우)
- TodoV2 활성화 시에는 TaskCreate/Get/Update/List를 대신 사용

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `todos` | TodoItem[] | 필수 | 업데이트된 전체 할 일 목록 |

### TodoItem 구조
```typescript
{
  id: string,
  content: string,
  status: "pending" | "in_progress" | "completed"
}
```

### 동작
- 현재 에이전트 ID 또는 세션 ID를 키로 앱 상태에 저장
- 모든 항목이 `completed`이면 빈 목록으로 초기화 (완료 후 클린업)
- 3개 이상 항목 완료 시 검증 에이전트 사용 유도 (VERIFICATION_AGENT 기능 활성화 시)

### 사용 권장 시점
- 복잡한 다단계 작업 시작 시 계획 수립
- 각 단계 완료 시 상태 업데이트
- 중간에 새로운 작업 발견 시 목록에 추가
