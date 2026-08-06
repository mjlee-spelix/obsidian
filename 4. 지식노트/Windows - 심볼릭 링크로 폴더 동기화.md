---
tags: [개발, CS, Windows, PowerShell]
date: 2026-04-29
---
# Windows - 심볼릭 링크로 폴더 동기화

## 핵심
- **심볼릭 링크(Symbolic Link)** = OS 수준의 "강화된 바로가기". 프로그램이 진짜 폴더/파일로 인식
- 일반 바로가기(`.lnk`)와 달리 모든 프로그램이 원본 폴더처럼 다룸 → 한쪽 수정 = 양쪽 즉시 반영
- 옵시디언(노트 마스터) ↔ VSCode 프로젝트(`.claude/`) 같은 **양방향 동기화**에 적합

## 일반 바로가기와의 차이

| | 일반 바로가기(.lnk) | 심볼릭 링크 |
|--|------------------|------------|
| 인식 주체 | Windows 탐색기만 | **모든 프로그램** (OS 수준) |
| VSCode/CLI에서 | 그냥 파일로 보임 | 진짜 폴더처럼 동작 |
| 더블클릭 시 | 원본 위치로 이동 | 자기 위치 그대로, 내용은 원본 |
| 만들기 | 우클릭 → 바로가기 만들기 | `New-Item -ItemType SymbolicLink` |

## 만드는 방법 (PowerShell)

### 기본 명령

```powershell
New-Item -ItemType SymbolicLink `
  -Path   "링크가 만들어질 위치" `
  -Target "링크가 가리킬 실제 폴더"
```

| 파라미터 | 의미 |
|---------|------|
| `-ItemType SymbolicLink` | 항목 종류 = 심볼릭 링크 |
| `-Path` | **만들 링크의 위치** (이 경로에 폴더처럼 보임) |
| `-Target` | **링크가 가리킬 실제 폴더** (원본 마스터) |
| `` ` `` | PowerShell 줄바꿈 (가독성용) |

### 실제 사례: 옵시디언 ↔ VSCode .claude 연동

```powershell
New-Item -ItemType SymbolicLink `
  -Path   "C:\Users\Administrator\Projects\spx-agent\.claude" `
  -Target "C:\Users\Administrator\Documents\Obsidian\Daily\3. 프로젝트\spx-agent"
```

→ VSCode `.claude/` 안에서 옵시디언 파일이 그대로 보임. 한쪽 수정 = 양쪽 반영.

## 전제 조건

- **관리자 PowerShell** 또는 **개발자 모드 활성화** 필요
  - 시작 메뉴 → "PowerShell" 우클릭 → "관리자 권한으로 실행"
  - 또는: 설정 → 개발자용 → 개발자 모드 켜기 (일반 권한으로도 가능)

## 흔한 에러

### `ResourceExists` (NewItemIOError)

```
New-Item : NewItemIOError
ResourceExists: ...
```

**원인**: `-Path`에 지정한 위치에 **이미 폴더/파일이 존재**해서 충돌.

**해결**:
```powershell
# 기존 폴더 백업 (안전 차원)
Rename-Item "기존경로" "기존경로.bak"

# 그 다음 심볼릭 링크 생성

# 동작 확인 후 백업 삭제
Remove-Item "기존경로.bak" -Recurse
```

기존 폴더에 보존할 파일 있으면(예: `settings.local.json`) **마스터 쪽으로 먼저 복사**한 뒤 진행.

## 동작 확인

### 만들어진 링크인지 검증

```powershell
ls "심볼릭 링크 경로 부모"
```

**Mode 컬럼**을 봄:
- 일반 디렉토리: `d----`
- 심볼릭 링크 디렉토리: `d----l` ← 마지막 `l`이 **link** 의미

### 어디를 가리키는지 확인

```powershell
(Get-Item "심볼릭 링크 경로").Target
```

→ 원본 절대 경로 출력

### 양방향 동기화 테스트

```powershell
# 한쪽에서 파일 생성
echo "test" > "링크쪽/test.txt"

