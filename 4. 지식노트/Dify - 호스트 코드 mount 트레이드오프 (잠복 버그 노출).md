---
tags: [Dify, Docker, SQLAlchemy, 아키텍처, 트레이드오프]
date: 2026-05-07
related_files: ["docker/docker-compose.override.yaml", "api/services/app_dsl_service.py", "api/services/plugin/plugin_auto_upgrade_service.py"]
---
# Dify - 호스트 코드 mount 트레이드오프 (잠복 버그 노출)

## 핵심

Dify 베이스 이미지(`langgenius/dify-api:1.13.3`)에 호스트의 **신 dify 1.13.3 코드**를 mount해서 우리 신규 모듈을 노출시키면, **호스트 신 코드에 박힌 잠복 버그까지 같이 활성화**되어 베이스 이미지 단독으로는 안 보였던 `DetachedInstanceError` 등이 폭발. mount 통째 vs 부분 vs 제거의 트레이드오프 결정 필요.

## 메커니즘

| 환경 | 실행되는 dify 본체 코드 | 우리 신규 모듈 | 잠복 버그 |
|---|---|---|---|
| 원본 dify (mount 없음) | 베이스 이미지 옛 코드 | 실행 안 됨 | 안 발동 (베이스 이미지 검증된 버전) |
| 부분 mount (우리 모듈만) | 베이스 이미지 옛 코드 + 일부 호스트 신 | 부분 노출 | 부분 활성 (mount된 영역만) |
| 통째 mount (`../api:/app/api`) | 호스트 신 1.13.3 코드 | 전체 노출 | **전체 활성** |

핵심 — 베이스 이미지 1.13.3과 호스트 신 1.13.3 코드가 **같은 버전 번호인데도 내용 다름**. dify upstream에서 SQLAlchemy 안티패턴 같은 잠복 버그를 도입한 commit이 호스트 신 코드엔 있고, 베이스 이미지 빌드 시점 코드엔 없음.

## 발생 타임라인

> mount 결정 시점부터 옵션 B 트리거 도래까지 추적. 다음 결정 시점에 비용/이득 재계산용.

### 2026-04-29 이전 — Individual file mount (A 패턴)
- base `docker-compose.yaml`에 **18개 파일 individual mount** 박혀있던 상태 (사용자 기존 패턴)
  - `passport.py`, `keycloak_admin.py`, `account_service.py`, `feature_service.py`, `controllers/console/{__init__, setup, wraps, auth/*, workspace/*}.py`, `configs/feature/__init__.py` 등
- 신규 모듈 추가 시마다 mount 한 줄씩 추가 패턴
- 잠복 버그 노출 0 (필요한 파일만 노출)

### 2026-05-04 — mount 통째화 결정 (전환점)

**시간 흐름**:
1. mock 컨트롤러 검증 시작 → `admin/dashboard.py` 부팅 실패 — `ImportError: cannot import name 'dashboard' from 'controllers.console.admin'`
2. **시도 1**: 기존 individual mount 패턴으로 `admin/__init__.py` 추가 → admin/__init__.py가 호스트 신 `billing_service.py` 의존 → `ImportError: LangContentDict`
3. **시도 2**: 부분 mount (`admin/` 폴더만) → 같은 LangContentDict 에러 (의존 체인 깊음)
4. **시도 3 — 채택**: 통째 mount (`../api:/app/api`) — graphon/LangContentDict 등 의존 체인 **한 번에 해결**
5. **H-ENV-02 등록**: `defect-catalog`에 *"api/worker 서비스에 build 지시 없어 호스트 코드 변경 미반영"* 정식 등록
6. **`docker-compose.override.yaml` 신설**: 처음엔 `controllers/`, `services/`, `tests/` 3폴더 부분 mount → 의존 체인 깨짐 → 통째 mount로 전환

**부수 효과 — 잠복 버그 2건 활성화** (같은 날):
- `plugin_auto_upgrade_service.py:get_strategy()` `DetachedInstanceError` (row 있을 때 발동)
- `app_dsl_service.py` DSL import `expire_on_commit` 누락

**결정**: 옵션 A 유지 (mount + 버그 case-by-case patch). 트리거 *"2회 더 발견 시 옵션 B 재검토"* 명시.

### 2026-05-04 — A 패턴 individual mount 정리
- 통째 mount로 기존 18개 individual mount redundant 상태
- base `docker-compose.yaml`에서 38줄 제거
- override.yaml 통째 mount(B 패턴)로 통합

### 2026-05-04 ~ 5/6 — 안정 기간
- mount 관련 신규 함정 발견 0
- 5/6 다른 사건들 (vitest Node 22 / RBAC 명명 H-DASH-17) — mount 무관

### 2026-05-07 오전 — `CAND-git-checkout-with-running-container`
- 컨테이너가 mount 잡은 채 git 브랜치 전환 → 워킹트리 `.py` 파일이 빈 디렉토리로 깨짐
- **mount의 부수 효과로 발견된 새 함정** (잠복 버그는 아니지만 mount + git 동시성 문제)
- defect-catalog `CAND-git-checkout-with-running-container` 등록

