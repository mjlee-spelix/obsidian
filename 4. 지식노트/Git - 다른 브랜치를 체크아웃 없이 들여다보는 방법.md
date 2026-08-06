---
tags: [개발, Git, 협업, CS]
date: 2026-05-06
---
# Git - 다른 브랜치를 체크아웃 없이 들여다보는 방법

## 핵심
- 협업 중 다른 분 브랜치 참고할 때 **체크아웃하면 본인 작업 흐름이 끊김**
- `git show` / `git diff` / `git log`로 **체크아웃 없이 안전하게** 들여다볼 수 있음
- 5분 분석으로 **환경 표준 / 컨벤션 / 사후 위반 상태**가 다 드러나는 정보 창고

## 사전 — 리모트 최신 받기

```bash
git fetch origin
```

브랜치 이력만 받음 (작업 디렉토리 영향 없음). `git pull`과 다름:
- `pull` = `fetch` + `merge` (작업 디렉토리 변경)
- **`fetch` = 이력만 받기 (안전)**

## 1. 이력 보기 — `git log`

```bash
# 그분 브랜치의 commit 이력
git log origin/feat/rbac --oneline -10

# 작성자 + 날짜 포함
git log origin/feat/rbac --pretty=format:"%h %an %ad %s" --date=short -10

# 특정 파일의 변경 이력만
git log origin/feat/rbac --oneline -- api/controllers/console/auth/keycloak.py
```

## 2. 파일 내용 보기 — `git show <revision>:<path>`

체크아웃 없이 **그 브랜치의 특정 파일 내용**을 볼 수 있음:

```bash
# 그분 브랜치의 keycloak.py 내용
git show origin/feat/rbac:api/controllers/console/auth/keycloak.py

# .nvmrc / package.json 같은 환경 파일
git show origin/feat/rbac:.nvmrc

# PowerShell에서 특정 패턴만
git show origin/feat/rbac:web/package.json | Select-String "engines|packageManager"
```

### 과거 commit의 파일도 가능

```bash
# 특정 commit hash의 파일 내용
git show f3e9dbe:api/controllers/console/auth/keycloak.py

# 3일 전 master의 파일
git show master@{3.days.ago}:src/config.py
```

## 3. 변경 비교 — `git diff`

### 두 브랜치 비교

```bash
# 변경 파일 목록만 (가장 자주 씀)
git diff master..origin/feat/rbac --stat

# 전체 diff
git diff master..origin/feat/rbac

# 특정 파일만
git diff master..origin/feat/rbac -- src/auth.py
```

### `..` vs `...` 차이

| 표기 | 의미 |
|------|------|
| `A..B` | A에 없고 B에 있는 모든 commit |
| `A...B` | **A와 B의 공통 조상부터의 차이** (보통 더 의미 있음) |

```bash
git diff master...origin/feat/rbac --stat   # 공통 조상 이후 그분 작업분만
```

### 본인 현재 브랜치 vs 그분 브랜치

```bash
# HEAD = 현재 브랜치 최신 commit
git diff HEAD..origin/feat/rbac --stat
git diff HEAD...origin/feat/rbac --stat
```

## 4. 폴더 구조만 빠르게 — `git ls-tree`

```bash
# 그분 브랜치의 특정 폴더 내용
git ls-tree origin/feat/rbac api/services/admin/

# 재귀
git ls-tree -r origin/feat/rbac api/services/admin/
```

## 5. 한 줄 워크플로우 (실전 가장 효율)

```bash
# 한 화면에 (a) 최신 받기 (b) 그분 commit 흐름 (c) 변경 파일 목록
git fetch origin && \
git log origin/feat/rbac --oneline -10 && \
git diff master..origin/feat/rbac --stat
```

PowerShell:
```powershell
git fetch origin; git log origin/feat/rbac --oneline -10; git diff master..origin/feat/rbac --stat
```

