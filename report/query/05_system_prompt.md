# 05. 시스템 프롬프트 내부 구조

이 문서는 OpenClaude 가 Claude API 로 보내는 `system` 블록이 **어떤 섹션들로 조립되는지**, **어디에 캐시 경계가 있는지**, **모드별로 어떻게 달라지는지**를 정리합니다.

## 관련 파일

- [src/constants/prompts.ts](../../src/constants/prompts.ts) — `getSystemPrompt()` 와 개별 섹션 빌더
- [src/utils/api.ts](../../src/utils/api.ts) — `splitSysPromptPrefix`, `appendSystemContext`, `prependUserContext`
- [src/services/api/claude.ts](../../src/services/api/claude.ts) — `buildSystemPromptBlocks` (최종 직렬화)
- [src/context.ts](../../src/context.ts) — `systemContext` 수집 (git, date, env)

---

## 전체 조립 파이프라인

```
getSystemPrompt(tools, model, ...)      ← src/constants/prompts.ts
        │
        ▼  string[] = [
            intro, system, doingTasks, actions, usingTools,
            tone, efficiency,
            SYSTEM_PROMPT_DYNAMIC_BOUNDARY,    ← 경계 마커
            sessionGuidance, memory, envInfo, language,
            outputStyle, mcp, scratchpad, frc, summarizeTool, ...
          ]
        │
        ▼
appendSystemContext(prompt, systemContext)   ← src/utils/api.ts
        │  배열 끝에 "key: value\n..." 한 덩어리 추가
        ▼
splitSysPromptPrefix(prompt)                 ← src/utils/api.ts
        │  경계 마커 기준으로
        │  static(global cache) / dynamic(no cache) 로 분할
        ▼
buildSystemPromptBlocks(prompt, cachingEnabled, ...)  ← claude.ts
        │  TextBlockParam[] 로 직렬화 + cache_control 부착
        ▼
anthropic.beta.messages.stream({ system: [...] })
```

---

## `getSystemPrompt()` — 동적 섹션 조립

[src/constants/prompts.ts](../../src/constants/prompts.ts) 의 `getSystemPrompt()` 가 전체 조립을 지휘합니다. 가장 가벼운 분기는 `CLAUDE_CODE_SIMPLE` 환경변수가 켜진 경우로, 한 줄짜리 minimal prompt 만 반환합니다.

```typescript
export async function getSystemPrompt(
  tools: Tools,
  model: string,
  additionalWorkingDirectories?: string[],
  mcpClients?: MCPServerConnection[],
): Promise<string[]> {
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    return [
      `You are OpenClaude, an open-source coding agent and CLI.\n\n` +
      `CWD: ${getCwd()}\nDate: ${getSessionStartDate()}`,
    ]
  }
  ...
}
```

일반 모드에서는 정적 섹션과 동적 섹션을 분리해서 모읍니다. 중요한 것은 **순서**와 **경계 마커**의 위치입니다.

```typescript
return [
  // ── Static content (global 캐시 대상) ────────────────────
  getSimpleIntroSection(outputStyleConfig),
  getSimpleSystemSection(),
  getSimpleDoingTasksSection(),
  getActionsSection(),
  getUsingYourToolsSection(enabledTools),
  getSimpleToneAndStyleSection(),
  getOutputEfficiencySection(),

  // ── BOUNDARY MARKER — 이 줄을 옮기거나 지우지 말 것 ──────
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),

  // ── Dynamic content (캐시 안 함) ─────────────────────────
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

경계 마커 자체는 나중에 `splitSysPromptPrefix` 가 기준점으로만 사용하고, 실제 API 전송에서는 버려집니다.

```typescript
// src/constants/prompts.ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

---

## 정적 섹션 목록 (경계 마커 "이전")

정적 섹션은 **실행 환경에 상관없이 내용이 고정**되어 있어서 프롬프트 캐시를 태울 수 있는 부분입니다. 각 섹션은 프롬프트 작성 의도가 뚜렷하게 분리되어 있습니다.

