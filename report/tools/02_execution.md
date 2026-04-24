# 실행 도구 (Execution Tools)

쉘 명령을 실행하고 그 결과를 반환하는 도구들입니다.

---

## Bash (BashTool)

**소스**: [src/tools/BashTool/BashTool.tsx](../../src/tools/BashTool/BashTool.tsx)

### 설명
쉘 명령을 실행하는 가장 핵심적인 도구. 포그라운드 실행과 백그라운드 실행을 모두 지원하며, 보안 검사, 타임아웃, 샌드박스 등 다양한 안전 장치를 갖추고 있다.

### 입력 파라미터

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| `command` | string | 필수 | 실행할 bash 명령어 |
| `timeout` | number | 선택 | 타임아웃(ms), 최대 600000ms (10분) |
| `description` | string | 선택 | 명령 설명 (사용자에게 표시) |
| `run_in_background` | boolean | 선택 | 백그라운드 실행 여부 (기본값: false) |

### 동작 흐름
1. **보안 검사** (`bashSecurity.ts`): 위험한 명령 패턴 탐지
2. **권한 확인** (`bashPermissions.ts`): allow/deny/ask 규칙 적용
3. **샌드박스 여부 결정** (`shouldUseSandbox.ts`): 환경에 따라 sandbox 실행
4. **명령 실행** (`exec()`): 포그라운드 또는 백그라운드 태스크로 실행
5. **결과 수집**: stdout, stderr, 종료 코드 수집
6. **출력 처리**: 이미지 출력 감지, 긴 출력 truncation 처리

### 특징
- **자동 백그라운드**: 2초 이상 실행되면 `BackgroundHint` 표시
- **cd 명령 처리**: `cd` 명령 포함 시 CWD 업데이트
- **이미지 출력**: 이미지 출력 감지 시 base64로 변환해 멀티모달 표시
- **sed 편집 파싱**: `sed -i` 명령을 FileEditTool 호출로 변환 가능
- **git 작업 추적**: git 관련 명령 실행 시 변경 사항 추적

### 권한 검사 우선순위
1. deny 규칙 → 거부
2. allow 규칙 → 허용
3. ask 규칙 → 사용자 승인 요청
4. 기본값: ask (사용자 승인 요청)

### 보안 검사 항목
- 위험한 명령 패턴 (rm -rf /, fork bomb 등)
- 읽기 전용 모드에서 쓰기 명령 차단
- UNC 경로 차단 (Windows NTLM 자격 증명 유출 방지)

### 상수

```
기본 타임아웃: 120000ms (2분)
최대 타임아웃: 600000ms (10분)
백그라운드 힌트 임계값: 2000ms
Assistant 모드 blocking budget: 15000ms
```

---

## PowerShell (PowerShellTool)

**소스**: [src/tools/PowerShellTool/](../../src/tools/PowerShellTool/)

### 설명
Windows 환경에서 PowerShell 명령을 실행하는 도구. BashTool의 Windows 대응 버전으로, 동일한 안전 장치를 갖추고 있다.

### 활성화 조건
- `isPowerShellToolEnabled()` 반환값이 `true`인 경우 (Windows 환경에서 PowerShell 사용 가능 시)

### 입력 파라미터
BashTool과 동일한 구조 (`command`, `timeout`, `description`, `run_in_background`)

### 동작
BashTool과 유사하나 PowerShell 전용 보안 검사 적용:
- `powershellSecurity.ts`: PowerShell 전용 위험 명령 패턴
- `powershellPermissions.ts`: PowerShell 전용 권한 규칙
- `gitSafety.ts`: git 명령 안전성 검사