## 체크아웃해서 봐야 할 때 — 안전 패턴

여러 파일 한꺼번에 탐색하거나 IDE 도구로 보고 싶을 때:

```bash
# 본인 작업 깨끗한지 확인
git status

# 그분 브랜치로 이동
git switch feat/rbac
# 또는 detached HEAD (실수로 commit 안 하게)
git switch --detach origin/feat/rbac

# 둘러본 후 원래 브랜치 복귀
git switch -    # 직전 브랜치로 한 번에
```

## VSCode + GitLens 확장

```
GitLens 확장 설치 시:
- 좌측 사이드바 → GitLens → "Branches"
- 다른 브랜치 우클릭 → "Open Branch on Remote"
- 또는 "Compare with HEAD"
```

명령줄보다 시각적이고 편함. **diff 색상 + 파일별 상세** 한 화면에.

## 실전 — 5분에 4가지 단서 확보 (2026-05-06)

본인이 동료 브랜치(`feat/rbac`) 한 번 들여다보고 발견한 것:

### 1. lint 위반 사후 상태 — 협업 매너 정리

```powershell
git show origin/feat/rbac:api/controllers/console/auth/keycloak.py | Select-String "logger.error|logger.exception"
```

→ except 블록 안 `logger.error` 다수 그대로. **그분도 모르는 상태 확인** → 슬랙 알림 결정.

### 2. Node 표준 — 회사 환경 표준 발견

```powershell
git show origin/feat/rbac:.nvmrc
# 22
```

→ 회사 표준 = Node 22. 본인 호스트 20에 머물러있던 drift 확정. **Node 22 업그레이드 결정 근거**.

### 3. 패키지 매니저 표준

```powershell
git show origin/feat/rbac:package.json | Select-String "packageManager"
# "packageManager": "pnpm@10.33.0"
```

→ corepack으로 자동 매칭 가능. **글로벌 pnpm 설치 안 해도 됨**.

### 4. Hook 패턴 검증

```powershell
git show origin/feat/rbac:.pre-commit-config.yaml
# fatal: path '.pre-commit-config.yaml' does not exist
```

→ pre-commit framework 미사용 확정. `.vite-hooks/` 가 husky rename 패턴임이 굳어짐.

→ **5분 만에 4가지 결정적 단서 확보**. 본인 환경만 보고 추측하느니 훨씬 효율적.

## 학습 — 외부 의존 분해 패턴

5/4 일지의 "외부 의존 블로커가 사실 진척의 기회였다" 패턴 그대로:

| 막혔던 곳 | 우회 경로 |
|----------|-----------|
| 승랑님 미팅 일정 안 잡힘 | 그분 브랜치 5분 들여다보기로 환경 표준 확보 |
| 김이사님 회의 일정 안 맞음 | feat/rbac 브랜치에서 RBAC 작업 진행 상황 확인 |
| 빌드 환경 설정 모름 | upstream / 동료 브랜치의 `.nvmrc`/`package.json` 등 확인 |

→ **다른 사람 브랜치 = 회사 표준 + 컨벤션 + 진행 상황이 모두 들어있는 정보 창고**. 막혔다 싶으면 5분 투자할 가치 있음.

## 함정 — `git pull`로 받지 말기

```bash
# ❌ 위험
git pull origin feat/rbac
# → 현재 브랜치에 그분 변경사항 머지함. 본인 코드 오염 가능

# ✅ 안전
git fetch origin
# → origin/feat/rbac만 갱신, 본인 작업 디렉토리 영향 없음
```

`fetch`로 받고 `git show`/`git diff`로 보는 게 정석.

## 관련 노트
- [[Git - core.hooksPath와 Hook 위치 추적]]
- [[Git - 기본 워크플로우 명령어]]
- [[Git - 브랜치 전략 명령어]]
- [[husky - .husky 폴더 패턴과 install 시점]]
- [[1. Daily/2026-05-06.md]]
