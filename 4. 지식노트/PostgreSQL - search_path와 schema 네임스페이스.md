---
tags: [지식, PostgreSQL, SQL, CS, 데이터베이스]
date: 2026-04-07
---
# PostgreSQL - search_path와 schema 네임스페이스

## 핵심
- PostgreSQL은 테이블을 **schema**라는 네임스페이스로 묶어 관리. 같은 이름의 테이블이 여러 스키마에 존재 가능
- `search_path`는 "스키마 이름 안 적었을 때 어느 스키마를 순서대로 찾을지"를 정하는 세션 변수
- 기본값 `"$user", public` → public 스키마가 아닌 곳의 테이블은 prefix 필수, 안 그러면 `relation does not exist` 에러
- 해결: `ALTER ROLE` / `ALTER DATABASE`로 사용자/DB에 search_path를 박아두면 prefix 생략 가능

## 상세

### Schema가 뭔가
PostgreSQL의 schema = 테이블/뷰/함수 등을 묶는 **네임스페이스**. 디렉터리와 비슷.

```
mydb
├─ public          ← 기본 스키마
│  ├─ users
│  └─ orders
├─ agent_demo      ← 사용자 정의 스키마
│  ├─ analytics_sales_fact
│  └─ analytics_monthly_trend
└─ hr
   └─ employees
```

같은 이름의 테이블이 다른 스키마에 공존 가능:
- `public.users`
- `hr.users`
- `agent_demo.users`

테이블 식별 풀네임: `database.schema.table` 또는 `schema.table`

### search_path의 역할

쿼리에서 `SELECT * FROM users`라고 schema 이름 없이 쓰면 PostgreSQL은 어떻게 찾을까?
→ `search_path` 변수에 적힌 스키마들을 **순서대로** 뒤져서 첫 번째로 매치되는 테이블 사용.

```sql
SHOW search_path;
-- 결과: "$user", public
```

기본값 해석:
1. `"$user"` — 현재 접속 사용자명과 같은 스키마 (예: 사용자가 `dify_user`면 `dify_user` 스키마)
2. `public` — 기본 공용 스키마

이 두 군데에 없으면 `ERROR: relation "users" does not exist`

### 자주 발생하는 함정 — Schema Prefix 누락

DB는 `agent_demo` 스키마에 테이블 만들어놨는데, 쿼리에서 `agent_demo.` prefix를 빼먹으면 search_path에 `agent_demo`가 없어서 못 찾음:

```sql
SELECT COUNT(*) FROM analytics_sales_fact;
-- ERROR: relation "analytics_sales_fact" does not exist

SELECT COUNT(*) FROM agent_demo.analytics_sales_fact;
-- 정상 동작
```

LLM 기반 NL2SQL에서 자주 발생하는 패턴 — instruction에 "prefix 붙여라" 적어놔도 짧은 쿼리(`COUNT(*)`, `LIMIT 1`)에서 LLM이 빼먹는 경향.

### search_path 설정 레벨

| 레벨 | 명령어 | 효과 범위 | 적용 시점 |
|---|---|---|---|
| 세션 | `SET search_path TO agent_demo, public;` | 현재 connection만 | 즉시 |
| **사용자(role)** | `ALTER ROLE myuser SET search_path TO agent_demo, public;` | 그 사용자가 새로 접속할 때마다 | **다음 connection부터** |
| 데이터베이스 | `ALTER DATABASE mydb SET search_path TO agent_demo, public;` | 그 DB에 접속하는 모든 사용자 | 다음 connection부터 |
| Connection string | `?options=-c%20search_path=agent_demo,public` (psycopg2 등) | connection별 | 즉시 |
| 전역 | `postgresql.conf`의 `search_path` | 서버 전체 | 재시작/reload |

#### `ALTER ROLE` (가장 흔히 쓰는 방법)

```sql
ALTER ROLE dify_user SET search_path TO agent_demo, public;
```

- 그 role로 접속하는 모든 새 connection에 자동 적용
- 한 번만 설정하면 됨
- 검증: `SHOW search_path;` 결과가 `agent_demo, public`인지 확인

#### `ALTER DATABASE`

```sql
ALTER DATABASE mydb SET search_path TO agent_demo, public;
```

- 그 DB에 접속하는 **모든 사용자**에게 적용 (role 설정보다 약한 우선순위)
- 다중 사용자 환경에서 한 번에 적용하기 좋음

#### Connection string

DB user/DB 권한 변경이 어려울 때 클라이언트 측에서 connection 옵션으로 전달:

```python
# psycopg2
conn = psycopg2.connect(
    host="...", dbname="...", user="...", password="...",
    options="-c search_path=agent_demo,public"
)
```

### 우선순위
같은 search_path가 여러 레벨에 설정되면 우선순위:

```
세션 SET > Connection string options > ALTER ROLE > ALTER DATABASE > postgresql.conf
```

### 주의사항

1. **`ALTER ROLE`/`ALTER DATABASE`는 기존 connection에 즉시 적용 안 됨**. 새 connection부터 효과. 클라이언트 connection pool 쓰면 pool 비우거나 재접속 필요
2. **공유 DB user의 위험성**: 여러 어플리케이션/워크플로우가 같은 DB user를 쓰면 한쪽에 search_path 박아두는 게 다른 쪽에 영향을 줄 수 있음. 격리가 필요하면 별도 user 만들거나 connection string 옵션 사용
3. **search_path와 보안**: search_path 첫 번째에 신뢰할 수 없는 스키마를 두면 schema injection 공격 가능. function 정의 시 `SET search_path = ...` 명시가 권장됨 ([CVE-2018-1058](https://www.postgresql.org/support/security/CVE-2018-1058/))
4. **`SHOW search_path`로 항상 검증**. 환경마다 다를 수 있음

### 실무 패턴

| 상황 | 권장 방법 |
|---|---|
| 단일 앱 전용 DB user | `ALTER ROLE` |
| 다중 앱 공유 DB user | Connection string options |
| 모든 사용자 대상 DB 단위 설정 | `ALTER DATABASE` |
| 임시/디버깅 | 세션 `SET` |
| 함수/트리거 내부 | `CREATE FUNCTION ... SET search_path = ...` (보안) |

### 핵심 교훈

- **schema는 디렉터리, search_path는 PATH 환경변수**라고 생각하면 직관적
- LLM에게 schema prefix를 강제하는 것보다 **인프라 레벨에 search_path를 박아두는 게 안정적**
- 단, 공유 DB user 환경에서는 사이드이펙트를 검토해야 함. 격리가 필요하면 connection string 옵션이나 별도 user
- prefix 누락은 production에서 흔한 함정. dev/staging/prod 환경 간 search_path 차이로도 같은 에러 재현 가능

## 관련 노트
- [[SQL - % 와일드카드 이스케이프]]
- [[Dify - db_client_node UNION 쿼리 Invalid operation type 에러]]
- [[LLM - JSON Over-escape 버그와 복구 패턴]]

## 참고 자료
- [PostgreSQL Docs - Schema Search Path](https://www.postgresql.org/docs/current/ddl-schemas.html#DDL-SCHEMAS-PATH)
- [PostgreSQL Docs - ALTER ROLE](https://www.postgresql.org/docs/current/sql-alterrole.html)
- [CVE-2018-1058 - search_path 보안 취약점](https://www.postgresql.org/support/security/CVE-2018-1058/)
