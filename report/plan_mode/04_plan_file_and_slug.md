# 04. 플랜 파일과 슬러그 (Slug) 관리

[src/utils/plans.ts](../../src/utils/plans.ts) 가 플랜 파일의 저장 위치와 슬러그 생성·재개·포크를 담당한다.

## 1. 저장 위치 — `getPlansDirectory`

### 1.1 결정 우선순위
1. `settings.plansDirectory` — 사용자가 지정한 프로젝트 내 상대 경로
2. 기본값: `~/.claude/plans/`

### 1.2 Path Traversal 방지

```typescript
const cwd = getCwd()
const resolved = resolve(cwd, settingsDir)
if (!resolved.startsWith(cwd + sep) && resolved !== cwd) {
  logError(new Error(`plansDirectory must be within project root: ${settingsDir}`))
  plansPath = join(getClaudeConfigHomeDir(), 'plans')  // fallback
}
```

설정으로 `../../etc/` 같은 상위 디렉토리를 지정하는 공격을 차단. 실패 시 기본 경로로 폴백.

### 1.3 Memoization
`memoize(getPlansDirectory)` — UI 렌더링(각 파일 도구의 메시지 렌더 시 호출) 에서 매번 `mkdirSync` 가 발생하면 느려진다. 세션 시작 시 한 번 mkdir 하고 캐시된 경로 반환.

## 2. 슬러그 생성 — `getPlanSlug`

### 2.1 단어 기반 슬러그
`generateWordSlug()` 가 `{형용사}-{명사}` 형태 랜덤 슬러그 생성 (예: `curious-sparrow`, `lazy-mountain`).

### 2.2 충돌 회피 루프
```typescript
for (let i = 0; i < MAX_SLUG_RETRIES; i++) {  // MAX = 10
  slug = generateWordSlug()
  const filePath = join(plansDir, `${slug}.md`)
  if (!existsSync(filePath)) break
}
```

기존 파일과 겹치면 최대 10번까지 재생성.

### 2.3 세션 단위 캐시
`getPlanSlugCache()` 는 `Map<SessionId, string>` — 한 세션은 하나의 슬러그 사용. 세션이 바뀌면 새 슬러그가 생성된다.

## 3. 파일 경로 — `getPlanFilePath`

```
메인 세션:      .claude/plans/{slug}.md
서브에이전트:   .claude/plans/{slug}-agent-{agentId}.md
```

`agentId` 가 있으면 파일명에 에이전트 ID 추가 — 서브에이전트가 부모와 **독립된 플랜 파일**을 쓴다. 서브에이전트가 플랜 모드로 돌 때 부모의 플랜을 덮어쓰지 않도록.

## 4. 플랜 읽기 — `getPlan`

```typescript
export function getPlan(agentId?: AgentId): string | null {
  const filePath = getPlanFilePath(agentId)
  try {
    return readFileSync(filePath, { encoding: 'utf-8' })
  } catch (error) {
    if (isENOENT(error)) return null
    logError(error)
    return null
  }
}
```

`ExitPlanMode` 가 플랜 내용을 노출할 때 사용. `ENOENT` 는 정상 (아직 작성 안됨) — 에러 로깅 생략.

## 5. 세션 재개 — `copyPlanForResume`

