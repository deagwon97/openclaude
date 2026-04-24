# 사용자 상호작용 도구 (User Interaction Tools)

사용자에게 질문하거나 메시지를 전달하는 도구들입니다.

---

## AskUserQuestion (AskUserQuestionTool)

**소스**: [src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx](../../src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx)

### 설명
사용자에게 구조화된 선택지를 제시하고 답변을 받는 도구. 단순 텍스트 질문이 아닌 선택지(options)가 있는 인터랙티브 UI를 표시한다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `questions` | Question[] | 필수 | 질문 목록 |

### Question 구조

```typescript
{
  question: string,     // 질문 텍스트 (물음표로 끝나야 함)
  header: string,       // 짧은 라벨/칩 (최대 25자)
  options: Option[],    // 선택지 (2~4개)
  multiSelect: boolean  // 복수 선택 허용 여부 (기본값: false)
}
```

### Option 구조

```typescript
{
  label: string,        // 표시 텍스트 (1~5 단어)
  description: string,  // 선택지 설명 (트레이드오프 등)
  preview?: string      // 선택 시 미리보기 콘텐츠 (선택사항)
}
```

### 특징
- **제약사항**: 선택지는 최소 2개, 최대 4개
- **자동 "기타" 옵션**: 4개 선택지 외에 자동으로 "Other" 옵션 제공
- **다중 선택 모드**: `multiSelect: true` 시 여러 선택지 동시 선택 가능
- **미리보기 지원**: 코드 스니펫, 목업 등 시각적 비교 가능
- **채널 지원**: KAIROS_CHANNELS 기능 활성화 시 채널로 질문 전달

### 사용 시점
- 구현 방향을 결정하기 전 사용자 의견이 필요할 때
- 여러 트레이드오프 있는 선택지 중 사용자가 선택해야 할 때
- 계획 모드에서 접근 방식 명확화가 필요할 때

---

## Brief (BriefTool)

**소스**: [src/tools/BriefTool/BriefTool.ts](../../src/tools/BriefTool/BriefTool.ts)

### 설명
사용자에게 메시지와 선택적 첨부파일을 전달하는 도구. 주로 백그라운드 작업 완료 알림, 진행 상황 업데이트, 또는 일반 대화 응답에 사용된다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `message` | string | 필수 | 사용자에게 보낼 메시지 (마크다운 지원) |
| `attachments` | string[] | 선택 | 첨부 파일 경로 목록 (절대경로 또는 cwd 기준 상대경로) |
| `status` | enum | 필수 | `"normal"` 또는 `"proactive"` |

### status 구분

| 값 | 사용 상황 |
|----|----------|
| `"normal"` | 사용자가 방금 말한 것에 대한 응답 |
| `"proactive"` | 사용자가 요청하지 않은 것을 자발적으로 알릴 때 (작업 완료, 블로커 발생, 상태 업데이트 등) |

### 첨부파일 지원 타입
- 이미지 (사진, 스크린샷)
- diff/로그 파일
- 기타 사용자가 봐야 할 파일

### 레거시 이름
- `BriefTool` (구버전 호환용 `LEGACY_BRIEF_TOOL_NAME`)

### 활성화 조건
- Kairos 기능 활성화 또는 `getUserMsgOptIn()` 반환값이 true일 때 완전 기능 활성화
- 기본 상태에서도 동작하지만 일부 기능 제한 가능
