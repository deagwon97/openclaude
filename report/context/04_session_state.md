# 04. 세션/상태 관리

## 관련 파일

- `src/bootstrap/state.ts` — 글로벌 상태 타입 정의 (~400줄)
- `src/utils/sessionStorage.ts` — JSONL 저장/로드 (~800줄)
- `src/utils/sessionStoragePortable.ts` — 세션 로드 (--resume)

## 글로벌 상태 (State)

파일: `src/bootstrap/state.ts`

런타임 중 메모리에 유지되는 상태:

```typescript
type State = {
  // 세션 식별
  sessionId: SessionId
  parentSessionId?: SessionId    // plan→implement 등 부모 세션
  originalCwd: string
  projectRoot: string
  cwd: string

  // API 추적
  lastAPIRequest: BetaMessageStreamParams | null
  lastAPIRequestMessages: Message[] | null
  lastMainRequestId?: string
  lastApiCompletionTimestamp: number | null

  // 비용/토큰 추적
  totalCostUSD: number
  totalAPIDuration: number
  modelUsage: { [modelName: string]: ModelUsage }

  // 컨텍스트 캐시
  systemPromptSectionCache: Map<string, string | null>
  cachedClaudeMdContent: string | null

  // Auto-Compact 상태
  pendingPostCompaction: boolean   // 압축 직후 첫 API 호출 태그용

  // 기타
  sessionPersistenceDisabled: boolean
  invokedSkills: Map<string, {...}>
  sessionCreatedTeams: Set<string>  // Swarm 팀 추적
}
```

## 세션 저장 형식

파일 경로: `~/.claude/sessions/<project-hash>/<session-id>.jsonl`

각 줄이 하나의 JSON 엔트리 (JSONL 포맷):

```jsonl
{"type":"user","uuid":"aaa","parentUuid":null,"content":"...","timestamp":"..."}
{"type":"assistant","uuid":"bbb","parentUuid":"aaa","message":{...},"usage":{...}}
{"type":"user","uuid":"ccc","parentUuid":"bbb","toolUseResult":{...}}
{"type":"marble-origami-commit","collapseId":"...","summaryContent":"..."}
{"type":"content-replacement","replacements":[...]}
{"type":"worktree-state","worktreeSession":{...}}
```

### 특수 엔트리 타입

| 타입 | 설명 |
|------|------|
| `user` / `assistant` | 일반 대화 메시지 |
| `marble-origami-commit` | Context collapse 경계 (feature-gated) |
| `content-replacement` | 대용량 tool 결과 외부 저장 참조 |
| `worktree-state` | Git worktree 상태 스냅샷 |

### Context Collapse 엔트리

```typescript
type ContextCollapseCommitEntry = {
  type: 'marble-origami-commit'
  collapseId: string           // 16자리 ID
  summaryContent: string       // 압축된 요약 내용
  firstArchivedUuid: string    // 압축 범위 시작 메시지 UUID
  lastArchivedUuid: string     // 압축 범위 끝 메시지 UUID
}
```

## 세션 저장 프로세스

각 턴(사용자 입력 → 어시스턴트 응답) 완료 후:

```typescript
appendTranscriptEntry(entry)
// JSONL 파일에 새 엔트리 추가 (append-only)
// 세션 끝에 메타데이터 재기록:
//   - CustomTitleMessage (대화 제목)
//   - SummaryMessage (짧은 요약)
//   - TagMessage (태그)
```

## 세션 로드 프로세스 (--resume)

파일: `src/utils/sessionStoragePortable.ts`

```typescript
readTranscriptForLoad(sessionId)
// 1. JSONL 파일 파싱
// 2. parentUuid 링크로 메시지 체인 재구성
// 3. content-replacement 엔트리로 대용량 내용 복원
// 4. marble-origami-commit 엔트리로 collapse 상태 복원
// 5. 재조립된 Message[] 반환
```

## 프롬프트 캐시 상태

state에서 관리하는 캐시 관련 플래그:

```typescript
bootstrapState.promptCache1hAllowlist    // 1시간 캐시 허용 모델 목록
bootstrapState.promptCache1hEligible     // 사용자 1시간 캐시 자격 (latch)
bootstrapState.cacheEditingHeaderLatched // API 헤더 재전송 방지
bootstrapState.afkModeHeaderLatched      // AFK 모드 헤더 보존
bootstrapState.fastModeHeaderLatched     // Fast 모드 헤더 보존
```