[src/utils/plans.ts:164-231](../../src/utils/plans.ts#L164-L231)

세션을 재개할 때 플랜 슬러그와 파일을 복원:

```
1. log 에서 slug 추출 (getSlugFromLog)
2. setPlanSlug(sessionId, slug) — 캐시에 세팅
3. 플랜 파일 존재 확인
   ├─ 있음 → return true
   └─ 없음 (ENOENT)
       ├─ 로컬 환경 → return false (파일이 원래 없어야 함)
       └─ 원격 환경 (CCR) → 복구 시도
           ├─ 파일 스냅샷에서 복구 (`findFileSnapshotEntry(messages, 'plan')`)
           └─ 메시지 히스토리에서 복구 (`recoverPlanFromMessages`)
       복구 성공 → writeFile 후 return true
```

### 5.1 왜 복구가 필요한가
CCR (원격 세션) 은 서버리스 환경이라 세션 간 디스크가 보존되지 않는다. 플랜 파일이 사라진 상태로 재개될 수 있으므로:
- **File snapshot**: 세션 중 주기적으로 메시지 스트림에 저장된 스냅샷 (`SystemFileSnapshotMessage`)
- **Message recovery**: ExitPlanMode 호출 시 메시지에 포함된 플랜 내용 추출

로컬 환경에서는 파일이 정말로 없는 것이므로 복구 시도 생략 — 없는 것을 복구하려 하면 혼란만.

## 6. 포크 — `copyPlanForFork`

```typescript
const newSlug = getPlanSlug(targetSessionId)  // 새 슬러그 생성
const newPlanPath = join(plansDir, `${newSlug}.md`)
// 원본 → 새 파일로 복사
```

포크된 세션은 원본과 슬러그를 **공유하지 않는다**. 같은 파일을 두 세션이 쓰면 서로 덮어쓰기 때문. `copyFile` 로 완전 분리된 파일 생성.

## 7. `persistFileSnapshotIfRemote`

[src/utils/plans.ts:360+](../../src/utils/plans.ts#L360)

원격 환경에서만 동작. 현재 플랜 내용을 `SystemFileSnapshotMessage` 로 **메시지 스트림에 기록** 하여 세션 로그에 영속화.

### 호출 지점
- `EnterPlanMode` 이후 첫 플랜 작성 시
- `ExitPlanMode` 가 편집된 플랜을 디스크에 기록한 직후 (재스냅샷)
- 기타 플랜 수정 지점

**역할**: 파일시스템에만 의존하지 않는 플랜 복원 경로 확보. `copyPlanForResume` 의 `findFileSnapshotEntry` 가 이것을 찾는다.

## 8. `/clear` 와 슬러그 삭제

```typescript
clearPlanSlug(sessionId?)      // 현재 세션의 슬러그만 제거
clearAllPlanSlugs()            // 모든 세션 (서브에이전트 포함) 슬러그 제거
```

`/clear` 커맨드가 컨텍스트를 지울 때 `clearAllPlanSlugs()` 를 호출해 서브에이전트들의 슬러그까지 청소.

## 9. 파일 수명 주기 요약

```
세션 시작
  ↓ getPlanSlug(sessionId) → "curious-sparrow" 캐시 저장
  ↓ 파일은 아직 없음 (lazy)

LLM 이 EnterPlanMode 호출
  ↓ 모드 전환만 — 파일은 아직 없음

LLM 이 Edit/Write 로 플랜 파일 생성
  ↓ .claude/plans/curious-sparrow.md 작성
  ↓ persistFileSnapshotIfRemote() — 원격이면 메시지에도 기록

ExitPlanMode 호출
  ↓ getPlan() 으로 디스크에서 읽어 승인 대화상자에 표시
  ↓ (CCR 편집 시) 편집된 버전으로 파일 재작성
  ↓ persistFileSnapshotIfRemote() 재실행
  ↓ 사용자 승인

다음 턴 (구현 단계)
  ↓ 플랜 파일은 남아 있음 — 참조 가능
  ↓ VerifyPlanExecution 도구가 이 파일을 기반으로 배경 검증 수행

/clear
  ↓ clearAllPlanSlugs() — 슬러그 캐시 비우기
  ↓ 다음 세션은 새 슬러그
```

## 10. 관련 상수·이벤트

- 분석 이벤트: `tengu_plan_exit` (플랜 크기 추적용 — `planLengthChars`)
- 분석 이벤트: `tengu_exit_plan_mode_called_outside_plan` (오용 감지)
- 환경 종류 판정: `getEnvironmentKind() === null` ↔ 로컬 CLI
