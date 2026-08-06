---
tags: [개발, PostgreSQL, SQLAlchemy, Python, 트랜잭션]
date: 2026-05-06
---
# PostgreSQL - InFailedSqlTransaction과 SQLAlchemy fallback 패턴

## 핵심
- **PostgreSQL은 트랜잭션 안에서 한 번 SQL 에러가 나면 rollback 전까지 모든 후속 쿼리를 거부**
- 에러 메시지: `current transaction is aborted, commands ignored until end of transaction block`
- SQLAlchemy `try/except` fallback 패턴 사용 시 **except 첫 줄에 `db.session.rollback()` 필수**
- MySQL/SQLite와 다른 PostgreSQL 고유 동작

## 동작 — 트랜잭션 abort 상태

```sql
BEGIN;
SELECT * FROM nonexistent_table;
-- ERROR: relation "nonexistent_table" does not exist
-- 트랜잭션 = aborted 상태로 전환

SELECT 1;
-- ERROR: current transaction is aborted, commands ignored
-- → 정상 SQL도 거부됨

ROLLBACK;
-- 이제야 회복

SELECT 1;
-- → OK
```

→ **첫 에러 후 모든 쿼리 차단**. 회복 유일 방법 = `ROLLBACK` (또는 `COMMIT`이지만 변경사항 없으니 의미 없음).

## SQLAlchemy fallback 패턴의 함정

### ❌ Bad — rollback 누락

```python
def _get_dept_counts(...):
    try:
        return db.session.query(SpDepartment).filter(...).all()
    except Exception:
        logger.exception("sp_departments fallback")
        return []   # ← 여기까진 OK

def _get_unassigned_counts(...):
    try:
        return db.session.query(SpObjectOwnership).filter(...).all()
    except Exception:
        # ↓ 다른 fallback SQL 시도
        return db.session.query(App.id).count()  # ← 폭발
        # InFailedSqlTransaction: current transaction is aborted

# Service 메서드 호출
def get_dept_objects(...):
    dept_counts = _get_dept_counts(...)         # 첫 쿼리 실패 + 트랜잭션 aborted
    unassigned = _get_unassigned_counts(...)    # 두 번째 쿼리 폭발 → 500
```

### ✅ Good — except 첫 줄에 rollback

```python
def _get_dept_counts(...):
    try:
        return db.session.query(SpDepartment).filter(...).all()
    except Exception:
        db.session.rollback()    # ← 트랜잭션 회복
        logger.exception("sp_departments fallback")
        return []

def _get_unassigned_counts(...):
    try:
        return db.session.query(SpObjectOwnership).filter(...).all()
    except Exception:
        db.session.rollback()    # ← 회복 후 fallback SQL 안전
        logger.exception("sp_object_ownership fallback")
        return db.session.query(App.id).count()
```

## 잠복하기 쉬운 이유

### 1. 단일 쿼리 service에선 발현 안 함

```python
def get_kpi(...):
    try:
        return db.session.query(...).all()
    except Exception:
        return None   # rollback 없어도 한 번만 실패하면 OK
```

→ 후속 쿼리 없으니 잠복. 다음에 같은 세션에서 쿼리 시도할 때 폭발.

### 2. 단위 테스트 mock 환경에선 발견 불가

```python
# 단위 테스트 — db.session을 mock으로 대체
db.session.execute = MagicMock(side_effect=ProgrammingError(...))
result = service.get_dept_objects(...)
assert result == []   # ← 트랜잭션 시뮬레이션 안 되니 통과
```

→ **통합 테스트(실제 PostgreSQL 사용)에서만 잡힘**. 단위 테스트로 안전 보장 안 됨.

### 3. fallback 분기 안에서만 발현

정상 흐름(테이블 존재)에선 except 진입 자체를 안 해서 잠복. **테이블 미생성 환경 / 권한 부족 / 컬럼 변경** 등 예외 상황에서만 노출.

## MySQL / SQLite와의 차이

| DB | 한 쿼리 실패 후 동작 |
|----|---------------------|
| **PostgreSQL** | 트랜잭션 abort, 후속 쿼리 전부 거부 (rollback 필수) |
| MySQL | 다음 쿼리 정상 진행 (트랜잭션 영향 없음) |
| SQLite | 다음 쿼리 정상 진행 |

→ MySQL/SQLite 환경에서 작성한 코드를 PostgreSQL로 마이그레이션하면 잠복 버그 노출 가능. **PostgreSQL 환경 표준 회사**에서 신코드 작성 시 rollback 패턴 의식 필수.

## 표준 패턴 — 회사 컨벤션 권장

```python
def _query_with_fallback(query_func, fallback_func=None):
    """SQLAlchemy fallback 표준 패턴."""
    try:
        return query_func()
    except Exception:
        db.session.rollback()
        logger.exception("query failed, fallback")
        if fallback_func:
            return fallback_func()
        return None
```

또는 데코레이터:

```python
def safe_fallback(default):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except Exception:
                db.session.rollback()
                logger.exception(f"{func.__name__} fallback")
                return default
        return wrapper
    return decorator

@safe_fallback(default=[])
def _get_dept_counts(...):
    return db.session.query(SpDepartment).filter(...).all()
```

## 검증 명령

`docker compose exec api psql ...`로 직접 트랜잭션 abort 시뮬레이션:

```sql
BEGIN;
SELECT * FROM nonexistent_table;
SELECT 1;            -- ← InFailedSqlTransaction 확인
ROLLBACK;
SELECT 1;            -- ← 회복 확인
```

또는 Python:

```python
from sqlalchemy.exc import ProgrammingError, InternalError

# 시뮬레이션
db.session.execute(text("SELECT * FROM nonexistent_table"))
try:
    db.session.execute(text("SELECT 1"))
except InternalError as e:
    print("Transaction aborted:", e)
    db.session.rollback()
db.session.execute(text("SELECT 1"))    # ← 회복 후 OK
```

## 실전 — 발견 사례 (2026-05-06)

SPX-Agent 프로젝트 `dashboard_dept_objects_service.py`:

```
_get_dept_counts() — sp_departments 미존재
    ↓ ProgrammingError, except에서 [] 반환 (rollback 누락)
    ↓ 트랜잭션 aborted 상태 잔존
    
_get_unassigned_counts() — 같은 세션에서 호출
    ↓ 첫 쿼리 또 실패
    ↓ except 안 fallback SQL 시도
    ↓ InFailedSqlTransaction → 500 에러
```

**해결**: 두 메서드 except 첫 줄에 `db.session.rollback()` 추가 → 200 응답 정상화.

**놓친 이유**: 단위 테스트는 mock 사용이라 트랜잭션 시뮬레이션 안 됨. sp_ 테이블 생성 후엔 except 진입 자체를 안 해서 정상 흐름에서 발견 불가. **통합 테스트가 잡혔어야 할 영역**.

## 학습 — 검출 위치 분리

| 결함 유형 | 검출 단계 |
|---------|----------|
| 로직 버그 | 단위 테스트 |
| **트랜잭션 abort** | **통합 테스트 (실제 DB)** |
| 동시성 문제 | 부하 테스트 |
| 시각/UX | 사용자 검증 |

→ 5/4 일지의 "Generator-Evaluator 분리" 원칙 그대로 — 검증 도구는 결함 유형별로 다름. 단위 테스트만으론 트랜잭션 결함 못 잡는다는 점 인지 필수.

## 관련 노트
- [[Pydantic - Python 데이터 검증 라이브러리]]
- [[Python - logger.error vs logger.exception (ruff TRY400)]]
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|defect-catalog]] H-DASH-04
- [[1. Daily/2026-05-06.md]]