| 섹션 | 빌더 함수 | 담기는 내용 |
|------|----------|------------|
| Intro | `getSimpleIntroSection(outputStyleConfig)` | "You are an interactive agent...", 사이버 위험 경고, URL 생성 금지 |
| System | `getSimpleSystemSection()` | 출력 텍스트가 사용자에게 보이는 방식, permission mode, `<system-reminder>` 태그 의미, hooks, auto-compact 고지 |
| Doing Tasks | `getSimpleDoingTasksSection()` | 불필요한 리팩토링·방어코드·주석 금지, 보안 취약점 주의, `/help` 안내 |
| Actions | `getActionsSection()` | 위험한 행동(파괴적/force-push/3rd party 업로드 등) 전에 확인하라는 지침 |
| Using Tools | `getUsingYourToolsSection(enabledTools)` | Bash 대신 FileRead/Edit/Write/Glob/Grep 쓰라는 가이드, REPL 모드면 task 도구 위주 |
| Tone & Style | `getSimpleToneAndStyleSection()` | 이모지 금지, `file_path:line_number` 형식, tool call 앞 콜론 금지 |
| Output Efficiency | `getOutputEfficiencySection()` | "Go straight to the point", 결정·상태·에러 위주로 짧게 |

정적 섹션의 핵심 스니펫:

```typescript
// 1) Intro
function getSimpleIntroSection(outputStyleConfig) {
  return `
You are an interactive agent that helps users ${
  outputStyleConfig !== null
    ? 'according to your "Output Style" below...'
    : 'with software engineering tasks.'
} Use the instructions below and the tools available...

${CYBER_RISK_INSTRUCTION}
IMPORTANT: You must NEVER generate or guess URLs...`
}

// 2) System
function getSimpleSystemSection() {
  return `# System
- All text you output outside of tool use is displayed to the user...
- Tools are executed in a user-selected permission mode...
- Tool results and user messages may include <system-reminder> tags...
- Tool results may include data from external sources...
- Users may configure 'hooks'...
- The system will automatically compress prior messages...`
}

// 5) Using Tools — enabledTools 에 따라 다른 문구
function getUsingYourToolsSection(enabledTools: Set<string>): string {
  if (isReplModeEnabled()) { /* task 도구 중심 가이드 */ }
  const providedToolSubitems = [
    `To read files use FileRead instead of cat...`,
    `To edit files use FileEdit instead of sed...`,
    `To create files use FileWrite...`,
    `To search files use Glob instead of find...`,
    `To search content use Grep instead of grep...`,
    `Reserve Bash for system commands only...`,
  ]
  ...
}
```

---

## 동적 섹션 목록 (경계 마커 "이후")

동적 섹션은 세션마다, 심지어 turn 마다 내용이 바뀔 수 있어서 캐시를 붙이지 않습니다. `resolveSystemPromptSections` 가 병렬로 모든 섹션을 평가한 뒤 null 을 걸러냅니다.

```typescript
const dynamicSections = [
  systemPromptSection('session_guidance', () =>
    getSessionSpecificGuidanceSection(enabledTools, skillToolCommands)),
  systemPromptSection('memory',            () => loadMemoryPrompt()),
  systemPromptSection('ant_model_override',() => getAntModelOverrideSection()),
  systemPromptSection('env_info_simple',   () =>
    computeSimpleEnvInfo(model, additionalWorkingDirectories)),
  systemPromptSection('language',          () => getLanguageSection(settings.language)),
  systemPromptSection('output_style',      () => getOutputStyleSection(outputStyleConfig)),
  DANGEROUS_uncachedSystemPromptSection(
    'mcp_instructions',
    () => isMcpInstructionsDeltaEnabled() ? null : getMcpInstructionsSection(mcpClients),
    'MCP servers connect/disconnect between turns',
  ),
  systemPromptSection('scratchpad',        () => getScratchpadInstructions()),
  systemPromptSection('frc',               () => getFunctionResultClearingSection(model)),
  systemPromptSection('summarize_tool_results', () => SUMMARIZE_TOOL_RESULTS_SECTION),
  ...(process.env.USER_TYPE === 'ant'
    ? [systemPromptSection('numeric_length_anchors', () =>
        'Length limits: keep text between tool calls to ≤25 words...')]
    : []),
  ...(feature('TOKEN_BUDGET')
    ? [systemPromptSection('token_budget', () =>
        'When the user specifies a token target...')]
    : []),
]
```

