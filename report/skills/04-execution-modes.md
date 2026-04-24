# 04. 실행 모드: Inline vs Forked

Skill 의 `context` frontmatter 필드가 실행 방식을 결정한다.

| 모드 | 기본값 | 의미 | 토큰 예산 | 컨텍스트 | 적합한 작업 |
|---|---|---|---|---|---|
| `inline` | ✅ | markdown 본문을 **현재 대화에 메시지로 주입** | 공유 | 동일 | 짧고 빠른 보조 작업, 템플릿 |
| `fork` | — | **별도 sub-agent 를 스폰** 해서 실행 | 독립 | 분리 | 복잡하고 자율성이 필요한 작업 |

## 4.1 Inline 실행

진입점은 [processSlashCommand.tsx](../../src/utils/processUserInput/processSlashCommand.tsx) 의 `processPromptSlashCommand()` → `getMessagesForPromptSlashCommand()`.

수행 단계:

1. `command.getPromptForCommand(args, context)` 호출 — markdown 본문 렌더 (치환 처리)
2. Skill 에 `hooks` 가 있고 허용되는 경우 `registerSkillHooks()` 로 세션 훅 등록
3. 아래 형태의 메시지들을 현재 대화에 삽입:
   - `createUserMessage({ content: metadata })` — skill 메타 정보
   - `createUserMessage({ content: mainMessageContent, isMeta: true })` — skill 본문
   - attachment messages (참조 파일 등)
   - `createAttachmentMessage({ type: 'command_permissions', allowedTools: ... })` — 이 턴 한정 tool 허용 목록
4. 반환: `{ messages, shouldQuery: true, allowedTools, model, effort, command }`

**중요한 특징**: inline skill 은 "현재 모델과 현재 컨텍스트" 에서 계속 실행되며, 메시지 히스토리에 그대로 남는다. Skill 의 `allowed-tools` 는 **그 턴에 한해** 병합된 tool 풀로 적용된다.

## 4.2 Forked 실행

진입점은 [SkillTool.ts](../../src/tools/SkillTool/SkillTool.ts) 의 `executeForkedSkill()`.

```ts
async function executeForkedSkill(
  command, commandName, args, context, canUseTool, parentMessage, onProgress,
): Promise<ToolResult<Output>> {
  const agentId = createAgentId()

  // 1. 서브에이전트용 독립 컨텍스트 준비
  const { modifiedGetAppState, baseAgent, promptMessages, skillContent } =
    await prepareForkedCommandContext(command, args || '', context)

  // 2. effort 오버라이드 병합
  const agentDefinition =
    command.effort !== undefined
      ? { ...baseAgent, effort: command.effort }
      : baseAgent

  // 3. runAgent() 로 서브에이전트 구동
  for await (const message of runAgent({
    agentDefinition,
    promptMessages,
    toolUseContext: { ...context, getAppState: modifiedGetAppState },
    canUseTool,
    isAsync: false,
    querySource: 'agent:custom',
    model: command.model as ModelAlias | undefined,
    availableTools: context.options.tools,
    override: { agentId },
  })) {
    // 4. 각 assistant/user 메시지마다 parent 에게 skill_progress 보고
    if ((message.type === 'assistant' || message.type === 'user') && onProgress) {
      onProgress({
        toolUseID: `skill_${parentMessage.message.id}`,
        data: { message: m, type: 'skill_progress', prompt: skillContent, agentId },
      })
    }
  }

  // 5. 최종 결과 텍스트만 추출해서 상위에 돌려줌
  return {
    data: { success: true, commandName, status: 'forked', agentId, result: resultText },
  }
}
```

핵심 포인트:

- **독립 에이전트로 실행** — `runAgent()` 는 에이전트 정의, 별도 메시지 히스토리, 자체 권한으로 돌린다.
- 진행 상황은 `onProgress(type: 'skill_progress')` 로 상위에 스트리밍되어 UI 에 표시된다.
- 최종적으로 상위 대화에 돌아가는 것은 **서브에이전트가 생성한 텍스트 결과 한 덩어리** 뿐이다. 중간 tool call 들은 부모 컨텍스트를 오염시키지 않는다.
- `command.model`, `command.effort`, `command.agent` frontmatter 로 서브에이전트의 모델과 사고 노력 수준을 직접 고를 수 있다.

## 4.3 언제 fork 를 써야 하나

| 상황 | 추천 모드 |
|---|---|
| 짧은 보조 작업 (커밋 메시지 생성 등) | `inline` |
| 정형화된 템플릿 주입 | `inline` |
| 컨텍스트 오염 없이 긴 조사를 하고 싶음 | `fork` |
| 메인 대화보다 작은/빠른 모델로 돌리고 싶음 | `fork` + `model: haiku` |
| 여러 tool 을 자율적으로 쓰는 복잡 워크플로 | `fork` |
| Main agent 와 다른 권한/시스템 프롬프트가 필요 | `fork` + `agent: <type>` |
