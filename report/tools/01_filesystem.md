# 파일 시스템 도구 (File System Tools)

파일 읽기, 쓰기, 편집, 검색을 담당하는 핵심 도구들입니다.

---

## Read (FileReadTool)

**소스**: [src/tools/FileReadTool/FileReadTool.ts](../../src/tools/FileReadTool/FileReadTool.ts)

### 설명
로컬 파일시스템에서 파일을 읽는 도구. 텍스트, 이미지(PNG/JPG 등), PDF, Jupyter 노트북(.ipynb)을 지원한다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `file_path` | string | 필수 | 읽을 파일의 절대 경로 |
| `offset` | number | 선택 | 읽기 시작 줄 번호 (대용량 파일 부분 읽기) |
| `limit` | number | 선택 | 읽을 줄 수 |

### 동작
- 최대 2000줄 기본 읽기 (offset/limit으로 조절)
- 이미지: 멀티모달로 시각적 처리
- PDF: 최대 20페이지, `pages` 파라미터로 범위 지정
- 노트북: 모든 셀과 출력 결합 반환
- 읽은 파일은 `readFileState`에 타임스탬프와 함께 캐싱 (이후 Edit/Write 검증에 사용)

### 권한
- `read` 권한 체크 (`checkReadPermissionForTool`)

---

## Edit (FileEditTool)

**소스**: [src/tools/FileEditTool/FileEditTool.ts](../../src/tools/FileEditTool/FileEditTool.ts)

### 설명
파일 내에서 특정 문자열을 찾아 다른 문자열로 대치하는 정밀 편집 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `file_path` | string | 필수 | 편집할 파일의 절대 경로 |
| `old_string` | string | 필수 | 대치할 원본 문자열 (빈 문자열이면 새 파일 생성) |
| `new_string` | string | 필수 | 새로운 문자열 |
| `replace_all` | boolean | 선택 | true이면 모든 일치 항목 대치 (기본값: false) |

### 동작 흐름
1. 파일 읽기 권한 및 deny 규칙 확인
2. `readFileState`에서 마지막 읽기 타임스탬프 확인 (파일을 먼저 Read해야 함)
3. 파일 수정 시각 vs 마지막 읽기 시각 비교 (외부 변경 감지)
4. `old_string` 문자열 탐색 (따옴표 정규화 포함)
5. 파일 쓰기 및 LSP 서버에 변경 알림 (`didChange`, `didSave`)
6. VSCode diff view 업데이트
7. `readFileState` 갱신

### 검증 규칙
- `old_string === new_string`이면 거부
- 파일을 먼저 Read하지 않으면 거부
- 외부에서 파일이 변경되었으면 거부
- `.ipynb` 파일은 `NotebookEdit` 사용 유도
- 파일 크기 최대 1 GiB

### 권한
- `write` 권한 체크 (`checkWritePermissionForTool`)

---

## Write (FileWriteTool)

**소스**: [src/tools/FileWriteTool/FileWriteTool.ts](../../src/tools/FileWriteTool/FileWriteTool.ts)

### 설명
파일을 완전히 새 내용으로 덮어쓰거나 새 파일을 생성하는 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `file_path` | string | 필수 | 쓸 파일의 절대 경로 |
| `content` | string | 필수 | 파일에 쓸 전체 내용 |

### 동작 흐름
1. 기존 파일이 있으면 먼저 Read했는지 확인 (스탈레니스 체크)
2. 부모 디렉토리 자동 생성
3. 파일 쓰기 (기존 파일 삭제 후 새로 생성)
4. LSP 서버 변경 알림
5. VSCode diff view 업데이트

### 반환값
- `type`: `"create"` (신규 생성) 또는 `"update"` (덮어쓰기)
- `structuredPatch`: diff 패치 정보

### 권한
- `write` 권한 체크 (`checkWritePermissionForTool`)

---

## Glob (GlobTool)

**소스**: [src/tools/GlobTool/GlobTool.ts](../../src/tools/GlobTool/GlobTool.ts)

