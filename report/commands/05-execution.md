# 05 · 실행 (Routing → Claude Message)

사용자가 `/mycommand arg1 arg2`를 입력한 순간부터 Claude API에 메시지가 전달될 때까지의 흐름입니다.

## 전체 호출 스택

```
REPL 입력
  ↓
handlePromptSubmit.handlePromptSubmit()
  ├─ skipSlashCommands 플래그 확인
  ├─ 즉시 처리 가능한 local-jsx 커맨드 (immediate flag) 처리
  └─ executeUserInput()
      ↓
processUserInput.processUserInputBase()
  ├─ 이미지/첨부 처리
  └─ processSlashCommand() 분기 (입력이 '/'로 시작하면)
      ↓
processSlashCommand.processSlashCommand()
  ├─ slashCommandParsing.parseSlashCommand()   ← {commandName, args, isMcp}
  ├─ 레지스트리에서 command 조회
  │    └─ 없으면 평문 프롬프트로 폴백
  └─ getMessagesForSlashCommand(command, args, ctx)
      ├─ type === 'local'      → mod.call(args, ctx)
      ├─ type === 'local-jsx'  → React UI + onDone 콜백
      └─ type === 'prompt'
           ├─ command.context === 'fork'
           │    └─ executeForkedSlashCommand() — 서브에이전트에서 실행
           └─ (inline, 기본값)
                └─ getMessagesForPromptSlashCommand()
                     ├─ command.getPromptForCommand(args, ctx)  ← 치환/쉘/변수
                     ├─ registerSkillHooks()                     ← hooks frontmatter
                     └─ messages = [ ... ] 조립
  ↓
query() → Claude API 호출
```

## 주요 파일과 함수

| 기능 | 파일 | 함수 |
|------|------|------|
| 입력 수신 | [src/utils/handlePromptSubmit.ts](../../src/utils/handlePromptSubmit.ts) | `handlePromptSubmit()` |
| 입력 분기 | [src/utils/processUserInput/processUserInput.ts](../../src/utils/processUserInput/processUserInput.ts) | `processUserInputBase()` |
| 슬래시 파싱 | [src/utils/slashCommandParsing.ts](../../src/utils/slashCommandParsing.ts) | `parseSlashCommand()` |
| 슬래시 실행 | [src/utils/processUserInput/processSlashCommand.tsx](../../src/utils/processUserInput/processSlashCommand.tsx) | `processSlashCommand()` · `getMessagesForSlashCommand()` · `getMessagesForPromptSlashCommand()` · `executeForkedSlashCommand()` |

## 1단계 — `parseSlashCommand()`

```ts
// 일반 커맨드
'/search foo bar'
  → { commandName: 'search', args: 'foo bar', isMcp: false }

// MCP 커맨드 (이름에 ' (MCP)' 접미사)
'/mcp:tool (MCP) arg1'
  → { commandName: 'mcp:tool (MCP)', args: 'arg1', isMcp: true }
```

공백으로 커맨드명과 인수를 분리합니다. MCP 커맨드는 이름 형식이 다르므로 별도 처리됩니다.

## 2단계 — 레지스트리 조회와 분기

`processSlashCommand()`는 파싱된 `commandName`으로 `loadAllCommands()` 결과에서 커맨드를 찾습니다. 찾지 못하면 `/mycommand`가 오타였거나 존재하지 않는 것으로 간주하고 **평문 프롬프트**로 폴백합니다.

찾았다면 `type`에 따라 세 갈래로 나뉩니다:

### 2-A. `type: 'local'` (코드 기반 동기 실행)

```ts
const mod = await command.load()
const result = await mod.call(args, context)  // { type: 'text' | 'compact' | 'skip', content }
// result를 시스템 메시지로 반환, Claude 호출 없이 UI에 즉시 표시
```

### 2-B. `type: 'local-jsx'` (React UI 렌더링)

```ts
const mod = await command.load()
const element = mod.call(onDone, context, args)
// UI 렌더링 후 사용자가 완료 → onDone(result, options) 호출
// options.display: 'skip' | 'system' | 'user'
```

### 2-C. `type: 'prompt'` (**커스텀 `.md` 커맨드의 기본**)

여기서 다시 `command.context === 'fork'` 여부로 갈라집니다:

#### Fork 모드 — `executeForkedSlashCommand()`

- 별도 서브에이전트(Task)를 띄워 격리된 컨텍스트에서 실행
- 진행 상황 UI 표시
- 메인 대화에 결과 요약만 반환 — 긴 작업의 컨텍스트 오염 방지

#### Inline 모드 (기본) — `getMessagesForPromptSlashCommand()`

아래 3단계에서 상세히 다룹니다.

## 3단계 — Inline Prompt 커맨드 메시지 조립

```ts
// 1) 스킬 본문을 동적으로 생성
const result = await command.getPromptForCommand(args, context)
//   ↑ 내부에서 substituteArguments + shell 실행 + 환경변수 치환 수행

// 2) frontmatter의 hooks를 현재 세션에 등록
if (command.hooks && hooksAllowed) {
  registerSkillHooks(context.setAppState, sessionId, command.hooks, ...)
}

// 3) 최종 메시지 배열 구성
return {
  messages: [
    createUserMessage({ content: metadata }),              // <command-message>, <command-name>, <command-args>
    createUserMessage({ content: result, isMeta: true }),  // 스킬 본문 — 모델만 보임
    ...attachmentMessages,                                  // @file mention 처리 결과
    createAttachmentMessage({
      type: 'command_permissions',
      allowedTools: additionalAllowedTools,                 // frontmatter의 allowed-tools
      model: command.model,                                 // 모델 오버라이드
    }),
  ],
  shouldQuery: true,
  allowedTools,
  model,
  effort,
  command,
}
```

### Claude에게 전달되는 메시지 모양

```xml
<!-- 첫 번째 user 메시지 (사용자에게도 표시됨) -->
<command-message>mycommand</command-message>
<command-name>/mycommand</command-name>
<command-args>arg1 arg2</command-args>

<!-- 두 번째 user 메시지 (isMeta: true — 사용자 UI에 숨김) -->
[치환 완료된 스킬 본문 마크다운]

<!-- 첨부: @README.md 등 mention 처리 결과 -->
[attachment 메시지들]

<!-- 첨부: command_permissions 메타데이터 -->
{ allowedTools, model }
```

`isMeta: true`는 **"모델에게는 보이지만 사용자 UI에는 표시되지 않는"** 메시지 플래그입니다. 스킬 본문이 대화 기록을 시각적으로 어지럽히지 않도록 하는 장치입니다.

## 권한 및 모델 오버라이드

frontmatter의 `allowed-tools`와 `model`은 `command_permissions` 첨부를 통해 `query()`까지 전달됩니다. 이 커맨드의 실행 동안만:

- **허용 도구**: 평소엔 막혀 있던 도구라도 `allowed-tools`에 있으면 임시 허용
- **모델**: 이 커맨드 쿼리에 한해 다른 모델(예: sonnet)로 실행

## Hooks 등록

`command.hooks`가 있으면 `registerSkillHooks()`가 현재 세션의 훅 상태에 임시 등록합니다. 이 훅은 해당 커맨드 실행 컨텍스트 안에서만 살아 있습니다.

## 관련 문서

- 이전: [04-registration.md](04-registration.md)
- 다음: [06-special-syntax.md](06-special-syntax.md) — `$ARGUMENTS`, 쉘 실행, `@file`, `paths:`
