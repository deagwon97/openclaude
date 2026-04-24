# 03. `Skill` tool 구현과 호출 파이프라인

## 3.1 `SkillTool` 이란

모델은 `.claude/skills` 의 skill 을 **직접 호출하지 않는다**. 대신 `Skill` 이라는 이름의 단일 tool 을 통해 skill 이름과 인자를 넘긴다. 이 tool 정의는 [SkillTool.ts](../../src/tools/SkillTool/SkillTool.ts) 에 있다.

```ts
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: 'Skill',
  description: async ({ skill }) => `Execute skill: ${skill}`,
  prompt: async () => getPrompt(getProjectRoot()),

  async validateInput({ skill }, context): Promise<ValidationResult>
  async checkPermissions({ skill, args }, context): Promise<PermissionDecision>
  async call({ skill, args }, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
})
```

입력 스키마는 단순하다:

```ts
z.object({
  skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
  args:  z.string().optional().describe('Optional arguments for the skill'),
})
```

## 3.2 모델에게 Skill 목록이 전달되는 방식

Skill 자체는 각각의 tool 로 노출되지 않는다. 대신 **SkillTool 의 프롬프트 본문에 사용 가능한 skill 목록을 문자열로 끼워 넣는 방식** 이다.

1. [commands.ts](../../src/commands.ts) 의 `getSkillToolCommands(cwd)` 가 현재 사용 가능한 skill 을 필터링한다.
   - `type === 'prompt'`
   - `!disableModelInvocation`
   - `source !== 'builtin'`
   - `loadedFrom` 이 `bundled` / `skills` / `commands_DEPRECATED` 중 하나이거나, `hasUserSpecifiedDescription` 또는 `whenToUse` 가 있음
2. [SkillTool/prompt.ts](../../src/tools/SkillTool/prompt.ts) 의 `formatCommandsWithinBudget(commands, contextWindowTokens)` 가 이 목록을 **토큰 예산 안에서** 포맷한다.
   - 예산: `contextWindowTokens * 4 * 0.01` (컨텍스트의 1%), 기본 8,000 자
   - `process.env.SLASH_COMMAND_TOOL_CHAR_BUDGET` 로 override 가능
   - Bundled skill 은 전체 description 을 유지하고, 나머지는 우선순위순으로 잘린다
3. 결과 문자열이 [constants/prompts.ts](../../src/constants/prompts.ts) 의 시스템 프롬프트 (`getSessionSpecificGuidanceSection`) 에 주입된다.

즉, 모델이 보는 것은 "사용 가능한 skills: `commit`, `review-pr`, `pdf` ..." 형태의 텍스트이며, 호출 자체는 `Skill(skill="commit", args="...")` 로 한다.

## 3.3 호출 파이프라인

```
Model 이 SkillTool 호출 생성  (skill: "commit", args: "...")
      ↓
SkillTool.validateInput()
  ├─ skill 이름이 실제 Command 로 존재하는가
  ├─ disableModelInvocation 여부
  └─ type === 'prompt' 인가
      ↓
SkillTool.checkPermissions()
  ├─ deny rule 매칭 → deny
  ├─ allow rule 매칭 → allow
  ├─ "safe properties" 만 쓰는 skill → 자동 allow
  └─ 그 외 → ask (사용자 승인 요청)
      ↓
SkillTool.call()
  ├─ command.context === 'fork' ?
  │     ├─ YES → executeForkedSkill()   → [04] 참조
  │     └─ NO  → processPromptSlashCommand() → inline 메시지 생성
  ↓
ToolResult 반환
  {
    success, commandName,
    status: 'inline' | 'forked',
    newMessages?: ..., contextModifier?: ...,  // inline
    agentId?, result?: ...,                    // forked
  }
```

## 3.4 "Safe properties" 자동 허용

사용자 승인 없이 자동 실행되려면 skill 이 아래 속성만 가져야 한다:

- `description`, `when_to_use`, `argument-hint`
- `user-invocable: false`

아래 속성 중 하나라도 있으면 **사용자 승인을 요청** 한다 (권한이 확장되는 속성들):

- `allowed-tools` — 추가 tool 권한 상승
- `model` — 모델 오버라이드
- `context: fork` — 별도 서브에이전트
- `hooks` — 이벤트 훅 등록

## 3.5 `validateInput` 의 특이사항

`validateInput()` 은 단순히 입력 형태만 보는 게 아니라 `getCommands(cwd)` 를 조회해서 해당 skill 이 **현재 로드되어 있는지** 를 확인한다. 로드되지 않았거나 `disableModelInvocation: true` 인 경우 실패한다.

> 참고: `validateInput()`, `checkPermissions()`, `call()` 의 구체적 줄번호는 [06-file-map.md](06-file-map.md) 참조.
