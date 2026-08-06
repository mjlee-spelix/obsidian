---
tags: [dify, 성능, Flask-Login, keycloak, 백엔드, 보류]
date: 2026-05-11
status: 원인 추적 보류 (Dify upstream + keycloak patch 결합 영역)
related_files: ["api/extensions/ext_login.py", "api/libs/login.py"]
---

# Dify - 대시보드 API 백엔드 2.7초 병목 분석 (Flask-Login 캐시 미작동)

## 핵심

> 대시보드 Phase 1 API(1요청당 2.7초) 측정 결과, **DB는 0.07%**, **백엔드 non-DB가 99.9%**.
> 백엔드 내부에선 **JWT 검증 + account/tenant 조회가 1요청당 59번 redundant 실행**되는 버그 발견.
> Flask-Login의 `g._login_user` 캐시가 한 번도 세팅되지 않음 — `_get_user()`가 매번 cache miss.
> **현재 결정**: 옵션 C(원인 추적 보류) — Dify upstream + 우리 keycloak patch + ssp RBAC 패치 결합 영역. 신입이 들어갈 risk 영역 아님.

## 발견 상황

- **시점**: 2026-05-11, 데이터 마트 1단계 2단계 측정 (레이어별 측정) 수행 중
- **목적**: "10초+ 응답 시간이 DB 때문인가?" 확인 → 마트화 가치 판정
- **부수 발견**: DB는 무죄(0.07%) 확정 후, 백엔드 어디가 느린지 추적하다가 발견

## 측정 방법 — 4-레이어 분해

### 1. 프론트 — F12 Performance
- Main thread 1st party: 478ms
- Scripting 743ms + Rendering 230ms + Painting 59ms
- 결론: **JS 작업은 ~1초, 9초는 서버 응답 대기**

### 2. 네트워크 — nginx `log_format` 패치 (가벼움)
- `/etc/nginx/nginx.conf`의 `log_format main`에 추가:
  ```
  rt=$request_time urt=$upstream_response_time uht=$upstream_header_time
  ```
- `nginx -s reload` → 모든 요청의 백엔드 응답 시간 자동 수집
- Flask 미들웨어 박을 필요 없음 (가장 가벼운 방법)
- **주의**: 컨테이너 안 직접 수정은 재시작 시 휘발. 영구화하려면 `docker/nginx/nginx.conf.template`도 수정

### 3. 백엔드 — nginx의 `$upstream_response_time`
- `rt ≈ urt ≈ uht` → 응답 본문 작아서 네트워크는 무시 가능
- urt 자체가 백엔드 총 시간

### 4. DB — `EXPLAIN ANALYZE` (psql 직접)
```bash
docker exec docker-db_postgres-1 sh -c "psql -U postgres -d dify -f /tmp/q.sql"
```
- 모든 쿼리 < 2ms (9-CTE 최복잡 쿼리도 1.5ms)

## 측정 결과 — 충격적 발견

### Phase 1 (메인 대시보드, 4개 동시 요청)

| 엔드포인트 | rt | DB | 백엔드 non-DB |
|---|---|---|---|
| `/dashboard/kpi` | 2.468s | ~7ms | ~2.46s |
| `/dashboard/dept-objects` | 2.712s | ~2ms | ~2.71s |
| `/dashboard/model-tokens` | 2.678s | ~3ms | ~2.67s |
| `/dashboard/dept-activity` | 2.719s | 1.4ms | ~2.72s |

### Phase 2 drill-through (클릭 시)

| 그룹 | rt 평균 | DB | 백엔드 non-DB |
|---|---|---|---|
| objects (3개) | 0.67s | ~1ms | ~0.67s |
| calls (3개) | 1.04s | ~1.5ms | ~1.04s |
| users (3개) | 0.67s | ~1ms | ~0.67s |
| errors (2개) | 0.54s | ~1.5ms | ~0.54s |

→ **DB 0.07% vs 백엔드 non-DB 99.9%**. 마트화는 현재 데이터 양에서 가치 0.

## 백엔드 99.9% 추적 — 59회 redundant load 발견

### 단서: api 컨테이너 로그 패턴

`docker logs docker-api-1`에 같은 req_id로 다음 패턴이 반복됨:

```
ddbdabd4bf decoded keys=['exp', 'iat', ...] sub=b7b98c73-...
ddbdabd4bf build_account OK sub=b7b98c73-... tenant=ff3ccc82-...
```

req_id는 `core/logging/context.py`의 `_request_id: ContextVar`로 관리되는 단일 요청 ID (10자 hex). ContextVar 격리되니까 같은 req_id = 단일 요청.

### 카운트

```bash
docker logs docker-api-1 --since 10m | grep "750794fb0b" | wc -l
# → 118 (decoded 59 + build_account OK 59)
```

**한 요청에 `load_user_from_request` 59번 호출**. 정상 시 1회/요청이어야 함.

### 호출 흐름

`api/extensions/ext_login.py`:
```python
@login_manager.request_loader
def load_user_from_request(request_from_flask_login):
    # KEYCLOAK_ENABLED path
    decoded = PassportService().verify_keycloak_token(auth_token)
    _kc_logger.info("decoded keys=%s sub=%s", ...)
    account = AccountService.build_account_from_keycloak(decoded)  # ← DB 쿼리
    _kc_logger.info("build_account OK ...")
    return account
```

`api/libs/login.py`:
```python
def _get_user():
    if has_request_context():
        if "_login_user" not in g:    # ← 매번 True (cache miss)
            _get_login_manager().load_user_from_request_context()
        return g._login_user
```

