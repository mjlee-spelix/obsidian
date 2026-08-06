---
tags: [개발, Git, husky, CS]
date: 2026-05-06
---
# Git - core.hooksPath와 Hook 위치 추적

## 핵심
- **Git hook을 찾을 땐 `.git/hooks/`만 보면 안 된다**
- `git config core.hooksPath` 설정으로 **다른 위치로 redirect** 되어 있을 수 있음
- husky / lefthook / simple-git-hooks 같은 도구가 비표준 위치에 hook 배치
- 진단 첫 명령: `git config --get core.hooksPath`

## Git hook이 도는 위치 — 우선순위

1. `core.hooksPath` 설정값 (있으면 **이게 절대 우선**)
2. 없으면 → 기본 `.git/hooks/`

```bash
# 어디서 도는지 확인
git config --get core.hooksPath

# 결과 예시
# (빈 출력)        → 기본 .git/hooks/ 사용
# .husky/_         → husky가 설정함
# .vite-hooks/_    → 회사 커스텀 설정
# .lefthook/_      → lefthook 도구
```

## 흔한 함정

### "분명 hook이 설정돼있는데 발동 안 한다"

원인 후보:
1. `core.hooksPath`가 **다른 경로** 가리키는데 거기 폴더가 비어있음
2. hook 파일은 있는데 **실행 권한 없음** (Unix/macOS)
3. `pre-commit install` 같은 framework 활성화 명령을 안 돌림

### ".git/hooks/는 비어있는데 commit 시 lint가 도는 이유?"

`core.hooksPath`가 다른 곳 가리키는 것. 진단:

```bash
git config --list --show-origin | grep hooks
# file:.git/config        core.hookspath=.husky/_

ls .husky/_/
# applypatch-msg  commit-msg  pre-commit  pre-push  ...
```

## 비표준 위치 패턴 모음

| 도구 | 경로 | 추적되는 부분 |
|------|------|--------------|
| **husky v9+** | `.husky/_/` | `.husky/<hook-name>` (사용자 정의)는 추적, `.husky/_/` 자동 생성은 미추적 |
| husky v8 이하 | `.husky/` 직접 | 모두 추적 |
| **lefthook** | `.git/hooks/` 그대로 | `lefthook.yml` 추적 |
| simple-git-hooks | `.git/hooks/` 그대로 | `package.json`의 `simple-git-hooks` 필드 |
| **회사 커스텀** | `.vite-hooks/_/`, `.scripts/git-hooks/`, 등 | 다양 |

## 진단 흐름 (실전)

```bash
# 1. hook 경로 확인
git config --get core.hooksPath
# .vite-hooks/_

# 2. 그 폴더 실제 내용
Get-ChildItem .vite-hooks/_ -Recurse
# 또는: ls -la .vite-hooks/_

# 3. 진짜 hook 스크립트 (사용자 정의 부분)
Get-ChildItem .vite-hooks/
# pre-commit (실제 스크립트, 추적됨)
# _/         (자동 생성, 미추적)

# 4. hook 스크립트 내용
Get-Content .vite-hooks/pre-commit
# → ruff check / eslint / 등 실행 명령 발견

# 5. git에 추적되는지
git ls-files .vite-hooks/
# .vite-hooks/pre-commit    ← 추적됨

git status .vite-hooks/
# (clean)                    ← .gitignore로 _/ 제외
```

## 누가 박았는지 추적

```bash
git log --diff-filter=A --oneline -- .vite-hooks/pre-commit
# 추가된 commit 확인 → 작성자 추적

git log --oneline -10 .vite-hooks/
# 최근 변경 이력
```

## hook 일시 비활성

### 단발성 우회
```bash
git commit --no-verify -m "..."

# 또는 더 명시적으로
git -c core.hooksPath=/dev/null commit -m "..."
# Windows: git -c core.hooksPath=NUL commit -m "..."
```

### ⚠️ 절대 하지 말 것

```bash
git config --unset core.hooksPath   # 팀 차원 lint 게이트를 본인이 끄는 셈
```

→ 본인 환경에서만 비활성. 팀 내 다른 사람은 그대로 hook 발동. **개인이 멋대로 끄면 팀 차원 검증이 본인에서만 안 도는 silent drift 발생**.

## 실전 — 발견 사례 (2026-05-06)

SPX-Agent 프로젝트에서 본인 commit이 ruff 위반으로 막힘. 그런데:
- `.git/hooks/` → sample 파일만 있고 비어있음
- `.git/hooks/pre-commit` 파일 자체가 없음
- 그래도 commit 시 `Running Ruff linter on api module` 출력

진단 결과:
```bash
git config --get core.hooksPath
# .vite-hooks/_

git config --list --show-origin | grep hooks
# file:.git/config    core.hookspath=.vite-hooks/_
```

→ **husky의 폴더 이름만 `.vite-hooks`로 변경한 회사 커스텀 패턴**. `.husky/`와 동일 구조.

이 경우 hook이 본인 환경에서만 활성화된 이유는 **`pnpm install`의 prepare script 시점**에 따라 다름 — [[husky - .husky 폴더 패턴과 install 시점]] 참조.

## 학습 — 진단 체크리스트

새 프로젝트 / 새 환경 세팅 시 다음 순서로 hook 상태 파악:

1. ✅ `git config --get core.hooksPath` — 설정값 확인
2. ✅ 그 경로(또는 `.git/hooks/`) 실제 내용 확인
3. ✅ 실제 hook 스크립트 내용 (`type` / `cat`)
4. ✅ `package.json` / `pyproject.toml`의 hook framework 의존 확인
5. ✅ `pre-commit install` / `husky install` 같은 활성화 명령 한 번 돌렸는지

## 관련 노트
- [[husky - .husky 폴더 패턴과 install 시점]]
- [[Git - 다른 브랜치를 체크아웃 없이 들여다보는 방법]]
- [[Python - logger.error vs logger.exception (ruff TRY400)]]
- [[Git - 기본 워크플로우 명령어]]
- [[1. Daily/2026-05-06.md]]
