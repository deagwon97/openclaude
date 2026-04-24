# 05. 시스템 프롬프트 구성

## 관련 파일

- `src/utils/api.ts` — 시스템 프롬프트 조립, 컨텍스트 주입
- `src/context.ts` — Git/날짜 컨텍스트 수집

## 시스템 프롬프트 구조

```typescript
type SystemPromptBlock = {
  text: string
  cacheScope: 'global' | 'org' | null  // 프롬프트 캐시 범위
}

// 시스템 프롬프트 = SystemPromptBlock[]
// 각 블록이 별도 캐시 단위로 관리됨
```

## 컨텍스트 수집

### 시스템 컨텍스트 (Git 상태)

파일: `src/context.ts`

```typescript
getSystemContext() → memoized
// 포함 내용:
//   - 현재 브랜치
//   - 기본 브랜치 (main/master)
//   - Git 사용자 이름
//   - git status (최대 2K 문자)
//   - 최근 5개 커밋 해시 + 메시지
//   - cacheBreaker?: "[CACHE_BREAKER: ...]"  // 캐시 무효화용
```

### 사용자 컨텍스트 (CLAUDE.md)

```typescript
getUserContext() → memoized
// 포함 내용:
//   - CLAUDE.md 내용 (프로젝트 지침)
//   - 현재 날짜 ("Today's date is YYYY-MM-DD.")
```

## 컨텍스트 주입 방식

### 사용자 컨텍스트 주입

```typescript
prependUserContext(messages, userContext) → Message[]
// 첫 번째 user 메시지 앞에 컨텍스트 블록 삽입
// CLAUDE.md 내용과 날짜가 대화 시작 전에 제공됨
```

### 시스템 컨텍스트 주입

```typescript
appendSystemContext(systemPrompt, systemContext) → string[]
// 시스템 프롬프트 배열 끝에 Git 상태 블록 추가
// 결과:
//   [...기존 시스템 프롬프트 블록들, "gitStatus: ...\nStatus:\n..."]
```

## 최종 API 요청 파라미터

파일: `src/query.ts`

```typescript
type QueryParams = {
  messages: Message[]                    // 정규화된 대화 히스토리
  systemPrompt: SystemPrompt             // 블록 배열
  userContext: { [k: string]: string }   // CLAUDE.md, 날짜
  systemContext: { [k: string]: string } // Git 상태
  canUseTool: CanUseToolFn               // 권한 확인 함수
  toolUseContext: ToolUseContext         // Tool 메타데이터
  fallbackModel?: string
  querySource: QuerySource               // 출처 추적 (REPL, SubAgent 등)
  maxOutputTokensOverride?: number
  maxTurns?: number
  taskBudget?: { total: number }         // Agent 작업 예산
}
```

## 프롬프트 캐시 전략

시스템 프롬프트 블록별로 캐시 범위를 설정하여 비용을 절감합니다:

```
cacheScope: 'global'
  → 여러 사용자가 공유 (조직 전체 공통 지침)

cacheScope: 'org'
  → 조직 내 공유

cacheScope: null
  → 캐시 없음 (매 요청마다 새로 전송)
```

캐시 TTL:
- **Ephemeral 캐시**: 5분 또는 1시간 (모델 및 자격에 따라 다름)
- Auto-Compact 이후에도 캐시가 유지되도록 설계됨

## 섹션 캐시

`state.systemPromptSectionCache` 에 섹션별로 캐싱하여 반복 재계산 방지:

```typescript
Map<string, string | null>
// key: 섹션 식별자 (e.g., "git-status", "claude-md")
// value: 캐싱된 문자열 또는 null (해당 섹션 없음)
```
