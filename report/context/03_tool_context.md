# 03. Tool 실행 컨텍스트

## 관련 파일

- `src/services/tools/` — Tool 실행 관련 모듈
- `src/utils/toolResultStorage.ts` — 대용량 결과 저장
- `src/utils/attachments.ts` — Tool 메타데이터

## Tool 결과가 컨텍스트에 포함되는 흐름

```
AssistantMessage (tool_use 블록 포함)
    │
    │  { type: 'tool_use', id: 'xyz', name: 'Bash', input: {...} }
    │
    ▼
runTools()  →  실제 Tool 함수 실행
    │
    ▼
UserMessage (tool_result 블록 포함)
    │
    │  { type: 'tool_result', tool_use_id: 'xyz', content: [...] }
    │
    ▼
다음 API 요청의 messages 배열에 포함
```

## tool_result 메시지 구조

```typescript
// UserMessage 안의 tool_result 블록
{
  type: 'tool_result',
  tool_use_id: string,          // 대응하는 tool_use의 ID
  content: ContentBlockParam[], // 결과 (텍스트, 이미지 등)
  is_error?: boolean            // Tool 실행 실패 여부
}
```

하나의 UserMessage가 여러 tool_result를 포함할 수 있습니다
(모델이 여러 tool을 병렬 호출한 경우).

## 대용량 결과 처리

파일: `src/utils/toolResultStorage.ts`

결과가 너무 클 경우 외부 저장소에 보관하고 메시지에는 참조(스텁)만 저장:

```typescript
recordContentReplacement(blockId, originalContent)
// 원본 내용은 별도 저장
// 메시지에는 "[content stored externally: <id>]" 형태의 스텁
// 세션 JSONL 파일에 ContentReplacementEntry로 기록
```

로드 시 스텁을 실제 내용으로 복원합니다.

## Tool 권한 확인

Tool 실행 전 반드시 권한 확인을 거칩니다:

```typescript
canUseTool(toolName, toolInput, context): Promise<boolean>
// 거부 규칙(deny rules) 확인
// 사용자 승인 필요 시 PermissionRequest UI 표시
// 승인/거부 결과에 따라 Tool 실행 또는 거부
```

Tool이 거부된 경우에도 tool_result를 생성합니다 (is_error: true).

## Tool 실행 중 컨텍스트 메타데이터

파일: `src/utils/attachments.ts`

Tool 실행 중 발생한 부가 정보를 AttachmentMessage로 관리:

```typescript
// Hook 실행 결과
HookAttachment: {
  type: 'hook'
  hookType: string    // pre-tool, post-tool 등
  output: string
}
```

## StreamingToolExecutor

Tool 실행 결과를 실시간 스트리밍으로 반환:

- 실행 중 ProgressMessage를 REPL에 표시
- 완료 후 tool_result를 메시지 컨텍스트에 추가
- 여러 Tool의 병렬 실행 지원 (toolOrchestration.ts)
