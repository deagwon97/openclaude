# OpenClaude 메모리 관리 시스템 분석

## 개요

OpenClaude의 메모리 시스템은 **3계층 구조**로 설계되어 있다.

| 계층 | 범위 | 저장 형식 | 파일 |
|------|------|-----------|------|
| 히스토리 | 전역 (세션 간 공유) | JSONL | `~/.claude/history.jsonl` |
| 세션 메모리 | 단일 대화 | JSONL 트랜스크립트 | `~/.claude/projects/.../session.jsonl` |
| 자동 메모리 | 프로젝트 장기 지식 | 마크다운 파일 | `~/.claude/projects/.../memory/` |

## 목차

| 파일 | 내용 |
|------|------|
| [01_conversation_history.md](01_conversation_history.md) | 대화 히스토리 저장/조회 구조 |
| [02_context_compression.md](02_context_compression.md) | 컨텍스트 압축(compact) 메커니즘 |
| [03_message_queue.md](03_message_queue.md) | 메시지 큐 및 우선순위 관리 |
| [04_token_management.md](04_token_management.md) | 토큰 측정 및 컨텍스트 윈도우 관리 |
| [05_session_vs_persistent.md](05_session_vs_persistent.md) | 세션 메모리 vs 영구(자동) 메모리 |

## 핵심 파일 지도

```
src/
├── history.ts                        # 전역 히스토리 (Ctrl+R, ↑ 화살표)
├── tools.ts                          # tool 목록 관리
├── query.ts                          # Claude API 호출 + tool 루프
├── QueryEngine.ts                    # 멀티턴 tool 시퀀스
├── memdir/
│   ├── paths.ts                      # 메모리 경로 및 활성화 여부
│   ├── memoryScan.ts                 # 메모리 파일 스캔
│   ├── memoryTypes.ts                # 메모리 타입 정의
│   └── memdir.ts                     # MEMORY.md 진입점 관리
├── services/
│   └── compact/
│       ├── compact.ts                # 압축 엔진 (이미지 제거 등)
│       └── prompt.ts                 # 압축 프롬프트 템플릿
└── utils/
    ├── messages.ts                   # 압축 경계 마커 생성/탐색
    ├── messageQueueManager.ts        # 메시지 큐 (우선순위 기반)
    ├── sessionStorage.ts             # 세션 트랜스크립트 저장
    ├── conversationRecovery.ts       # 세션 복구
    ├── tokens.ts                     # 토큰 사용량 측정
    └── context.ts                    # 컨텍스트 윈도우 크기
```
