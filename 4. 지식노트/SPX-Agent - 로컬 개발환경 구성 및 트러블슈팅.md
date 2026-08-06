---
tags: [dify, 개발환경, 트러블슈팅]
date: 2026-04-28
---
# SPX-Agent - 로컬 개발환경 구성 및 트러블슈팅

## 결론 (TL;DR)

```bash
# 1. 미들웨어 (DB, Redis, Weaviate 등)
cd docker
docker compose -f docker-compose.middleware.yaml up -d

# 2. Flask 백엔드 API (localhost:5001)
cd ..
./dev/start-api   # chroma 빌드 에러 나도 무시 — Flask는 정상 실행됨

# 3. 프론트엔드 (localhost:3000)
cd web
pnpm dev
```

접속: `http://localhost:3000`

> **주의**: `./dev/start-api`는 단순히 Flask 설치가 아님. 아래 세 가지를 한 번에 처리함:
> 1. 의존성 전체 설치 (Flask 포함 수백 개 패키지, uv 사용)
> 2. DB 마이그레이션 (`flask db upgrade`)
> 3. Flask 서버 실행 (`flask run --port=5001`)
>
> Flask만 따로 설치해서는 안 됨 — SQLAlchemy, Redis 클라이언트 등 없으면 서버 안 뜸

---

## 설정 파일 변경 사항

### `web/.env.local` — 변경 없음 (기본값 그대로)
```env
NEXT_PUBLIC_API_PREFIX=http://localhost:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://localhost:5001/api
HONO_CONSOLE_API_PROXY_TARGET=
HONO_PUBLIC_API_PROXY_TARGET=
```
Flask가 localhost:5001에서 직접 응답하므로 Hono 프록시 설정 불필요

### `docker/.env` — 변경됨
```env
KEYCLOAK_ENABLED=false   # 로컬 개발 시 Keycloak 비활성화 (토큰 발급자 충돌 방지)
```

---

## 삽질 전체 기록

### 초기 상황
- 목표: 프론트 코드 수정 후 로컬에서 바로 확인
- Docker 전체 스택으로 `localhost:80` 접속 중이었음
- `pnpm dev`로 로컬 Next.js 띄우려 함

---

### 문제 1: `localhost:5001` NetworkError
```
NetworkError: GET http://localhost:5001/console/api/system-features
```
- Claude 진단: "Docker api 컨테이너가 5001 포트를 호스트에 노출 안 해서"
- 실제 원인: **Flask 자체가 실행 안 되고 있었던 것**
- 시도한 방법들 (전부 불필요):
  1. `.env.local`을 `http://localhost/console/api` (nginx 80번)으로 변경
  2. `docker compose up -d nginx` 추가
  3. docker-compose.override.yml로 5001 포트 노출 제안

---

### 문제 2: nginx Bad Gateway (502)
- 원인: nginx가 Docker web 컨테이너로 연결하려는데 web 컨테이너 없음
- Keycloak 비활성화로 넘어감

---

### 문제 3: Keycloak 인증 실패
```
인증 방법이 구성되지 않음
```
- 원인: 토큰 발급자 불일치
  - 발급: `http://localhost:8180/realms/dify`
  - 기대: `http://keycloak:8080/realms/dify`
- 해결: `docker/.env`에서 `KEYCLOAK_ENABLED=false`

---

### 문제 4: CORS 에러
```
[object Response] / unhandledRejection
```
- 원인: 브라우저(`:3000`) → nginx(`:80`) cross-origin 충돌
- 시도한 방법 (불필요):
  - Hono 프록시 설정 (`HONO_CONSOLE_API_PROXY_TARGET=http://localhost/console/api`)
  - `pnpm dev:proxy` 별도 실행

---

### 문제 5: `./dev/start-api` chroma 빌드 에러
```
× Failed to build `chroma-hnswlib==0.7.6`
error: Microsoft Visual C++ 14.0 or greater is required.
```
- Claude가 한 말: "Windows C++ 빌드 문제라 로컬 백엔드 실행 어렵다"
- 실제 상황: **Flask는 정상 실행됐음**
  - `VECTOR_STORE=weaviate`이라 chroma 코드가 임포트 자체 안 됨
  - chroma 에러는 치명적 에러가 아니라 경고 수준
- Claude가 제시한 불필요한 해결책:
  1. Visual C++ Build Tools 설치
  2. WSL2 사용 권장
  3. Docker volume mount
  4. 백엔드는 Docker로만 쓰라고 권장

---

## 근본 원인

**처음부터 `./dev/start-api` 실행하면 됐다.**

- `curl http://localhost:5001/console/api/system-features` 로 포트 확인만 먼저 했어도 바로 해결됐을 것
- chroma 에러가 `×` 기호로 치명적 에러처럼 보였지만 실제로는 비치명적 경고
- Flask 실행 확인 없이 엉뚱한 방향(nginx, Hono 프록시 등)으로 계속 유도한 게 문제

---

## 관련 노트
- [[4. 지식노트/Dify - 설정 모달(account-setting) 구조.md]]
