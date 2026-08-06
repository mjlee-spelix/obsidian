---
tags: [개발, Node, pnpm, corepack, CS]
date: 2026-05-06
---
# Node - corepack과 패키지 매니저 버전 통일

## 핵심
- **corepack** = Node.js에 내장된 패키지 매니저 버전 관리 도구 (Node 16.10+ 표준 포함)
- `package.json`의 `packageManager` 필드를 읽어 **해당 버전의 pnpm/yarn/npm을 자동 다운로드 + 사용**
- 글로벌 설치 불필요. 모든 개발자가 동일 버전 사용 → **개발자 환경 drift 차단**

## 왜 필요한가 — 환경 drift 문제

### 글로벌 설치 시 발생하는 문제

```bash
# 개발자 A (회사 신입)
npm install -g pnpm    # 최신 → pnpm 10.x.x 설치

# 개발자 B (3개월 전 입사)
npm install -g pnpm@7  # 그때 회사 표준이 7

# 개발자 C
# pnpm 안 쓰고 npm 그대로
```

→ **같은 프로젝트, 다른 패키지 매니저 → `lockfile` 충돌, 빌드 차이, 디버깅 지옥**.

### corepack 사용 시

```json
// package.json
{
  "packageManager": "pnpm@10.33.0"
}
```

```bash
corepack enable           # 한 번만
cd my-project
pnpm install              # corepack이 10.33.0 자동 사용 → 모두 동일
```

## 활성화 방법

### 1. corepack이 있는지 확인

```bash
corepack --version
# 0.34.6  ← Node 22 기본 포함
```

없으면 Node가 너무 옛날 버전 (16.10 미만). Node 업그레이드 필요.

### 2. 활성화

```bash
corepack enable
```

- 한 번만 실행. Windows에선 가끔 관리자 권한 필요.
- 활성화 후 `pnpm` / `yarn` / `npm` 명령이 corepack을 거쳐 호출됨.

### 3. 프로젝트 사용

```bash
cd <project-with-packageManager-field>
pnpm install
# → corepack이 packageManager 필드 읽고 해당 버전 fetch (처음만)
# → 이후 캐시된 버전 사용
```

## 실전 — nvm으로 Node 버전 바꿨을 때 함정

```bash
nvm install 22
nvm use 22
pnpm install
# pnpm: command not found
```

**원인**: nvm은 Node 버전마다 글로벌 패키지를 **별도로 관리**. Node 20에 깔린 글로벌 pnpm은 Node 22 환경에 안 따라옴.

**해결 (corepack 권장)**:
```bash
corepack enable    # Node 22 환경에 corepack 활성화
pnpm install       # 자동으로 packageManager 버전 사용
```

이게 가장 깔끔. **글로벌 설치 안 해도 됨**.

## 글로벌 설치 vs corepack 비교

| 항목 | `npm install -g pnpm` | corepack |
|------|----------------------|----------|
| 모든 프로젝트에 같은 버전 | ✅ | ❌ (프로젝트별 다름) |
| 프로젝트 표준 자동 매칭 | ❌ | ✅ |
| nvm Node 전환 후 재설치 필요 | ✅ | ❌ |
| 신입 셋업 단계 | 수동 | `corepack enable` 한 번 |
| 회사 표준 강제력 | 약함 | 강함 (`packageManager` 필드) |

→ **회사/팀 단위에선 corepack이 정답**. 개인 사이드 프로젝트는 글로벌도 OK.

## packageManager 필드 작성

```json
{
  "packageManager": "pnpm@10.33.0"
}
```

문법:
- `<manager>@<version>` 형식
- 정확한 버전 박기 권장 (`^` / `~` 안 됨, corepack은 정확 매치)
- `npm@10.x.x`, `yarn@4.x.x`, `pnpm@10.x.x` 모두 가능

### 설정 명령으로 추가

```bash
corepack use pnpm@10.33.0
# → package.json에 packageManager 필드 자동 추가
```

## 모노레포에선 어디에 박나

루트 `package.json`에 한 번만:

```
my-monorepo/
├── package.json          ← "packageManager": "pnpm@10.33.0"
├── apps/
│   ├── web/
│   │   └── package.json  ← 없어도 됨
│   └── api/
│       └── package.json  ← 없어도 됨
```

`pnpm` 명령은 가장 가까운 상위 `package.json`을 찾아 올라감 → 루트 한 곳만 박으면 충분.

## 실전 — 발견 사례 (2026-05-06)

SPX-Agent 프로젝트:
```bash
git show origin/feat/rbac:package.json | grep packageManager
# "packageManager": "pnpm@10.33.0"
```

→ 회사 표준 = pnpm 10.33.0 확정. 본인 호스트 Node 22 업그레이드 후:

```bash
corepack enable
cd web && pnpm install     # corepack이 10.33.0 자동 fetch
```

5분 안에 환경 일치 완료. 글로벌 pnpm 설치 시도 시 발생할 수 있는 **버전 충돌 우회**.

## 함정 — Windows에서 corepack 권한

`corepack enable`이 다음 에러 뱉을 수 있음:

```
EPERM: operation not permitted
```

해결:
1. **관리자 PowerShell 한 번 열어서** `corepack enable`
2. 일반 권한으로 복귀해서 사용

또는:
```bash
corepack prepare pnpm@10.33.0 --activate
```
이게 안 먹으면 글로벌 fallback (단, drift 위험 인지하고 사용).

## 관련 노트
- [[husky - .husky 폴더 패턴과 install 시점]]
- [[Git - core.hooksPath와 Hook 위치 추적]]
- [[SPX-Agent - 로컬 개발환경 구성 및 트러블슈팅]]
- [[Docker - 호스트 코드를 컨테이너에 반영하는 패턴]]
- [[1. Daily/2026-05-06.md]]
