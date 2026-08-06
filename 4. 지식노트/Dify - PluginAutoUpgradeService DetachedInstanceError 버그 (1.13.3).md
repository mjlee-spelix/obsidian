---
tags: [dify, 개발, SQLAlchemy, 백엔드, 결함]
date: 2026-05-04
status: 보류 (옵션 C — 무시)
related_files: ["api/services/plugin/plugin_auto_upgrade_service.py"]
---
# Dify - PluginAutoUpgradeService DetachedInstanceError 버그 (1.13.3)

## 핵심

> Dify 1.13.3의 `services/plugin/plugin_auto_upgrade_service.py:get_strategy()`가 SQLAlchemy 안티패턴(`with sessionmaker.begin() as session: return obj`)을 사용해, 호출자가 반환된 객체의 속성을 읽으려 하면 `DetachedInstanceError` 폭발.
>
> **상황 조건**: `tenant_plugin_auto_upgrade_strategies` 테이블에 row가 있을 때만 발동.
> **현재 결정**: 옵션 C(무시) — 우리 대시보드 동작에 영향 없음. Dify upstream 수정 대기.

## 발견 상황

- **시점**: 2026-05-04, KPI 백엔드 fetch 연동 완료 후 검증 중
- **증상**: 대시보드 페이지 우상단에 빨간 "Internal Server Error" 토스트, F12 Network 탭에 `/console/api/workspaces/current/plugin/preferences/fetch` 500 에러 9개
- **사용자 영향**: 토스트 거슬림. 단 **대시보드 자체 동작은 정상** (KPI 4종 + 차트 + 테이블 모두 200 OK).

## 문제 (Stack trace 핵심)

```
File "/app/api/controllers/console/workspace/plugin.py", line 792, in get
    "strategy_setting": auto_upgrade.strategy_setting,
File "/app/api/.venv/.../sqlalchemy/orm/state.py", line 828, in _load_expired
    self.manager.expired_attribute_loader(self, toload, passive)
File "/app/api/.venv/.../sqlalchemy/orm/loading.py", line 1607, in load_scalar_attributes
    raise orm_exc.DetachedInstanceError(
sqlalchemy.orm.exc.DetachedInstanceError: Instance <TenantPluginAutoUpgradeStrategy> is not bound to a Session
```

## 원인 — SQLAlchemy 안티패턴

### 문제 코드 (`services/plugin/plugin_auto_upgrade_service.py:9-15`)

```python
@staticmethod
def get_strategy(tenant_id: str) -> TenantPluginAutoUpgradeStrategy | None:
    with sessionmaker(bind=db.engine).begin() as session:
        return (
            session.query(TenantPluginAutoUpgradeStrategy)
            .where(TenantPluginAutoUpgradeStrategy.tenant_id == tenant_id)
            .first()
        )
```

### 폭발 메커니즘

1. `with sessionmaker.begin() as session:` 블록이 새 세션 + 트랜잭션 시작
2. `session.query(...).first()`로 ORM 객체 fetch
3. **블록을 빠져나갈 때 `begin()`이 자동 commit + 세션 close**
4. SQLAlchemy의 자동 expire 동작으로 **commit 시점에 모든 객체 속성을 "stale"로 표시**
5. 반환된 객체는 **detached** 상태 (어떤 세션에도 묶여있지 않음)
6. 호출자가 `obj.strategy_setting` 접근 → 속성이 stale이라 DB refresh 시도 → 세션 없음 → `DetachedInstanceError`

### 호출자 (`controllers/console/workspace/plugin.py:781-797`)

```python
auto_upgrade = PluginAutoUpgradeService.get_strategy(tenant_id)
# auto_upgrade는 detached 객체

if auto_upgrade:                        # row 있을 때만 분기 진입
    auto_upgrade_dict = {
        "strategy_setting": auto_upgrade.strategy_setting,   # ← 폭발 (line 792)
        "upgrade_time_of_day": auto_upgrade.upgrade_time_of_day,
        "upgrade_mode": auto_upgrade.upgrade_mode,
        "exclude_plugins": auto_upgrade.exclude_plugins,
        "include_plugins": auto_upgrade.include_plugins,
    }
```

## 이전엔 왜 없었나

이 SPX-Agent 환경은 그동안 **Dify 호스트 신버전(1.13.3) 코드를 컨테이너에 부분 mount만**해왔음. `plugin_auto_upgrade_service.py`는 mount 등록 안 됨 → 컨테이너는 base 이미지(`langgenius/dify-api:1.13.3`)의 옛날 코드 사용.

가설: base 이미지의 옛날 코드는 동일 함수가 다른 패턴(예: `db.session` 사용)이었을 가능성. 또는 호출 경로 자체가 달랐거나.

→ **컨테이너에서 1.13.3 신버전 `plugin_auto_upgrade_service.py` 코드가 실행된 적 없음** → 버그 발동 안 함.