### 2026-05-07 오후 — 잠복 버그 2차 발견 (트리거 도래)
- `app_dsl_service.py:474` `_create_or_update_app` `DetachedInstanceError`
- 5/4 trigger *"2회 추가 발견"* 카운트 = **2건째** (5/4 자체 2건은 1차 묶음, 5/7이 2차)
- → **트리거 도래 직전**: 추가 1건이면 옵션 B 즉시 진행

### 다음 시점 (예정)
- chart-drawer + context-bar 완료 → Phase 2 안정화 시점에 옵션 B 진지 검토
- 잠복 버그 추가 1건 발견 → 트리거 충족 → 즉시 옵션 B

## 안티패턴 잔존 위험 매트릭스

| 영역 | 안티패턴 건수 | 발동 위험 (우리 사용 흐름 한정) |
|---|---|---|
| 발동된 영역 — `plugin_auto_upgrade`, `app_dsl_service` | 3건 (2건 fix, 1건 fix 예정) | ✅ 검증됨 |
| `async_workflow_service.py` (line 240, 266, 289) | 3건 | **중** — workflow 모드 앱 사용 시 트리거 가능 (mock에 workflow_runs 75건) |
| `clear_free_plan_tenant_expired_logs.py` (line 123, 163, 295, 346, 398) | 5건 | 낮 — 로그 clean job, 우리 흐름 무관 |
| `credit_pool_service.py` (line 74) | 1건 | 낮 — billing/credit 영역 |
| `account_service.py` (line 1713) | 1건 (`expire_on_commit=False` 명시) | 매우 낮 — 안전 옵션 박혀있음 |

→ **추가 발동 위험 = `async_workflow_service` 3곳**. workflow 앱 검증 시 trigger 가능. 발견 시 옵션 B 즉시 트리거 충족.

## 발견 사례

### 1차 (2026-05-04)

mock 컨트롤러 검증 위해 `docker-compose.override.yaml`에 호스트 `api/` 통째 mount 추가 → 같은 날 2건 폭발:

1. `services/plugin/plugin_auto_upgrade_service.py:get_strategy()` — `with sessionmaker.begin() as session: return obj` 안티패턴. row 있는 환경에서만 발동. `DetachedInstanceError`. → [[4. 지식노트/Dify - PluginAutoUpgradeService DetachedInstanceError 버그 (1.13.3)]]
2. `services/app_dsl_service.py` 어딘가 — DSL import expire_on_commit 누락. 같은 패턴.

### 2차 (2026-05-07)

`services/app_dsl_service.py:474` `_create_or_update_app` — `self._session.commit()` 후 detached `app` 객체를 `app_was_created.send(app, ...)` blinker signal로 핸들러 전달 → 핸들러가 `app.tenant_id` 접근 시 lazy load → 세션 없음 → `DetachedInstanceError`. 1차의 같은 가족.

### 안티패턴 잔존 grep 결과

```bash
grep -rn "with sessionmaker" api/services/
```

10+ 곳 잔존: `account_service.py`, `async_workflow_service.py` (3곳), `clear_free_plan_tenant_expired_logs.py` (5곳), `credit_pool_service.py` 등. 추가 발견 가능성 큼.

## 옵션 비교

### 옵션 A — 통째 mount 유지 + 발견 시마다 host patch (현재 결정)

```python
# 예시 — _create_or_update_app fix
self._session.add(app)
self._session.commit()
self._session.refresh(app)        # ← 추가
app_was_created.send(app, account=account)
```

| 측면 | 평가 |
|---|---|
| 즉시 작업 시간 | 5분/건 |
| 누적 비용 | N건 발견 시 N×5분 + 다음 dify 업그레이드 시 patch 추적/충돌 |
| dev 흐름 | 빠름 (mount 통해 호스트 변경 즉시 반영) |
| 깔끔함 | 낮음 (dify 본체 patch 누적) |
| 미발견 위험 | 중 (호스트 신 코드 전체 노출) |

**적합한 시나리오**: 발견되는 잠복 버그가 적고(< 5건), Phase 2 같이 빠른 진행이 필요할 때.

### 옵션 B — mount 제거 + 부분 mount + 호스트 신 코드 의존 끊기

| 단계 | 내용 | 시간 |
|---|---|---|
| 1 | 의존 체인 분석 — mount 제거 시 깨지는 import 추적 (`admin/__init__.py` → 호스트 신 `billing_service.py` 의존, `graphon`, `LangContentDict` 등) | ~30분 |
| 2 | 부분 mount 재설계 — `docker-compose.override.yaml`에 우리 신규 모듈만 (`services/admin/`, `controllers/console/admin/`, `tests/services/admin/`) | ~20분 |
| 3 | 호스트 신 코드 의존 끊기 — 우리 모듈 안에서 호스트 의존 제거 또는 conditional import. dify 본체 = 베이스 이미지 옛 코드 | ~60분 |
| 4 | 누적 patch 제거 — 5/4 plugin_auto_upgrade + 오늘 DSL fix 등 | ~10분 |
| 5 | 검증 — 우리 endpoint 정상 + 잠복 버그 안 발동 + 다른 dify 기능 영향 X | ~30분 |

