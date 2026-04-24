# 웹 도구 (Web Tools)

인터넷에서 정보를 가져오는 도구들입니다.

---

## WebFetch (WebFetchTool)

**소스**: [src/tools/WebFetchTool/WebFetchTool.ts](../../src/tools/WebFetchTool/WebFetchTool.ts)

### 설명
특정 URL에서 콘텐츠를 가져와 마크다운으로 변환한 뒤, 지정한 프롬프트에 따라 처리하는 도구.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `url` | string | 필수 | 콘텐츠를 가져올 URL |
| `prompt` | string | 필수 | 가져온 콘텐츠에 적용할 처리 지시사항 |

### 동작 흐름
1. URL 유효성 검사
2. 권한 확인 (preapproved 호스트, allow/deny/ask 규칙)
3. Firecrawl API 사용 가능 시: Firecrawl로 마크다운 스크래핑
4. 일반 경우: 직접 URL 페치 후 마크다운 변환
5. 프롬프트를 적용해 결과 요약 (내부적으로 Haiku 모델 사용)
6. 바이너리 콘텐츠(PDF 등)는 디스크에 저장 후 경로 알림

### 특징
- **Firecrawl 통합**: `FIRECRAWL_API_KEY` 환경변수 설정 시 Firecrawl 사용
- **리디렉션 처리**: 다른 호스트로의 리디렉션 감지 시 사용자에게 안내
- **바이너리 처리**: PDF 등 바이너리는 임시 파일로 저장
- **Preapproved 호스트**: 특정 신뢰할 수 있는 도메인은 권한 없이 접근 가능
- **인증 URL 불가**: Google Docs, Confluence 등 인증 필요 URL은 실패함

### 권한
- 호스트명 기반 allow/deny/ask 규칙 (`domain:hostname` 형식)
- 첫 접근 시 기본적으로 사용자 승인 요청

### 반환값
```typescript
{
  bytes: number,        // 콘텐츠 크기 (바이트)
  code: number,         // HTTP 응답 코드
  codeText: string,     // HTTP 응답 텍스트
  result: string,       // 처리된 결과 (프롬프트 적용 후)
  durationMs: number,   // 처리 시간
  url: string           // 요청한 URL
}
```

---

## WebSearch (WebSearchTool)

**소스**: [src/tools/WebSearchTool/WebSearchTool.ts](../../src/tools/WebSearchTool/WebSearchTool.ts)

### 설명
웹 검색을 수행하고 결과를 반환하는 도구. 다양한 제공자(Anthropic native, Vertex, Codex, 외부 어댑터)를 지원한다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `query` | string | 필수 | 검색 쿼리 (최소 2자) |
| `allowed_domains` | string[] | 선택 | 허용할 도메인 목록 |
| `blocked_domains` | string[] | 선택 | 차단할 도메인 목록 (allowed_domains와 동시 사용 불가) |

### 검색 제공자 결정 로직

```
WEB_SEARCH_PROVIDER 환경변수에 따라:
  - "native": Anthropic native 경로만 사용
  - "auto" (기본):
    1. 외부 어댑터 (Tavily, DuckDuckGo, custom) 시도
    2. 실패 시 Codex/native로 폴백
  - 특정 모드 (tavily, ddg, custom): 해당 어댑터 사용
```

### 활성화 조건 (isEnabled)
다음 중 하나라도 해당되면 활성화:
- 외부 어댑터 사용 가능 (Tavily 등 설정됨)
- Codex Responses API 사용 가능 (OpenAI 프로바이더)
- Anthropic firstParty 프로바이더
- Vertex AI + Claude 4.x 모델
- Foundry 프로바이더

### 네이티브 경로 (Anthropic)
- `web_search_20250305` 도구를 Claude 모델에 전달
- 최대 8회 검색 수행 (`max_uses: 8`)
- 스트리밍으로 진행 상황 실시간 전달

### 반환값
```typescript
{
  query: string,
  results: Array<SearchResult | string>,  // 검색 결과 + 텍스트 요약
  durationSeconds: number
}
```

### 지원 외부 어댑터 (`WEB_SEARCH_PROVIDER`)
| 값 | 설명 |
|----|------|
| `tavily` | Tavily Search API |
| `ddg` | DuckDuckGo |
| `custom` | `WEB_URL_TEMPLATE` 환경변수로 지정한 커스텀 URL |
| `auto` | 사용 가능한 어댑터 자동 선택 후 native 폴백 |
