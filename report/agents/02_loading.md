# 에이전트 로딩 흐름 (Loading)

커스텀 에이전트가 디스크에서 읽혀 런타임의 `AgentDefinition` 객체로 변환되는 과정입니다.

핵심 파일:
- [src/tools/AgentTool/loadAgentsDir.ts](../../src/tools/AgentTool/loadAgentsDir.ts)
- [src/utils/markdownConfigLoader.ts](../../src/utils/markdownConfigLoader.ts)

---

## 1. 진입점

모든 에이전트 목록은 다음 한 함수로 얻습니다.

```
getAgentDefinitionsWithOverrides(cwd)
  → loadMarkdownFilesForSubdir('agents', cwd)   // 디스크 스캔
  → parseAgentFromMarkdown() / parseAgentFromJson()  // 파싱
  → getActiveAgentsFromList()                    // 우선순위 병합
```

이 함수는 `lodash.memoize(by cwd)`로 캐시되며, 결과는 런타임 전체에서 재사용됩니다. 캐시 무효화는 `clearAgentDefinitionsCache()`로 수행합니다 (플러그인 에이전트 캐시도 함께 비움).

---

## 2. 디렉토리 스캔

`loadMarkdownFilesForSubdir('agents', cwd)`는 세 그룹의 경로에서 `.md` 파일을 수집합니다. 구현은 [src/utils/markdownConfigLoader.ts](../../src/utils/markdownConfigLoader.ts) 참고.

| 그룹 | 탐색 경로 |
|------|----------|
| managed | `getManagedFilePath()/.claude/agents/` — 관리자/조직 정책 |
| user | `~/.claude/agents/` |
| project | `cwd`에서 git root까지 상향 순회하며 만나는 모든 `.claude/agents/` |

프로젝트 범위는 모노레포를 고려해 중간 디렉토리까지 포함하도록 설계되어 있습니다.

---

## 3. 파싱

파일별로 다음 파서가 선택됩니다.

- `.md` → [parseAgentFromMarkdown()](../../src/tools/AgentTool/loadAgentsDir.ts)
- `.json` → [parseAgentFromJson()](../../src/tools/AgentTool/loadAgentsDir.ts)

공통 단계:

1. **Frontmatter 추출** — `gray-matter` 계열 파서로 YAML 블록을 읽음
2. **Zod 스키마 검증** — `AgentJsonSchema`로 타입·범위 검증 (잘못된 필드는 해당 에이전트 무효화)
3. **필드 정규화**:
   - `name`: 파일명에서 기본값 유도, frontmatter가 우선
   - `tools`: [parseAgentToolsFromFrontmatter()](../../src/utils/markdownConfigLoader.ts) — `'*'`, `[]`, 구체적 목록 구분
   - `memory`: 지정 시 File Read/Write/Edit 자동 주입
   - `maxTurns`: 양의 정수만 허용, 그 외는 무시
   - `background`, `isolation`: 실행 모드 플래그로 저장
4. **CustomAgentDefinition 생성** — 런타임에서 사용할 표준 구조로 변환

파싱 실패 시 해당 파일은 스킵되며, 경고 로그는 남되 다른 에이전트 로딩을 막지 않습니다.

---

## 4. 우선순위 병합

파싱된 에이전트는 [getActiveAgentsFromList()](../../src/tools/AgentTool/loadAgentsDir.ts)에서 병합됩니다. 병합 순서(낮음 → 높음)는 다음과 같습니다.

```
builtInAgents   →  pluginAgents  →  userAgents  →  projectAgents
              →  flagAgents     →  managedAgents
```

같은 `agentType`(이름)이 여러 그룹에 있으면 **뒤 그룹이 이전 그룹을 완전히 덮어씁니다** (필드 단위 병합이 아님). 따라서:

- 내장 `general-purpose`를 프로젝트에서 재정의하면 프로젝트 버전이 사용됨
- 관리자 정책(managed)이 존재하면 사용자/프로젝트 설정을 강제 오버라이드

---

## 5. 캐싱

| 항목 | 키 | 무효화 |
|------|----|--------|
| `getAgentDefinitionsWithOverrides` | `cwd` | `clearAgentDefinitionsCache()` |
| 플러그인 에이전트 | 내부 관리 | 상위 캐시 클리어와 연동 |
| 색상 매핑 | 에이전트 이름 | 프로세스 수명과 동일 |

런타임에서 파일을 수정해도 캐시가 살아 있으면 변경이 반영되지 않습니다. 일반적으로 프로세스 재시작(또는 명시적 캐시 클리어) 시점에 새 정의가 읽힙니다.

---

## 6. 로딩 흐름 다이어그램

```
┌──────────────────────────────────────┐
│  .md / .json in .claude/agents       │
└──────────────────┬───────────────────┘
                   │
    loadMarkdownFilesForSubdir('agents')
                   │
     ┌─────────────┼─────────────┐
     ▼             ▼             ▼
  managed        user         project
                   │
                   ▼
        parseAgentFromMarkdown / Json
                   │
                   ▼
           Zod 검증 + 정규화
                   │
                   ▼
        CustomAgentDefinition[]
                   │
                   ▼
       getActiveAgentsFromList()
    (builtIn → plugin → user → project
              → flag → managed 순 덮어쓰기)
                   │
                   ▼
         memoize(by cwd) 캐시
                   │
                   ▼
        activeAgents: AgentDefinition[]
                   │
                   ▼
        AgentTool.prompt() / call()
           (→ 03_registration.md)
```
