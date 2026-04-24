# 05. Hook 등록과 권한 체크

Skill 의 `hooks` 와 `allowed-tools` 는 실행 시점에 **세션 상태로 주입** 된다. 이 글은 그 주입 메커니즘과 권한 처리 방식을 정리한다.

## 5.1 Skill Hook 등록

Skill frontmatter 에 `hooks` 가 있으면, inline 실행 시 `registerSkillHooks()` 가 세션 훅으로 등록한다.

```yaml
---
description: Auto-lint after edit
hooks:
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - action: log
        - command: "npm run lint"
---
```

등록 코드 ([registerSkillHooks.ts](../../src/utils/hooks/registerSkillHooks.ts)):

```ts
export function registerSkillHooks(
  setAppState, sessionId, hooks, skillName, skillRoot,
): void {
  for (const eventName of HOOK_EVENTS) {
    const matchers = hooks[eventName]
    if (!matchers) continue
    for (const matcher of matchers) {
      for (const hook of matcher.hooks) {
        const onHookSuccess = hook.once
          ? () => removeSessionHook(setAppState, sessionId, eventName, hook)
          : undefined
        addSessionHook(
          setAppState, sessionId, eventName,
          matcher.matcher || '', hook, onHookSuccess, skillRoot,
        )
      }
    }
  }
}
```

특징:

- **세션 스코프** — `settings.json` 의 영구 훅과는 별개로, 현재 세션의 `SessionHooksState` 맵에만 추가된다.
- `hook.once: true` 면 한 번 실행된 뒤 자동 제거 (`removeSessionHook`).
- `skillRoot` 를 함께 저장해서, 훅 커맨드 안에서 skill 디렉토리를 참조할 수 있다.
- 등록 대상 이벤트는 `HOOK_EVENTS` 상수: `PreToolUse`, `PostToolUse`, `PreSubmit`, `PostMessage`, `Stop` 등.

세션 훅 저장 구조:

```ts
type SessionHooksState = Map<string, SessionStore>

type SessionStore = {
  hooks: { [event in HookEvent]?: SessionHookMatcher[] }
}

type SessionHookMatcher = {
  matcher: string
  skillRoot?: string
  hooks: Array<{
    hook: HookCommand | FunctionHook
    onHookSuccess?: OnHookSuccess
  }>
}
```

## 5.2 SkillTool 의 권한 체크 (`checkPermissions`)

Skill 호출 시 [SkillTool.ts](../../src/tools/SkillTool/SkillTool.ts) 의 `checkPermissions()` 가 단계적으로 결정한다.

```
1. Deny rules 조회 → 매칭되면 즉시 deny
2. Allow rules 조회 → 매칭되면 allow
3. Skill 이 "safe properties" 만 쓰는가? → allow
4. 그 외 → ask (사용자 승인 요청)
```

**Safe properties (승인 없이 자동 허용)**

- `description`, `when_to_use`, `argument-hint` 만 설정
- `user-invocable` 만 설정
- 기타 권한 상승 속성이 없음

**Non-safe properties (승인 필요)**

- `allowed-tools` — 추가 tool 접근
- `model` — 모델 오버라이드
- `context: fork` — 별도 에이전트
- `hooks` — 세션 훅 등록

## 5.3 `allowed-tools` 의 적용 범위

Skill 이 선언한 `allowed-tools` 는 **skill 실행 동안만** 유효하다.

- **Inline 모드**: 메시지에 `command_permissions` attachment 가 붙어서, 해당 턴의 tool 풀이 병합된다. 다음 사용자 입력이 들어오면 원래 권한으로 돌아간다.
- **Fork 모드**: `runAgent()` 호출 시 서브에이전트의 tool 풀로 건네지며, 부모 대화의 권한과는 완전히 분리된다.

## 5.4 실제 호출 흐름에서 5장 내용의 위치

```
SkillTool.call()
     │
     ├─ checkPermissions() ← 5.2
     │     ↓
     ├─ inline?
     │   └─ getMessagesForPromptSlashCommand()
     │         ├─ registerSkillHooks()           ← 5.1
     │         └─ attach command_permissions    ← 5.3 (inline)
     │
     └─ fork?
         └─ executeForkedSkill()
               └─ runAgent({ availableTools }) ← 5.3 (fork)
```
