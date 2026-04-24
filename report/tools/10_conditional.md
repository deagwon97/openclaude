# 조건부 도구 (Conditional / Internal Tools)

특정 환경 변수, 기능 플래그, 또는 빌드 설정에 따라 조건부로 활성화되는 도구들입니다.

---

## Config (ConfigTool)

**소스**: [src/tools/ConfigTool/ConfigTool.ts](../../src/tools/ConfigTool/ConfigTool.ts)

### 설명
Claude Code 설정을 읽고 쓰는 도구. 허용된 설정 항목만 변경 가능하다.

### 활성화 조건
- `process.env.USER_TYPE === 'ant'` (Anthropic 내부 빌드)

---

## Tungsten (TungstenTool)

**소스**: [src/tools/TungstenTool/TungstenTool.ts](../../src/tools/TungstenTool/TungstenTool.ts)

### 설명
Anthropic 내부 분석 및 모니터링 도구.

### 활성화 조건
- `process.env.USER_TYPE === 'ant'` (Anthropic 내부 빌드)

---

## REPL (REPLTool)

**소스**: [src/tools/REPLTool/REPLTool.ts](../../src/tools/REPLTool/REPLTool.ts)

### 설명
VM 샌드박스 환경에서 JavaScript/Python 등의 코드를 실행하는 도구. REPL 모드에서는 Bash, FileRead, FileEdit 등의 primitive tool들이 직접 노출되지 않고 REPL 내부에서만 접근 가능하다.

### 활성화 조건
- `process.env.USER_TYPE === 'ant'` AND REPL 모드 활성화

### 특징
- REPL 모드에서는 다른 파일 시스템/실행 도구들이 숨겨짐 (`REPL_ONLY_TOOLS`)
- Simple 모드 + REPL 모드 조합 시 REPL이 Bash/FileRead/FileEdit을 대체

---

## Sleep (SleepTool)

**소스**: [src/tools/SleepTool/SleepTool.ts](../../src/tools/SleepTool/SleepTool.ts)

### 설명
지정된 시간(초) 동안 대기하는 도구. 주로 proactive/자율 에이전트가 다음 작업 전에 대기할 때 사용한다.

### 활성화 조건
- `feature('PROACTIVE')` 또는 `feature('KAIROS')` 활성화 시

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `seconds` | number | 필수 | 대기할 시간 (초) |

---

## Monitor (MonitorTool)

**소스**: [src/tools/MonitorTool/MonitorTool.ts](../../src/tools/MonitorTool/MonitorTool.ts)

### 설명
백그라운드 프로세스의 stdout을 스트리밍으로 모니터링하는 도구. 이벤트 기반 알림(각 stdout 줄이 알림)을 지원한다.

### 활성화 조건
- `feature('MONITOR_TOOL')` 활성화 시

---

## WebBrowser (WebBrowserTool)

**소스**: [src/tools/WebBrowserTool/WebBrowserTool.ts](../../src/tools/WebBrowserTool/WebBrowserTool.ts)

### 설명
Playwright 등을 이용한 헤드리스 브라우저 자동화 도구. 인증이 필요한 페이지나 JavaScript 렌더링이 필요한 페이지에 접근할 수 있다.

### 활성화 조건
- `feature('WEB_BROWSER_TOOL')` 활성화 시

---

## TeamCreate (TeamCreateTool)

**소스**: [src/tools/TeamCreateTool/TeamCreateTool.ts](../../src/tools/TeamCreateTool/TeamCreateTool.ts)

### 설명
멀티에이전트 팀을 구성하는 도구. 여러 에이전트가 협력하는 스웜(Swarm) 아키텍처에서 사용한다.

### 활성화 조건
- `isAgentSwarmsEnabled()` 반환값이 `true`인 경우

### 동작
- 지정된 에이전트 타입들로 팀 구성
- 각 팀원에게 역할과 작업 분배 가능

---

## TeamDelete (TeamDeleteTool)

**소스**: [src/tools/TeamDeleteTool/TeamDeleteTool.ts](../../src/tools/TeamDeleteTool/TeamDeleteTool.ts)

### 설명
생성된 멀티에이전트 팀을 해체하는 도구.

### 활성화 조건
- `isAgentSwarmsEnabled()` 반환값이 `true`인 경우

---

## SendMessage (SendMessageTool)

**소스**: [src/tools/SendMessageTool/SendMessageTool.ts](../../src/tools/SendMessageTool/SendMessageTool.ts)

### 설명
에이전트 간 또는 에이전트에서 사용자에게 메시지를 전송하는 도구. Coordinator 모드나 멀티에이전트 아키텍처에서 에이전트 간 통신에 사용된다.

### 활성화 조건
- 항상 포함 (단, 기능 플래그에 따라 동작 범위 달라짐)

### 주요 사용 시나리오
- Coordinator → Worker 에이전트 지시
- Worker → Coordinator 결과 보고
- Agent → 사용자 알림

---

## 조건부 도구 활성화 요약

| 도구 | 활성화 조건 | 환경/플래그 |
|------|-----------|------------|
| Config | `USER_TYPE=ant` | 환경변수 |
| Tungsten | `USER_TYPE=ant` | 환경변수 |
| REPL | `USER_TYPE=ant` + REPL 모드 | 환경변수 |
| SuggestBackgroundPR | `USER_TYPE=ant` | 환경변수 |
| Sleep | PROACTIVE 또는 KAIROS | 기능 플래그 |
| Monitor | MONITOR_TOOL | 기능 플래그 |
| WebBrowser | WEB_BROWSER_TOOL | 기능 플래그 |
| CronCreate/Delete/List | AGENT_TRIGGERS | 기능 플래그 |
| RemoteTrigger | AGENT_TRIGGERS_REMOTE | 기능 플래그 + GrowthBook |
| TeamCreate/Delete | Agent Swarms 활성화 | `isAgentSwarmsEnabled()` |
| EnterWorktree/ExitWorktree | Worktree 모드 활성화 | `isWorktreeModeEnabled()` |
| LSP | `ENABLE_LSP_TOOL=1` | 환경변수 |
| VerifyPlanExecution | `CLAUDE_CODE_VERIFY_PLAN=true` | 환경변수 |
| SendUserFile | KAIROS | 기능 플래그 |
| PushNotification | KAIROS 또는 KAIROS_PUSH_NOTIFICATION | 기능 플래그 |
| SubscribePR | KAIROS_GITHUB_WEBHOOKS | 기능 플래그 |
| TaskCreate/Get/Update/List | `isTodoV2Enabled()` | 설정 함수 |
| ToolSearch | `isToolSearchEnabledOptimistic()` | tool 수 임계값 |