**총 ~2.5시간**. 1회 작업.

| 측면 | 평가 |
|---|---|
| 즉시 작업 시간 | 2.5시간 |
| 누적 비용 | 거의 0 (베이스 이미지 옛 코드라 잠복 버그 안 노출) |
| dev 흐름 | 빠름 (부분 mount는 변경 반영) |
| 깔끔함 | 높음 (dify 본체 무수정) |
| 미발견 위험 | 낮음 |

**적합한 시나리오**: 잠복 버그 패턴 2회 이상 재발, Phase 2 안정화 시점, 1회 시간 투자 가능할 때.

### 옵션 C — 우리 코드 박힌 커스텀 이미지 rebuild

```yaml
# docker-compose.yaml 변형
api:
  build:
    context: ../
    dockerfile: api/Dockerfile.spx
  # mount 제거
```

`Dockerfile.spx`에서 base 이미지 `FROM langgenius/dify-api:1.13.3` + 우리 신규 모듈만 COPY.

| 측면 | 평가 |
|---|---|
| 즉시 작업 시간 | ~1.5시간 |
| 누적 비용 | 매 변경 시 rebuild 5~10분 (dev 흐름 부담) |
| 깔끔함 | 높음 (이미지에 우리 코드 박힘) |
| 미발견 위험 | 낮음 (dify 본체 = 베이스 이미지) |
| dev 흐름 | 느림 (rebuild 필요) |

**적합한 시나리오**: 본격 운영 전환 시점 (변경 빈도 낮음). dev 단계엔 부담.

## 결정 매트릭스

| 상황                              | 권장 옵션                             |
| ------------------------------- | --------------------------------- |
| Phase 2 진행 중 (오늘 시점) — 빠른 흐름 우선 | **A**                             |
| Phase 2 완료 후 안정화 — 깔끔함 우선       | **B**                             |
| 운영 전환 / production 배포 시점        | **C**                             |
| 잠복 버그 1건 추가 발견                  | **B 즉시 진행** (5/4 일지 트리거 충족 시점 도래) |

## 현재 결정 (2026-05-07)

**옵션 A 유지**. 근거:
- Phase 2 마지막 마일스톤 (chart-drawer + context-bar) 진행 중
- 오늘 DSL import 사례까지 발견 2건 → 5/4 일지 트리거 *"2번 더 같은 패턴 발견되면 옵션 B 재검토"* 자격 충족 직전
- chart-drawer + context-bar 끝나고 별도 의제로 옵션 B 진지 검토 예정
- 트리거 신호: 추가 1건 발견 시 즉시 옵션 B 진행

## 5/4 일지 원본 결정

> "오늘 결정: A 유지 (실용적). 내일 다시 볼 때 옵션 B 무게가 더 크다면 전환 검토."  
> "2번 더 같은 패턴 발견되면 옵션 B 진지하게 재검토 — 패턴이 너무 흔하면 patch보다 mount 제거가 낫다"

## 메타 학습

### 1. "같은 버전 번호 ≠ 같은 코드"
베이스 이미지 1.13.3과 호스트 신 1.13.3은 같은 버전 라벨이지만 내용 다름. **이미지 build 시점이 핵심**. 호스트 git pull로 최신 dify 받으면 이미지 build 시점 이후 commit 다 들어옴.

### 2. mount는 양날의 검
- 장점: 호스트 변경 즉시 반영 → dev 흐름 빠름 + 우리 신규 모듈 노출 용이
- 단점: dify 본체의 잠복 버그도 같이 노출

### 3. dify 무수정 원칙과의 긴장
옵션 A의 host patch는 dify 본체 파일 수정 → 무수정 원칙 위반. 옵션 B/C가 원칙에 충실하지만 시간 비용. 트레이드오프.

### 4. 트리거 기반 의사결정
*"N건 발견 시 옵션 전환"* 같은 임계값을 미리 정해두면 미루기 함정 회피. 본 사례는 5/4에 *"2회"* 트리거 명시 → 5/7 사례로 임계값 도래 → 다음 단계 진입.

## 관련 노트

- [[4. 지식노트/Dify - PluginAutoUpgradeService DetachedInstanceError 버그 (1.13.3)]] — 1차 사례 자세
- [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]] — mount 패턴 일반론
- [[4. 지식노트/Python - silent fallback과 logger.exception 의무]] — 같은 결의 디버깅 함정
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] — 5/4 + 5/7 변경 이력
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] — H-ENV-02 (api/worker mount 미지정과 같은 가족 — 그건 mount 부재 / 이건 mount 통째)

## 다음 검토 시점

chart-drawer + context-bar 완료 후. 그 시점에 잠복 버그 추가 발견 카운트 + Phase 2 완료 시점 확인 → 옵션 A 누적 vs B 1회 작업 비용 재계산 → 결정.