| 섹션 | 내용 |
|------|------|
| `session_guidance` | 활성화된 tool 과 skill 에 따라 동적으로 조합되는 세션 가이드 |
| `memory` | auto-memory 파일(`MEMORY.md` 인덱스 등)에서 로드한 장기 기억 |
| `ant_model_override` | Anthropic 내부 전용 오버라이드 (일반 사용자에겐 null) |
| `env_info_simple` | OS, 플랫폼, 쉘, 현재 date, CWD, 추가 working dirs, 모델 ID, 지식 컷오프 |
| `language` | 사용자 설정 언어 안내 |
| `output_style` | `.claude/output-styles/*.md` 에서 읽은 스타일 지시 |
| `mcp_instructions` | 연결된 MCP 서버의 instructions. 연결 상태가 턴 사이에 바뀔 수 있어 캐시 금지 플래그가 명시됨 |
| `scratchpad` | 사용자 scratchpad 기능 안내 |
| `frc` (function result clearing) | 큰 tool 결과를 자동으로 정리한다는 사실을 모델에 고지 |
| `summarize_tool_results` | 큰 결과를 모델이 스스로 요약하도록 유도 |
| `numeric_length_anchors` | Ant 전용, 응답 길이 제한 |
| `token_budget` | `TOKEN_BUDGET` feature flag 가 켜진 경우만 |

---

## 캐시 경계: `splitSysPromptPrefix`

[src/utils/api.ts](../../src/utils/api.ts) 의 `splitSysPromptPrefix` 가 실제로 `SystemPromptBlock[]` 을 만듭니다. 각 블록은 `text` 와 `cacheScope` 를 가지며, `cacheScope` 가 non-null 이면 그만큼 prompt caching 이 활성화됩니다.

```typescript
// src/utils/api.ts
export function splitSysPromptPrefix(
  systemPrompt: SystemPrompt,
  options?: { skipGlobalCacheForSystemPrompt?: boolean },
): SystemPromptBlock[] {
  const useGlobalCacheFeature = shouldUseGlobalCacheScope()

  if (!useGlobalCacheFeature) {
    // 캐싱 OFF — 모두 no-cache
    return systemPrompt.map((text) => ({ text, cacheScope: null }))
  }

  const boundaryIndex = systemPrompt.findIndex(
    s => s === SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
  )
  if (boundaryIndex === -1) {
    // 경계 마커 없으면 전체 no-cache (보수적)
    return systemPrompt.map((text) => ({ text, cacheScope: null }))
  }

  const result: SystemPromptBlock[] = []

  // (선택) attribution header — cacheScope: null
  if (attributionHeader) {
    result.push({ text: attributionHeader, cacheScope: null })
  }
  // (선택) CLI prefix — cacheScope: null
  if (systemPromptPrefix) {
    result.push({ text: systemPromptPrefix, cacheScope: null })
  }

  // 경계 이전 → 하나로 합쳐 global 캐시 블록
  const staticJoined = staticBlocks.join('\n\n')
  if (staticJoined) {
    result.push({ text: staticJoined, cacheScope: 'global' })
  }

  // 경계 이후 → 하나로 합쳐 no-cache 블록
  const dynamicJoined = dynamicBlocks.join('\n\n')
  if (dynamicJoined) {
    result.push({ text: dynamicJoined, cacheScope: null })
  }

  return result
}
```

### `cacheScope` 의 의미

| 값 | 의미 | 사용처 |
|----|------|--------|
| `'global'` | 여러 요청·여러 사용자 간 공유 가능한 ephemeral cache | 정적 섹션 전체 |
| `'org'` | 동일 조직 내 공유 | 조직 전용 블록 (해당 시) |
| `null` | 캐시 없음, 매 요청 새로 전송 | 경계 이후 동적 섹션, attribution header, CLI prefix |

