# MCP 도구 (MCP Tools)

Model Context Protocol(MCP) 서버와 연동하는 도구들입니다.

---

## ListMcpResources (ListMcpResourcesTool)

**소스**: [src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts](../../src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts)

### 설명
연결된 MCP 서버에서 사용 가능한 리소스 목록을 조회하는 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `server` | string | 선택 | 특정 서버만 필터링 |

### 반환값
```typescript
Array<{
  uri: string,          // 리소스 URI
  name: string,         // 리소스 이름
  mimeType?: string,    // MIME 타입
  description?: string, // 설명
  server: string        // 제공 서버
}>
```

### 특징
- `shouldDefer: true` - 지연 로드 가능 (ToolSearch를 통해 스키마 조회 후 사용)
- 동시 실행 안전 (`isConcurrencySafe: true`)
- 읽기 전용 (`isReadOnly: true`)

---

## ReadMcpResource (ReadMcpResourceTool)

**소스**: [src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts](../../src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts)

### 설명
MCP 서버에서 특정 리소스의 내용을 읽어오는 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `uri` | string | 필수 | 읽을 리소스의 URI |
| `server` | string | 선택 | 리소스를 제공하는 서버 이름 |

### 특징
- `shouldDefer: true` - 지연 로드 가능
- 동시 실행 안전

---

## ToolSearch (ToolSearchTool)

**소스**: [src/tools/ToolSearchTool/ToolSearchTool.ts](../../src/tools/ToolSearchTool/ToolSearchTool.ts)

### 설명
지연 로드된(deferred) 도구들의 스키마를 검색하고 로드하는 도구. tool 수가 많을 때 Claude API의 컨텍스트 토큰을 절약하기 위해 일부 tool을 지연 로드하며, 이 도구로 필요한 tool의 스키마를 가져온다.

### 활성화 조건
- `isToolSearchEnabledOptimistic()` 반환값이 `true`인 경우
  (일반적으로 전체 tool 수가 임계값을 초과할 때)

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `query` | string | 필수 | 검색 쿼리. `"select:ToolName"` 형식으로 직접 선택 가능 |
| `max_results` | number | 선택 | 최대 결과 수 (기본값: 5) |

### 쿼리 형식
```
"select:Read,Edit,Grep"          → 이름으로 직접 선택
"notebook jupyter"               → 키워드 검색
"+slack send"                    → 이름에 "slack" 필수, 나머지로 순위 지정
```

### 동작
1. 현재 지연 로드된 tool 목록에서 쿼리와 일치하는 tool 탐색
2. 일치하는 tool의 전체 JSONSchema 반환
3. 반환된 스키마를 사용해 해당 tool 호출 가능

### 반환값
일치하는 tool들의 완전한 JSONSchema 정의 (`<functions>` 블록 형식)

---

## MCP Tool 통합 흐름

```
MCP 서버 연결
    ↓
assembleToolPool() 호출
    ├── getTools() → 빌트인 tool 목록
    └── filterToolsByDenyRules(mcpTools) → MCP tool 필터링
    ↓
uniqBy(빌트인 + MCP, 'name') → 이름 중복 시 빌트인 우선
    ↓
tool pool 완성 (프롬프트 캐시 안정성을 위해 이름순 정렬)
```

### 주의사항
- 빌트인 tool과 MCP tool이 같은 이름인 경우 빌트인이 우선 적용
- MCP tool도 `filterToolsByDenyRules()`로 deny 규칙 적용 가능 (`mcp__server` 프리픽스 규칙)
- `shouldDefer: true`인 tool은 ToolSearch로 스키마를 먼저 조회해야 사용 가능
