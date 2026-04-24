# 세션 메모리 vs 영구(자동) 메모리

---

## 두 메모리의 차이

| 항목 | 세션 메모리 | 자동 메모리(Auto Memory) |
|------|------------|--------------------------|
| 범위 | 단일 대화 | 프로젝트 전체 / 세션 간 공유 |
| 형식 | JSONL 트랜스크립트 | 마크다운 파일 |
| 저장 위치 | `~/.claude/projects/.../YYYY-MM-DD.jsonl` | `~/.claude/projects/.../memory/` |
| 관리 주체 | 자동 (모든 메시지 기록) | Claude가 판단하여 선택적 저장 |
| 수명 | 대화가 끝나도 파일로 남음 | 명시적 삭제 전까지 영구 보존 |
| 용도 | 대화 재현, 로그 | 장기 컨텍스트, 사용자 선호도 |

---

## 세션 메모리

### 핵심 파일

- [src/utils/sessionStorage.ts](../../src/utils/sessionStorage.ts)
- [src/utils/conversationRecovery.ts](../../src/utils/conversationRecovery.ts)
- [src/types/logs.ts](../../src/types/logs.ts)

### TranscriptMessage 타입

```typescript
// src/types/logs.ts:221-231
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null        // 메시지 체인 연결
  logicalParentUuid?: UUID       // 압축 경계 넘는 논리적 부모
  isSidechain: boolean           // 서브에이전트 대화 여부
  gitBranch?: string
  agentId?: string               // 서브에이전트 ID
  teamName?: string
  agentName?: string
  agentColor?: string
  promptId?: string              // OTel 추적 ID
}
```

### SerializedMessage 타입

```typescript
// src/types/logs.ts:8-17
type SerializedMessage = Message & {
  cwd: string                    // 작업 디렉토리
  userType: string
  entrypoint?: string            // cli / sdk-ts / sdk-py / vscode 등
  sessionId: string
  timestamp: string              // ISO 8601
  version: string
  gitBranch?: string
  slug?: string                  // 계획(plan) 파일 연결
}
```

### LogOption (세션 파일 메타데이터)

```typescript
// src/types/logs.ts:19-53
type LogOption = {
  date: string                   // YYYY-MM-DD
  messages: SerializedMessage[]
  fullPath?: string
  firstPrompt: string            // 첫 프롬프트 (미리보기)
  messageCount: number
  fileSize?: number
  isSidechain: boolean
  sessionId?: string
  customTitle?: string           // 사용자 지정 제목
  tag?: string                   // 검색 태그
  summary?: string               // AI 생성 요약
  contextCollapseCommits?: ContextCollapseCommitEntry[]  // 압축 기록
  contentReplacements?: ContentReplacementRecord[]       // 콘텐츠 교체 기록
}
```

---

## 자동 메모리 (Auto Memory)

### 핵심 파일

- [src/memdir/paths.ts](../../src/memdir/paths.ts)
- [src/memdir/memoryScan.ts](../../src/memdir/memoryScan.ts)
- [src/memdir/memoryTypes.ts](../../src/memdir/memoryTypes.ts)
- [src/memdir/memdir.ts](../../src/memdir/memdir.ts)

### 활성화 조건

```typescript
// src/memdir/paths.ts:30-55
export function isAutoMemoryEnabled(): boolean {
  // 비활성화 조건 (우선순위 순):
  // 1. CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 환경변수
  // 2. CLAUDE_CODE_SIMPLE (--bare 모드)
  // 3. CCR 영구 저장소 없음
  // 4. settings.json의 autoMemoryEnabled: false
  // 기본값: 활성화
}
```

### 메모리 경로

```typescript
// src/memdir/paths.ts:223-235
export const getAutoMemPath = memoize((): string => {
  // 우선순위:
  // 1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE 환경변수
  // 2. settings.json의 autoMemoryDirectory
  // 3. ~/.claude/projects/{sanitized-git-root}/memory/
})
```

### 디렉토리 구조

```
~/.claude/projects/{PROJECT}/memory/
├── MEMORY.md                    # 메인 인덱스 (최대 200줄 / 25KB)
├── user_role.md                 # 사용자 역할/선호도
├── feedback_testing.md          # 작업 피드백
├── project_context.md           # 프로젝트 맥락
└── logs/                        # 일일 로그 (KAIROS)
    └── YYYY/MM/YYYY-MM-DD.md
```

### MEMORY.md 제한

```typescript
// src/memdir/memdir.ts:57-102
MAX_ENTRYPOINT_LINES = 200      // 최대 200줄 (이후 잘림)
MAX_ENTRYPOINT_BYTES = 25_000   // 최대 25KB
```

MEMORY.md는 인덱스 역할만 한다. 실제 내용은 개별 .md 파일에 있다.

### 메모리 타입

```typescript
// src/memdir/memoryTypes.ts
type MemoryType =
  | 'User'      // 사용자 역할, 선호도, 배경
  | 'Project'   // 프로젝트 구조, 목표, 진행 상황
  | 'Local'     // 로컬 환경 설정
  | 'Managed'   // 관리형 메모리
  | 'AutoMem'   // 자동 생성 메모리
  | 'TeamMem'   // 팀 멤버 메모리 (feature 플래그)
```

### 메모리 스캔

```typescript
// src/memdir/memoryScan.ts:35-84
export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal,
): Promise<MemoryHeader[]> {
  // 조건:
  // - *.md 파일 (MEMORY.md 제외)
  // - 최대 3단계 깊이 (DoS 방지)
  // - 최대 200개 파일
  // - 최신 수정 순 정렬
}

type MemoryHeader = {
  filename: string
  filePath: string
  mtimeMs: number
  description: string | null   // frontmatter의 description
  type: MemoryType | undefined
}
```

### 메모리 파일 frontmatter 형식

```markdown
---
name: 메모리 이름
description: 한 줄 설명 (대화 컨텍스트 로드 여부 판단용)
type: user | project | feedback | reference
---

메모리 내용...
```

---

## 세션 복구

대화 중 비정상 종료 시 `conversationRecovery.ts`가  
마지막 세션 파일에서 메시지를 재로드한다.

- `parentUuid` 체인으로 메시지 트리 재구성
- `isSidechain: true` 메시지는 별도 서브에이전트 브랜치로 분리
- 압축 경계가 있으면 경계 이후 메시지만 API에 전송
