# OpenClaude Tools 전체 목록

이 디렉토리는 OpenClaude CLI에서 사용 가능한 모든 tool들을 카테고리별로 정리한 문서입니다.

## 카테고리 목록

| 파일 | 카테고리 | 포함 tool 수 |
|------|---------|------------|
| [01_filesystem.md](01_filesystem.md) | 파일 시스템 | 6개 |
| [02_execution.md](02_execution.md) | 실행 | 2개 |
| [03_agent_task.md](03_agent_task.md) | 에이전트 & 태스크 | 7개 |
| [04_web.md](04_web.md) | 웹 | 2개 |
| [05_planning.md](05_planning.md) | 계획 & 흐름 제어 | 5개 |
| [06_user_interaction.md](06_user_interaction.md) | 사용자 상호작용 | 2개 |
| [07_scheduling.md](07_scheduling.md) | 스케줄링 | 4개 |
| [08_mcp.md](08_mcp.md) | MCP | 3개 |
| [09_dev_support.md](09_dev_support.md) | 개발 지원 | 3개 |
| [10_conditional.md](10_conditional.md) | 조건부 (실험적/내부) | 8개 |

---

## 전체 Tool 요약

### 파일 시스템 (File System)

| Tool 이름 | 설명 |
|-----------|------|
| `Read` | 파일 읽기 (이미지, PDF, 노트북 포함) |
| `Edit` | 파일 내 문자열 대치 (정밀 편집) |
| `Write` | 파일 전체 내용 쓰기 (생성/덮어쓰기) |
| `Glob` | 글로브 패턴으로 파일 검색 |
| `Grep` | 정규식으로 파일 내용 검색 (ripgrep) |
| `NotebookEdit` | Jupyter 노트북 셀 편집 |

### 실행 (Execution)

| Tool 이름 | 설명 |
|-----------|------|
| `Bash` | 쉘 명령 실행 (포그라운드/백그라운드) |
| `PowerShell` | PowerShell 명령 실행 (Windows) |

### 에이전트 & 태스크 (Agent & Task)

| Tool 이름 | 설명 |
|-----------|------|
| `Agent` | 독립적인 서브에이전트 실행 |
| `TaskOutput` | 백그라운드 태스크 출력 조회/대기 |
| `TaskStop` | 실행 중인 백그라운드 태스크 중지 |
| `TaskCreate` | 태스크 생성 (TodoV2 활성화 시) |
| `TaskGet` | 특정 태스크 조회 (TodoV2 활성화 시) |
| `TaskUpdate` | 태스크 상태 업데이트 (TodoV2 활성화 시) |
| `TaskList` | 태스크 목록 조회 (TodoV2 활성화 시) |

### 웹 (Web)

| Tool 이름 | 설명 |
|-----------|------|
| `WebFetch` | URL에서 콘텐츠 가져오기 |
| `WebSearch` | 웹 검색 |

### 계획 & 흐름 제어 (Planning & Flow Control)

| Tool 이름 | 설명 |
|-----------|------|
| `EnterPlanMode` | 계획 모드 진입 (읽기 전용 탐색/설계 단계) |
| `ExitPlanMode` | 계획 모드 종료 및 실행 승인 요청 |
| `EnterWorktree` | 격리된 git worktree 생성 및 진입 |
| `ExitWorktree` | worktree 종료 (유지 또는 삭제) |
| `TodoWrite` | 현재 세션의 할 일 목록 관리 |

### 사용자 상호작용 (User Interaction)

| Tool 이름 | 설명 |
|-----------|------|
| `AskUserQuestion` | 사용자에게 구조화된 질문 제시 |
| `Brief` | 사용자에게 메시지/알림 전송 |

### 스케줄링 (Scheduling)

| Tool 이름 | 설명 |
|-----------|------|
| `CronCreate` | 크론 스케줄로 프롬프트 예약 |
| `CronDelete` | 예약된 크론 작업 삭제 |
| `CronList` | 예약된 크론 작업 목록 조회 |
| `RemoteTrigger` | 원격 에이전트 트리거 관리 |

### MCP

| Tool 이름 | 설명 |
|-----------|------|
| `ListMcpResources` | 연결된 MCP 서버 리소스 목록 조회 |
| `ReadMcpResource` | MCP 서버 리소스 내용 읽기 |
| `ToolSearch` | 지연 로드된 tool 스키마 조회 |

### 개발 지원 (Dev Support)

| Tool 이름 | 설명 |
|-----------|------|
| `Skill` | 슬래시 커맨드(스킬) 실행 |
| `LSP` | LSP 기반 코드 분석 (정의 이동, 참조 찾기 등) |
| `ToolSearch` | 지연 로드된 tool 검색 및 스키마 로드 |

### 조건부 도구 (Conditional / Internal)

| Tool 이름 | 활성화 조건 | 설명 |
|-----------|-----------|------|
| `Config` | `USER_TYPE=ant` | Claude 설정 관리 |
| `Tungsten` | `USER_TYPE=ant` | 내부 분석 도구 |
| `REPL` | `USER_TYPE=ant` | VM 내 REPL 환경 |
| `Sleep` | `PROACTIVE/KAIROS` 기능 | 일정 시간 대기 |
| `Monitor` | `MONITOR_TOOL` 기능 | 백그라운드 프로세스 모니터링 |
| `WebBrowser` | `WEB_BROWSER_TOOL` 기능 | 브라우저 자동화 |
| `TeamCreate` | Agent Swarms 활성화 | 멀티에이전트 팀 생성 |
| `TeamDelete` | Agent Swarms 활성화 | 멀티에이전트 팀 삭제 |
| `SendMessage` | 상시 포함 | 에이전트 간 메시지 전송 |

---

## Tool 로드 방식

```
getAllBaseTools()         ← 환경에 따라 모든 base tool 반환
    ↓
filterToolsByDenyRules() ← 사용자 deny 규칙으로 필터링
    ↓
isEnabled() 체크         ← 각 tool의 활성화 여부 확인
    ↓
assembleToolPool()       ← 빌트인 + MCP tool 통합
```

- **Simple 모드** (`CLAUDE_CODE_SIMPLE=true`): Bash, FileRead, FileEdit만 제공
- **REPL 모드**: primitive tool들은 VM 내부에서만 접근 가능
- **MCP tool**: 빌트인 tool과 동일 파이프라인으로 처리, 이름 충돌 시 빌트인 우선
