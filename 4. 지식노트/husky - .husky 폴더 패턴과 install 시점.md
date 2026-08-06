---
tags: [개발, Git, husky, CS, 협업]
date: 2026-05-06
---
# husky - .husky 폴더 패턴과 install 시점

## 핵심
- husky v9+는 **두 종류 폴더**로 분리:
  - `.husky/<hook-name>` — 사용자 정의 hook 스크립트, **git에 추적됨**
  - `.husky/_/` — 자동 생성 wrapper + helper, **`.gitignore`로 미추적**
- `.husky/_/`는 **`pnpm install`의 prepare script로 자동 생성**됨
- **install 안 한 개발자는 hook 미발동** — silently skipped, 본인은 모름

## 폴더 구조

```
.husky/
├── pre-commit            ← 사용자 정의 (추적됨, 모든 사람 받음)
├── commit-msg            ← 사용자 정의 (추적됨)
└── _/                    ← 자동 생성 (미추적, 개인별)
    ├── .gitignore        ← "*"  (이 폴더 통째로 무시)
    ├── h                 ← helper script (624 bytes 정도)
    ├── pre-commit        ← `. "$(dirname "$0")/h"` wrapper (39 bytes)
    ├── commit-msg
    ├── post-commit
    ├── ...               ← 모든 git hook 종류에 대해 wrapper 생성
```

### 두 종류 pre-commit의 역할

| 파일 | 크기 | 추적 | 역할 |
|------|------|------|------|
| `.husky/pre-commit` | 사용자 작성 | ✅ git에 추적 | 실제 lint/test 명령 |
| `.husky/_/pre-commit` | 39 bytes | ❌ `.gitignore` | git이 호출하는 wrapper. 위 파일을 호출 |

git은 `core.hooksPath=.husky/_`를 따라 wrapper를 호출 → wrapper가 helper(`h`) 거쳐 사용자 정의 hook 호출 → 그 안에서 lint 등 실행.

## 활성화 메커니즘 — `prepare` script

### package.json의 prepare 스크립트

```json
{
  "scripts": {
    "prepare": "husky"
  }
}
```

- `pnpm install` (또는 `npm install`/`yarn install`) 실행 시 자동으로 `prepare` script가 돈다
- husky 9+에선 `husky` 명령 자체가 `.husky/_/` 폴더 + wrapper들을 자동 생성하고 `git config core.hooksPath` 설정도 박음

### 흐름

```
개발자 A: pnpm install 실행
   ↓
prepare script 자동 실행 → husky 명령 호출
   ↓
.husky/_/ 폴더 생성 + git config core.hooksPath=.husky/_
   ↓
개발자 A 환경에서 hook 활성화 ✅
```

## 함정 — install 시점 차이

### 시나리오: 개발자 B는 hook이 안 돈다

```
개발자 B: 처음 clone 후 pnpm install 안 돌리고 작업 시작
   ↓
.husky/_/ 미생성, core.hooksPath 미설정
   ↓
개발자 B 환경에서 hook 미발동 ❌
   ↓
lint 위반 코드 그대로 commit + push
   ↓
master에 위반 진입
```

→ 이후 hook 활성화된 다른 개발자가 commit 시도 → **그 사람 commit이 막힘** (남의 위반에).

### 사후 발견 패턴

본인이 commit 시도 → ruff/eslint hook이 master의 기존 위반 코드를 잡음 → "내 코드 아닌데 왜 막혀?"

### 진단

```bash
# 본인 환경 hook 활성화 상태
git config --get core.hooksPath
ls .husky/_/
```

`.husky/_/`가 비어있거나 없으면 → 본인은 hook 미활성. 그 상태에서 commit한 사람들이 만든 위반이 사후 발견되는 구조.

## 회사 커스텀 — 폴더 이름 rename

회사/팀이 `.husky/`를 `.vite-hooks/` 같은 이름으로 변경한 사례:

```
.vite-hooks/
├── pre-commit          ← 추적 (사용자 정의)
└── _/                  ← 자동 생성
    ├── .gitignore
    ├── h
    └── pre-commit, commit-msg, ...
```

```bash
git config --get core.hooksPath
# .vite-hooks/_
```

→ **husky 폴더만 rename된 형태**. 동작은 동일. 폴더 이름 보고 husky 패턴인지 식별:
- `.gitignore` 파일이 `_/` 안에 있고 내용이 `*`
- `_/` 안에 모든 hook이 `. "$(dirname "$0")/h"` wrapper로 동일
- **= husky 패턴 100%**

## 일반 개발자가 알아야 할 것 — 신입 셋업

### 표준 셋업 순서

```bash
git clone <repo>
cd <repo>
pnpm install              # ← 이 단계에서 prepare script가 husky 활성화
git config --get core.hooksPath   # 활성화 확인
```

`pnpm install` 까먹으면 → hook 미발동 → 위반 commit 가능.

### 셋업 검증 체크리스트

```bash
# 1. core.hooksPath 설정됐는지
git config --get core.hooksPath
# .husky/_  또는 .vite-hooks/_ 같은 출력

# 2. 자동 생성 폴더 존재
ls .husky/_/    # 또는 .vite-hooks/_/

# 3. 실제 hook 스크립트
cat .husky/pre-commit    # 어떤 검사 도는지

# 4. 테스트 commit (안전한 변경으로)
git commit -m "test" --allow-empty
# → hook 메시지(예: "Running ruff...") 떠야 정상
```

## CI lint job — 진짜 안전망

husky hook은 **개인 환경 의존**이라 신뢰할 수 있는 게이트가 아님:

| 게이트 | 강제력 | 우회 가능성 |
|-------|-------|-------------|
| husky hook | 약함 | install 안 함 / `--no-verify` / 환경 mismatch |
| **CI lint job** | **강함** | PR/머지 차단, 우회 거의 불가 |

→ 회사가 정말 lint 강제하려면 **CI에 동일 lint 명령 박아야 함**. husky만 의존하면 silent drift 발생.

## 실전 — 발견 사례 (2026-05-06)

SPX-Agent 프로젝트:

```bash
git config --get core.hooksPath
# .vite-hooks/_

ls .vite-hooks
# pre-commit (2548 bytes, 추적됨)
# _/         (4-28 생성)

ls .vite-hooks/_/
# .gitignore  h  pre-commit  ... (전형적 husky 구조)
```

→ husky 폴더만 `.vite-hooks`로 rename된 상태. 본인은 `pnpm install` 시 활성화됐고, `feat/rbac` 작업자(권수현님)는 활성화 시점이 달라 자기 코드의 ruff TRY400 위반을 사후 발견 못함. 본인이 처음으로 발견 → 슬랙 알림 + `--no-verify` 우회 commit으로 진행.

## 학습 — Generator-Evaluator 분리 측면

husky hook은 **Generator(개발자)와 Evaluator(lint)를 같은 머신에서 돌림**. 본인 환경에 의존.

CI lint는 **Evaluator를 별도 머신(GitHub Actions / Jenkins)에서 돌림**. 환경 독립.

→ 진짜 강한 검증은 **Evaluator를 분리해야** 가능. husky는 빠른 피드백용, CI가 진짜 게이트.

## 관련 노트
- [[Git - core.hooksPath와 Hook 위치 추적]]
- [[Node - corepack과 패키지 매니저 버전 통일]]
- [[Python - logger.error vs logger.exception (ruff TRY400)]]
- [[Git - 다른 브랜치를 체크아웃 없이 들여다보는 방법]]
- [[1. Daily/2026-05-06.md]]
