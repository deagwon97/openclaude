# 02. 설정 로드와 매칭

## 1. 설정 로드 계층

Hook 정의는 세 단계로 수집된다.

```
user settings (~/.claude/settings.json)
         │
         ▼
project settings (<repo>/.claude/settings.json)
         │
         ▼
local settings (<repo>/.claude/settings.local.json)
         +
   plugin / skill hooks (registerFrontmatterHooks, registerSkillHooks)
         +
   session hooks (sessionHooks.ts – 메모리 기반 임시 등록)
```

관련 파일:

- [src/utils/hooks/hooksSettings.ts](../../src/utils/hooks/hooksSettings.ts) — `getAllHooks()`, `getHooksForEvent()`
- [src/utils/hooks/registerFrontmatterHooks.ts](../../src/utils/hooks/registerFrontmatterHooks.ts) — 플러그인 frontmatter에서 hook 등록
- [src/utils/hooks/registerSkillHooks.ts](../../src/utils/hooks/registerSkillHooks.ts) — 스킬 정의 안의 hook 등록
- [src/utils/hooks/sessionHooks.ts](../../src/utils/hooks/sessionHooks.ts) — 세션 내 동적 등록

## 2. 설정 스냅샷 (captureHooksConfigSnapshot)

앱 시작 직후 **스냅샷**을 찍어두고 이후에는 그 스냅샷만 사용한다. 이유는 두 가지:

1. 실행 도중 사용자가 settings.json을 편집해 악성 hook을 주입하는 것을 막는다.
2. 세션 전체에서 일관된 hook 동작을 보장한다.

```
src/utils/hooks/hooksConfigSnapshot.ts
├── captureHooksConfigSnapshot()      # 시작 시 1회 호출
├── getHooksConfigFromSnapshot()      # 이벤트 발생 시 참조
└── shouldAllowManagedHooksOnly()     # policy에 따라 managed 전용 모드
```

정책상 `managed-only` 모드에서는 플러그인/관리자가 등록한 hook만 통과시키고, 사용자 settings의 hook은 차단된다.

## 3. Workspace Trust 검증

대화형 모드에서는 `shouldSkipHookDueToTrust()`가 각 hook을 필터링한다. 신뢰되지 않은 워크스페이스의 project/local settings에서 온 hook은 조용히 스킵되고 `diag_log`에만 기록된다. non-interactive(자동화) 모드에서는 명시적 플래그가 있어야만 실행된다.

## 4. Matcher 필터링

[src/utils/hooks.ts](../../src/utils/hooks.ts)의 `matchesPattern(matchQuery, matcher)` 함수가 사용된다.

| matcher 형태 | 예시 | 동작 |
|-------------|------|------|
| 빈 문자열 / `*` | `""` | 모든 경우 매칭 |
| 심플 문자열 | `"Write"` | `tool_name === "Write"` 정확 매칭 |
| 파이프 리스트 | `"Read\|Edit\|Write"` | 포함된 이름 중 하나와 정확 매칭 |
| 정규식 | `"^mcp__.*"` | `new RegExp(matcher).test(tool_name)` |

구현 개요:

```typescript
function matchesPattern(matchQuery: string, matcher: string): boolean {
  if (!matcher || matcher === '*') return true

  if (/^[a-zA-Z0-9_|]+$/.test(matcher)) {
    const patterns = matcher.split('|').map(p => normalizeLegacyToolName(p.trim()))
    return patterns.includes(matchQuery)
  }

  try {
    const regex = new RegExp(matcher)
    if (regex.test(matchQuery)) return true
    for (const legacy of getLegacyToolNames(matchQuery)) {
      if (regex.test(legacy)) return true
    }
    return false
  } catch {
    return false
  }
}
```

> `normalizeLegacyToolName`은 `FileWriteTool` ↔ `Write` 같은 과거 네이밍 호환을 맞춰준다.

## 5. `if` 조건 (권한 규칙 문법)

Tool 관련 이벤트(`PreToolUse`, `PostToolUse`, `PermissionRequest`)에서만 사용 가능한 추가 필터. 문법은 권한 규칙과 동일한 `ToolName(pattern)` 형태다.

```
if: "Bash(git rebase *)"        # Bash 호출이고 command가 "git rebase *" 패턴
if: "Write(src/**/*.ts)"         # Write가 src 하위 ts 파일을 대상으로 할 때
if: "Read(*.env)"                # Read가 .env 파일을 대상으로 할 때
```

처리 과정:

1. `prepareIfConditionMatcher(hookInput, tools)` — 이벤트당 1회 준비
2. 각 tool의 `preparePermissionMatcher()` 호출
   - `Bash`는 tree-sitter로 커맨드 구문 트리를 파싱
   - `Read/Write/Edit`는 경로 glob 매칭
3. 매칭 결과를 캐시 → 여러 hook이 같은 조건 평가 시 재사용

**핵심 최적화**: `if` 조건에서 떨어지는 hook은 프로세스 스폰 없이 미리 제거된다. tree-sitter 파싱 같은 비싼 작업은 이벤트당 1회만 수행된다.

## 6. 중복 제거

여러 설정 계층을 머지하면 동일한 hook이 중복될 수 있다. `getHooksConfig()` 단계에서 namespaced key로 중복을 제거한다.

**Dedup key 구성요소:**

```
pluginRoot (또는 skillRoot) || ''
 + hook.shell
 + hook.command
 + hook.if
```

- 같은 key → 마지막 계층 것만 남김
- 다른 `shell` 값이면 별개 hook으로 취급
- `function`/`callback` 타입 hook(SDK 내부용)은 dedup 대상에서 제외

## 7. 실행 후보 선정 플로우 요약

```
(이벤트 발생)
    ▼
getHooksConfigFromSnapshot(event)
    ▼
shouldAllowManagedHooksOnly() 필터
    ▼
shouldSkipHookDueToTrust() 필터
    ▼
matchesPattern(matcher) 필터           ← 문자열/정규식
    ▼
prepareIfConditionMatcher() 1회 준비
    ▼
if-condition 필터                      ← 권한 규칙 문법
    ▼
dedup by (pluginRoot + shell + command + if)
    ▼
→ 실행 대상 hook 리스트
```

이후 리스트는 [03_execution_engine.md](03_execution_engine.md)에서 다루는 파이프라인으로 전달된다.
