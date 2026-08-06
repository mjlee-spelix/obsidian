---
tags: [Keycloak, 인증, 디버깅, CAND]
date: 2026-05-07
---
# Keycloak — 토큰 만료 시 API CPU 100% 행 패턴

## 핵심

브라우저 세션의 JWT 토큰이 만료된 채로 자동 refresh 안 되면, 프론트가 만료 토큰으로 여러 endpoint를 동시 호출. **Keycloak이 unhealthy 상태라 검증 응답이 느림** → API 워커 누적 → CPU 100% 행. 화면은 무한 로딩.

## 발견 사례 (2026-05-07)

### 증상
- localhost 로딩 매우 느림 (페이지 자체는 1.8~2초인데 안 뜸)
- `docker stats`: api 컨테이너 **CPU 101%**, web 컨테이너 0%
- nginx 로그: `/console/api/system-features` 요청 → **499 (클라이언트 취소)** 반복

### 결정적 단서
api 로그에서 같은 토큰으로 같은 millisecond에 15회+ 반복:
```
ERROR [ext_login.py:85] verify_keycloak_token FAILED:
401 Unauthorized: Keycloak token has expired
```

### 동시 조건
- `docker-keycloak-1: Up 4 hours (unhealthy)` ← healthcheck 실패 (다만 8080 listen + Realm import 정상)
- 4시간 전 로그인한 세션의 JWT 만료
- 자동 refresh 흐름이 어떤 이유로 막힘 (Keycloak unhealthy 영향 추정)

## 메커니즘 추정

1. 사용자 브라우저 → 페이지 진입 → 만료 JWT 그대로 사용
2. 프론트가 12종 endpoint 동시 호출 (KPI/dept-objects/.../drill-through 등)
3. 각 호출마다 API → Keycloak에 토큰 검증 HTTP 요청
4. Keycloak이 unhealthy 상태라 응답 느림 (5~10초)
5. gunicorn worker별로 검증 대기 누적 → 다른 요청 처리 못 함
6. 사용자가 "로딩 중"으로 인식 → 새로고침 → 또 12종 호출 → 누적 가속
7. CPU 100% 행

## 해결

### 1단계 (즉시) — 사용자 측

```
브라우저 시크릿/InPrivate 창 열기 → 새 로그인 → 새 JWT 발급
```

저장된 만료 토큰 우회가 가장 빠름. 일반 창에서 로그아웃 → 재로그인도 가능하지만 만료 토큰 잡고 있는 상태에서 로그아웃 자체가 막힐 수 있음.

### 2단계 (API 행 정리) — 운영자

```powershell
docker compose stop -t 5 api worker worker_beat
docker compose start api worker worker_beat
```

CPU 100% 워커 강제 종료. 1분 정도 후 healthy 회복.

### 3단계 (Keycloak 자체 문제 시)

```powershell
docker compose restart keycloak
```

Keycloak healthcheck 명령 실패만이고 실제 동작은 OK였더라도, 토큰 갱신 흐름에서 막히면 재시작 필요. 30~50초 소요.

## 방어

### 정기 모니터링

`docker ps | grep keycloak | grep unhealthy` 발견 시 경계. 4시간 이상 unhealthy 상태면 재시작 검토.

### 운영 시 재시작 주기

unhealthy 상태가 잦으면 일 1회 또는 시간 N회 자동 재시작 cron 검토.

## 진단 우선순위

API 무한 로딩 증상 시 점검 순서:
1. `docker stats` — CPU 100% 컨테이너 식별
2. CPU 100% 컨테이너의 로그 → 반복 ERROR 패턴 검색
3. **Keycloak 토큰 만료 메시지** 발견 시 → 시크릿 창 + API 재시작
4. 다른 ERROR면 다른 진단 흐름 (silent fallback / DB 행 / 등)

## 검증 필요 (CAND 단계)

- 1차 발생만 검증됨 (2026-05-07)
- Keycloak unhealthy + 4시간 경과 + 토큰 만료 = 3가지 조건 동시 발생이 핵심 트리거인지 미완전 검증
- 자동 refresh가 왜 안 됐는지 미확인 (Keycloak 자체 문제 vs Dify 프론트 흐름 문제)
- 1건 추가 발생 시 정식 H-ENV-04 또는 별도 ID 승격

## 관련 노트

- [[4. 지식노트/Python - silent fallback과 logger.exception 의무]] — 같은 진단 우선순위 패턴

## 메타 — "unhealthy" 신호 무시 금지

5/7 오전 *"healthcheck 명령만 어긋나고 동작은 OK"* 라고 Keycloak unhealthy 무시 → 4시간 후 토큰 만료 시점에 폭발. **healthy 신호는 시점 의존적**. 지금 OK여도 미래 OK 보장 X. unhealthy는 *"잠재 문제 신호"* 로 취급해야.