## 지금 왜 발생

[[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]]에서 다룬 그 작업의 부수 효과:

1. 2026-05-04, mock 컨트롤러 검증 위해 `docker-compose.override.yaml`에 **호스트 `api/` 폴더 통째 mount** 추가
2. 그 결과 컨테이너가 호스트의 1.13.3 신버전 `plugin_auto_upgrade_service.py` 사용 시작
3. **버그 코드가 처음 실행** → 사용자 DB에 strategy row가 있던 상태라 발동
4. 모든 페이지 로드 시 호출되는 endpoint라 토스트 9번 뜸

→ **환경 셋업 변경의 부수 효과로 잠복 버그가 노출됨** (코드 자체 변경 없음).

## 해결 방법 (3가지 옵션)

### 옵션 A — Host patch (5분, 깔끔)

`get_strategy()`를 Flask request 세션 사용으로 변경:

```python
@staticmethod
def get_strategy(tenant_id: str) -> TenantPluginAutoUpgradeStrategy | None:
    return (
        db.session.query(TenantPluginAutoUpgradeStrategy)
        .where(TenantPluginAutoUpgradeStrategy.tenant_id == tenant_id)
        .first()
    )
```

원리: `db.session`은 Flask가 HTTP request 시작 ~ 응답 끝까지 유지. 객체가 attached 상태 유지 → 호출자가 속성 접근 OK.

장점: 즉시 해소, 5분.
단점: Dify upstream 코드 수정 (추적 필요), 향후 Dify 업그레이드 시 충돌 가능성.

### 옵션 B — DB row 삭제 (30초, 임시)

```sql
DELETE FROM tenant_plugin_auto_upgrade_strategies;
```

원리: row 0개면 `get_strategy()`가 None 반환 → if 분기 안 들어감 → 버그 회피.

장점: 코드 무수정.
단점: 누군가 strategy를 다시 설정하면 재발. 본질 해결 아님.

### 옵션 C — 무시 (현재 결정)

장점:
- Dify upstream 무수정 원칙 충실
- 우리 대시보드 동작에 영향 0
- 다음 Dify 업그레이드 시 자동 해결 가능성 높음 (업스트림 PR 또는 자연 수정)

단점:
- Internal Server Error 토스트가 페이지마다 떠서 거슬림
- 콘솔 로그에 빨간 에러 메시지

### 옵션 D — Upstream PR (정석, 시간 걸림)

Dify GitHub에 issue + PR 제출. 가장 깔끔한 해결이지만 머지·릴리스까지 대기 필요. 우리는 그동안 옵션 C로 버팀.

## 보류 이유 (왜 옵션 C 선택)

1. **우리 작업과 무관**: 대시보드 fetch 5종 모두 200 OK. 사용자 검증한 KPI 값(42, 156, 9.8K, 1.2%)도 정상 표시.
2. **Dify upstream 무수정 원칙**: 사용자 우려대로 한 번 수정하면 나중에 Dify 업그레이드 시 merge 충돌 가능. 이미 사용자가 광범위하게 patch한 다른 파일들도 있어서 patch 면적이 늘어나는 게 부담.
3. **Dify 자체 fix 가능성**: 1.13.3은 비교적 최신 버전이고 SQLAlchemy 안티패턴은 명백한 결함이라 upstream에서 fix할 가능성 높음. 다음 마이너 버전에서 해결 기대.
4. **임시 회피 가능**: 진짜 거슬리면 옵션 B(row 삭제)로 30초에 회피 가능. 본질 해결 아니지만 일시 해소엔 충분.
5. **시간 비용 vs 가치**: 토스트 1개 거슬림 ↔ patch 추적 부담. 후자가 더 큼.

## 모니터링 포인트

- Dify 새 버전(1.13.4+ 또는 1.14) 릴리스 시 `services/plugin/plugin_auto_upgrade_service.py:get_strategy` 변경 여부 확인
- upstream에서 fix되면 우리도 자동 해소 (호스트 코드 git pull로 가져옴)
- 만약 upstream이 오래도록 fix 안 하면 옵션 A 재검토

## 임시 회피 — 거슬릴 때

옵션 B 즉시 적용 (psql 또는 컨테이너 진입):
```bash
docker compose exec db_postgres psql -U postgres -d dify -c "DELETE FROM tenant_plugin_auto_upgrade_strategies;"
```

토스트 사라짐. 단 누군가 플러그인 자동 업그레이드 설정 다시 만들면 재발.

## 관련 노트

- [[4. 지식노트/Docker - 호스트 코드를 컨테이너에 반영하는 패턴]] (이 버그가 노출된 환경 셋업)
- [[3. 프로젝트/spx-agent/SESSION_HISTORY.md]] (2026-05-04 항목)
- SQLAlchemy 공식 문서: https://sqlalche.me/e/20/bhk3 (DetachedInstanceError 일반론)