### 옵션 1 진단 시도 (5/11)

ext_login.py 시작부에 `g._login_user` 상태 확인 진단 라인 1줄 추가:

```python
_kc_logger.warning("LOAD_CALLED in_g=%s _login_user_type=%s path=%s",
                   "_login_user" in g,
                   type(getattr(g, "_login_user", None)).__name__,
                   request.path)
```

**결과**:
- `in_g=False` 매번 → cache가 한 번도 세팅 안 됨 확정
- 진단 코드의 `request.path` 접근이 또 `current_user` 트리거 → **무한재귀로 서비스 다운**
- 즉시 롤백

## 미해결 — 캐시 미작동 원인

Flask-Login 0.6.3 소스의 `_load_user` 흐름:
```python
def _load_user(self):
    ...
    user = self._load_user_from_request(request)  # request_loader 호출
    return self._update_request_context_with_user(user)  # ← g._login_user = user 세팅

def _update_request_context_with_user(self, user=None):
    if user is None:
        user = self.anonymous_user()
    g._login_user = user
```

코드 정적 분석상 `_update_request_context_with_user`가 호출되어 `g._login_user`를 세팅해야 함. 그런데 실제론 안 됨. 원인 후보:

1. `request_loader` return 후 흐름이 어딘가에서 끊김
2. SPX-Agent의 keycloak patch가 `_load_user` 흐름을 우회
3. ssp RBAC 패치가 g를 reset하는 코드 추가
4. gevent worker class + ContextVar 결합 부수 효과

**확정하려면**: `_load_user` 자체에 진단 라인 추가가 필요한데, Flask-Login 라이브러리 코드 수정이라 더 위험.

## 운영 영향 평가

- **개발 환경**: 백엔드 2.7초 (참기 어려움)
- **운영 환경**: 사용자 수 많아지면 DB connection pool 고갈 + worker thread 점유 → 응답 시간 폭발 가능성
- **데이터 양 영향**: 마트화로 줄일 수 있는 부분은 0.07%. 진짜 임팩트는 이 redundant load fix
- **사용 범위**: `console` blueprint 전체 (대시보드뿐 아니라 모든 admin API). 즉 전사 영향

## 보류 이유 (왜 옵션 C 선택)

1. **신입 책임 영역 아님**: Dify upstream + keycloak patch + ssp 패치 결합. 영향 추적이 복잡하고 다른 팀원 코드 영향
2. **Dify upstream 무수정 원칙**: [[4. 지식노트/Dify - PluginAutoUpgradeService DetachedInstanceError 버그 (1.13.3).md]]와 같은 정신. patch 면적 키우면 향후 dify 업그레이드 시 충돌 위험
3. **마트화 무가치 결론은 확정**: 추가 디버깅 없이도 회의 답변/설계 결정 가능
4. **수정 risk 큼**: 진단 1줄 추가에서도 무한재귀 만남. 본격 fix는 더 큰 risk
5. **상위 의사결정 필요**: 김이사님/팀과 공유 후 fix 범위 결정해야 할 영역

## 향후 추적 재개 시 출발점

다음 사람(또는 본인 미래)이 이 문제 다시 들어갈 때:

1. **첫 진단**: `_load_user`에 진단 라인 (Flask-Login 라이브러리 코드 수정). 무한재귀 피하려면 `request.path` 같은 LocalProxy 접근 절대 금지
2. **의심 1순위**: SPX-Agent의 keycloak patch가 `_update_request_context_with_user` 호출을 우회하는 path 있는지
3. **의심 2순위**: ssp RBAC 패치 (controllers/console/admin/*) 안에서 `g._login_user`를 pop하거나 user 객체 재할당하는 코드
4. **검증 방법**: nginx `$upstream_response_time` + api logs에서 `build_account OK` 카운트. cache 작동 시 → 1회/요청

## 임시 회피 — 거슬릴 때

옵션 2(hotfix) 1줄 패치:

```python
@login_manager.request_loader
def load_user_from_request(request_from_flask_login):
    del request_from_flask_login
    
    # Request-scoped manual cache (workaround for broken Flask-Login cache)
    if hasattr(g, '_dify_cached_account'):
        return g._dify_cached_account
    
    # ... existing code ...
    
    # Before return:
    g._dify_cached_account = account
    return account
```

→ 59회 → 1회 강제 축소. 백엔드에서 ~300ms 즉시 절감. 단 upstream patch라 적용 결정은 팀 공유 후.

## 보고할 가치 있는 사실 한 줄 요약

> "대시보드 API 1요청당 JWT 검증 + account/tenant 조회가 59번 redundant 실행되는 버그 발견. Flask-Login의 `g._login_user` 캐시 미작동. 운영 환경 사용자 응답 시간에 영향 가능성. 원인 추적 보류 (Dify upstream + 우리 keycloak patch 결합 영역)."

## 관련 노트

- [[4. 지식노트/Dify - PluginAutoUpgradeService DetachedInstanceError 버그 (1.13.3).md]] — 같은 옵션 C 정신 (upstream 무수정)
- [[4. 지식노트/Dify - 호스트 코드 mount 트레이드오프 (잠복 버그 노출).md]] — 환경 셋업 변경의 부수 효과 패턴
- [[3. 프로젝트/spx-agent/references/dashboard-query-inventory.md]] — 측정 대상 엔드포인트 인벤토리
- [[1. Daily/2026-05-11.md]] — 측정 수행 일지