### 설명
글로브 패턴으로 파일을 검색하는 도구. 수정 시각 기준으로 정렬하여 반환한다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `pattern` | string | 필수 | 글로브 패턴 (예: `**/*.ts`, `src/**/*.tsx`) |
| `path` | string | 선택 | 검색 기준 디렉토리 (기본값: 현재 작업 디렉토리) |

### 동작
- 기본 최대 100개 파일 반환 (`globLimits.maxResults`로 조절)
- 결과는 cwd 기준 상대경로로 반환 (토큰 절약)
- 동시 실행 안전 (`isConcurrencySafe: true`)
- 읽기 전용 (`isReadOnly: true`)

> **참고**: ant 빌드에서 embedded search tools(bfs/ugrep)가 있으면 GlobTool 비활성화

### 권한
- `read` 권한 체크 (`checkReadPermissionForTool`)

---

## Grep (GrepTool)

**소스**: [src/tools/GrepTool/GrepTool.ts](../../src/tools/GrepTool/GrepTool.ts)

### 설명
ripgrep을 이용해 파일 내용을 정규식으로 검색하는 도구. 세 가지 출력 모드를 지원한다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `pattern` | string | 필수 | 정규식 패턴 |
| `path` | string | 선택 | 검색 경로 |
| `glob` | string | 선택 | 파일 필터 글로브 (예: `*.ts`, `*.{ts,tsx}`) |
| `type` | string | 선택 | 파일 타입 필터 (예: `js`, `py`, `rust`) |
| `output_mode` | enum | 선택 | `files_with_matches` (기본) / `content` / `count` |
| `-B` | number | 선택 | 매치 이전 N줄 표시 (content 모드) |
| `-A` | number | 선택 | 매치 이후 N줄 표시 (content 모드) |
| `-C` / `context` | number | 선택 | 매치 전후 N줄 표시 (content 모드) |
| `-n` | boolean | 선택 | 줄 번호 표시 (content 모드, 기본값: true) |
| `-i` | boolean | 선택 | 대소문자 무시 |
| `head_limit` | number | 선택 | 최대 결과 수 (기본값: 250, 0이면 무제한) |
| `offset` | number | 선택 | 건너뛸 결과 수 (페이지네이션) |
| `multiline` | boolean | 선택 | 여러 줄에 걸친 패턴 매치 |

### 출력 모드
- **`files_with_matches`**: 매치되는 파일 경로 목록 (수정 시각 기준 정렬)
- **`content`**: 매치되는 줄의 실제 내용
- **`count`**: 파일별 매치 횟수

### 동작
- VCS 디렉토리(`.git`, `.svn`, `.hg` 등) 자동 제외
- 줄 최대 길이 500자 제한 (base64/minified 코드 노이즈 방지)
- 결과는 cwd 기준 상대경로로 반환

> **참고**: ant 빌드에서 embedded search tools 있으면 GrepTool 비활성화

### 권한
- `read` 권한 체크 (`checkReadPermissionForTool`)

---

## NotebookEdit (NotebookEditTool)

**소스**: [src/tools/NotebookEditTool/NotebookEditTool.ts](../../src/tools/NotebookEditTool/NotebookEditTool.ts)

### 설명
Jupyter 노트북(`.ipynb`) 파일의 셀을 편집, 삽입, 삭제하는 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `notebook_path` | string | 필수 | `.ipynb` 파일의 절대 경로 |
| `cell_id` | string | 선택 | 편집할 셀 ID (insert 모드에서는 이 셀 뒤에 삽입) |
| `new_source` | string | 필수 | 셀의 새 소스 코드 |
| `cell_type` | enum | 선택 | `code` 또는 `markdown` (insert 모드에서 필수) |
| `edit_mode` | enum | 선택 | `replace` (기본) / `insert` / `delete` |

### 동작
- `replace`: 지정한 셀의 소스를 새 내용으로 교체
- `insert`: 지정한 셀 뒤에 새 셀 삽입 (`cell_id` 미지정 시 맨 앞에 삽입)
- `delete`: 지정한 셀 삭제

### 권한
- `write` 권한 체크 (`checkWritePermissionForTool`)
