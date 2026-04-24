# 개발 지원 도구 (Dev Support Tools)

개발 작업을 보조하는 도구들입니다.

---

## Skill (SkillTool)

**소스**: [src/tools/SkillTool/SkillTool.ts](../../src/tools/SkillTool/SkillTool.ts)

### 설명
슬래시 커맨드(`/commit`, `/review-pr` 등)를 실행하는 도구. 사용자 정의 스킬 또는 내장 커맨드를 에이전트 파이프라인 내에서 실행할 수 있게 한다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `skill` | string | 필수 | 실행할 스킬 이름 (예: `"commit"`, `"review-pr"`) |
| `args` | string | 선택 | 스킬에 전달할 인자 |

### 동작 흐름
1. 스킬 이름으로 커맨드 레지스트리에서 탐색
2. 스킬 파일의 frontmatter 파싱 (모델 오버라이드 등)
3. 스킬을 독립적인 서브에이전트 컨텍스트에서 실행
4. 결과 반환

### 스킬 발견 경로
- `.claude/skills/` 디렉토리 내 `.md` 파일
- `~/.claude/skills/` 글로벌 스킬 디렉토리
- 마켓플레이스 플러그인 스킬

### 권한 규칙
- 스킬별 allow/deny 규칙 설정 가능
- 공식 마켓플레이스 스킬은 별도 검증 과정 존재

### 예시 사용 내장 스킬
| 스킬 이름 | 설명 |
|---------|------|
| `commit` | git 커밋 생성 |
| `review-pr` | PR 리뷰 수행 |
| `simplify` | 코드 품질 검토 및 개선 |
| `schedule` | 원격 에이전트 스케줄링 |
| `claude-api` | Claude API 앱 빌드 |

---

## LSP (LSPTool)

**소스**: [src/tools/LSPTool/LSPTool.ts](../../src/tools/LSPTool/LSPTool.ts)

### 설명
Language Server Protocol(LSP)을 통해 코드 분석 기능을 제공하는 도구. 정의 찾기, 참조 찾기, 심볼 검색 등 IDE 수준의 코드 네비게이션 기능을 사용할 수 있다.

### 활성화 조건
- `ENABLE_LSP_TOOL=1` 환경변수가 설정된 경우

### 지원 작업 (operation)

| operation | 설명 |
|-----------|------|
| `go_to_definition` | 심볼의 정의로 이동 |
| `find_references` | 심볼의 모든 참조 찾기 |
| `document_symbols` | 현재 파일의 모든 심볼 목록 |
| `workspace_symbols` | 워크스페이스 전체 심볼 검색 |
| `hover` | 심볼의 타입/문서 정보 |
| `prepare_call_hierarchy` | 함수 호출 계층 준비 |
| `incoming_calls` | 해당 함수를 호출하는 곳 목록 |
| `outgoing_calls` | 해당 함수가 호출하는 곳 목록 |

### 입력 파라미터 (공통)

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `operation` | enum | 필수 | 수행할 LSP 작업 |
| `file_path` | string | 대부분 필수 | 대상 파일 경로 |
| `line` | number | 대부분 필수 | 줄 번호 (0-indexed) |
| `character` | number | 대부분 필수 | 문자 위치 (0-indexed) |
| `query` | string | workspace_symbols 시 필수 | 검색할 심볼 이름 |

### 동작
1. LSP 서버 초기화 대기 (`waitForInitialization()`)
2. LSP 서버에 요청 전송
3. 결과 포매팅 후 반환

### 파일 크기 제한
- 최대 10MB 파일만 처리 가능

---

## ToolSearch (ToolSearchTool)

> MCP 카테고리에도 포함됨. [08_mcp.md](08_mcp.md#toolsearch-toolsearchtool) 참조.

**소스**: [src/tools/ToolSearchTool/ToolSearchTool.ts](../../src/tools/ToolSearchTool/ToolSearchTool.ts)

### 설명
지연 로드된 tool들의 스키마를 검색하고 활성화하는 도구. tool 수가 많을 때 컨텍스트 창을 절약하기 위해 일부 tool의 스키마를 지연 로드하며, ToolSearch로 필요한 tool의 스키마를 가져온다.

### 주요 용도
- 특정 tool이 `shouldDefer: true`로 표시된 경우, 실제 사용 전에 ToolSearch로 스키마를 먼저 로드해야 함
- MCP tool이 연결된 서버가 많을 때 특히 유용

### 쿼리 예시
```
"select:WebFetch,WebSearch"   → 이름으로 직접 로드
"notebook edit"               → 키워드로 검색
"+mcp github"                 → 이름에 "mcp" 필수, "github"로 순위
```