# 다른 쪽에서 보이는지 확인
ls "원본쪽/test.txt"
```

## 주의사항

| 주의 | 설명 |
|------|------|
| **삭제 시 위험** | 링크 자체를 삭제하면 링크만 사라지지만, **링크 안의 파일을 삭제하면 원본도 삭제됨** |
| 관리자 권한 | 일반 권한으로 실행 시 권한 에러. 개발자 모드 활성화로 우회 가능 |
| 이미 폴더 있으면 실패 | `-Path` 위치가 비어 있어야 함. 백업 후 진행 |
| Git 등에서 인식 | 대부분 도구가 정상 인식하지만, 백업 도구는 링크 자체만 백업하기도 함 |

## 사용 사례

- **옵시디언 노트 ↔ 코드 프로젝트** 연동 (이번 사례)
- 여러 프로젝트가 **공통 설정 파일** 공유
- 큰 데이터(영상·게임 등)를 **다른 드라이브**에 두고 원래 위치처럼 접근
- WSL ↔ Windows 폴더 매핑

## 비교: 다른 링크 방식

| 방식 | Windows | 설명 |
|------|---------|------|
| 심볼릭 링크 | `mklink /D`, `New-Item -SymbolicLink` | **다른 드라이브 OK**, 가장 일반적 |
| 정션(Junction) | `mklink /J`, `New-Item -ItemType Junction` | 같은 드라이브 폴더만, **관리자 권한 불필요** |
| 하드 링크 | `mklink /H` | 파일만 가능 (폴더 X), 같은 드라이브만 |

**추천**: 폴더는 심볼릭 링크가 가장 유연하고 호환성 좋음. **하지만 같은 드라이브 안에서만 연결한다면 Junction이 더 편함** (관리자 권한 없이 가능).

## Junction 상세

### 만드는 방법

```powershell
New-Item -ItemType Junction `
  -Path   "C:\Users\Administrator\Documents\Obsidian\Daily\3. 프로젝트\spx-agent\images" `
  -Target "C:\Users\Administrator\Documents\Obsidian\Daily\이미지\spx-agent"
```

→ **일반 권한 PowerShell**에서 실행 가능. 결과 동작은 SymbolicLink와 동일하게 OS 수준에서 진짜 폴더로 인식.

### Junction vs SymbolicLink 비교표

| 항목 | SymbolicLink | Junction |
|------|------------|---------|
| 관리자 권한 | **필요** (또는 개발자 모드) | **불필요** ✅ |
| 다른 드라이브 | OK (예: C → D) | **불가** (같은 볼륨만) |
| 폴더 대상 | OK | OK |
| 파일 대상 | OK | 불가 (폴더만) |
| 네트워크 경로 | OK | 제한적 |
| 동작 (앱 인식) | OS 수준 동일 | OS 수준 동일 |
| `ls` Mode 표시 | `d----l` | `d----l` (동일) |

### 언제 Junction을 쓸까

- ✅ 같은 드라이브 안에서 폴더 연결할 때 (대부분의 케이스)
- ✅ 관리자 권한 없이 빠르게 만들 때
- ✅ 일회성 동기화 / 개발자 도구 워크플로

### 언제 SymbolicLink를 쓸까

- 다른 드라이브 간 연결 (C ↔ D)
- 파일 단위 링크 필요
- 네트워크 경로 연결

### 실제 사례 (이번 프로젝트)

| 링크 | 종류 | 이유 |
|------|------|------|
| `spx-agent\.claude` → `옵시디언/3. 프로젝트/spx-agent` | SymbolicLink (관리자) | 다른 드라이브 가능성 + 처음 만들 때 관리자 권한 있었음 |
| `옵시디언/3. 프로젝트/spx-agent/images` → `옵시디언/이미지/spx-agent` | **Junction** | 같은 드라이브, 관리자 권한 없는 상태에서 즉시 필요 |

### 주의

- **관리자 권한 PowerShell이 안 열려 있는 상태**에서 폴더 동기화가 즉시 필요하면 Junction이 정답
- 단, **다른 드라이브로 이동할 가능성이 있는 폴더**는 SymbolicLink가 더 안전
- Junction도 `(Get-Item "경로").Target`으로 가리키는 곳 확인 가능

## 관련 노트
- [[3. 프로젝트/SPX-Agent 하네스 설계.md]]
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]]