`cache_control` 은 [src/services/api/claude.ts](../../src/services/api/claude.ts) 의 `buildSystemPromptBlocks` 가 부착합니다.

```typescript
export function buildSystemPromptBlocks(
  systemPrompt: SystemPrompt,
  enablePromptCaching: boolean,
  options?: { skipGlobalCacheForSystemPrompt?: boolean; querySource?: QuerySource },
): TextBlockParam[] {
  return splitSysPromptPrefix(systemPrompt, {
    skipGlobalCacheForSystemPrompt: options?.skipGlobalCacheForSystemPrompt,
  }).map(block => ({
    type: 'text' as const,
    text: block.text,
    ...(enablePromptCaching && block.cacheScope !== null && {
      cache_control: getCacheControl({
        scope: block.cacheScope,
        querySource: options?.querySource,
      }),
    }),
  }))
}
```

---

## 모드별 차이

| 모드 | 특이점 |
|------|--------|
| **REPL 메인 스레드** | 기본 전체 구조. `session_guidance` 가 REPL UI 명령까지 안내 |
| **SubAgent (`AgentTool`)** | `getSystemPrompt` 를 **호출하지 않고**, 에이전트 정의의 본문을 직접 시스템 프롬프트로 사용. 일부 env details 만 append. 자세한 내용은 [../agents/04_execution.md](../agents/04_execution.md) |
| **Plan 모드** | 정적 섹션 일부 + plan 전용 지시 섹션 추가 |
| **Simple 모드 (`CLAUDE_CODE_SIMPLE`)** | 단 한 줄 ("You are OpenClaude..." + CWD + Date) |
| **Proactive / Kairos** | 별도 경로. Intro 대신 자동 에이전트용 지시, 끝에 `getProactiveSection()` 추가 |

```typescript
// src/constants/prompts.ts (Proactive 분기)
if ((feature('PROACTIVE') || feature('KAIROS')) && proactiveModule?.isProactiveActive()) {
  return [
    `\nYou are an autonomous agent. Use the available tools...`,
    getSystemRemindersSection(),
    await loadMemoryPrompt(),
    envInfo,
    getLanguageSection(settings.language),
    getMcpInstructionsSection(mcpClients),
    getScratchpadInstructions(),
    getFunctionResultClearingSection(model),
    SUMMARIZE_TOOL_RESULTS_SECTION,
    getProactiveSection(),
  ].filter(s => s !== null)
}
```

---

## `--dump-system-prompt` 로 실체 확인하기

실제 조립 결과는 CLI 옵션으로 바로 확인할 수 있습니다. 내부적으로는 `fetchOverride` 에 덤프 함수를 끼워넣어, API 호출 직전의 페이로드를 콘솔로 흘려보내는 방식입니다.

```bash
# 현재 설정 기준 system prompt 전체를 stdout 으로
openclaude --dump-system-prompt
```

덤프된 블록은 이 문서의 정적/동적 구분과 일대일로 대응됩니다. 디버깅 시 `splitSysPromptPrefix` 의 결과가 기대한 대로 쪼개졌는지, 캐시 플래그가 어디에 붙었는지 빠르게 확인할 수 있습니다.

---

## 섹션 캐시 (구현 디테일)

동적 섹션은 파일 I/O 나 MCP 상태 조회 등이 포함될 수 있어서, 같은 turn 안에서 여러 번 재계산되지 않도록 `state.systemPromptSectionCache` 에 섹션 식별자를 키로 캐싱합니다.

```typescript
Map<string, string | null>
// key:   섹션 식별자 (예: "env_info_simple", "memory")
// value: 캐싱된 문자열 또는 null (해당 섹션 없음)
```

이 캐시는 요청 단위로 관리되며, auto-compact 이후에도 유효합니다. 섹션이 명시적으로 `DANGEROUS_uncachedSystemPromptSection` 로 등록되면 이 section cache 도 우회합니다 (예: MCP instructions).

---

## 다음 문서

- [06_context_injection.md](06_context_injection.md) — `systemContext` / `userContext` 가 언제, 어떻게 메시지 / 시스템 프롬프트에 주입되는가
