---
tags: [지식, dify, SQL, PostgreSQL, 트러블슈팅, NL2SQL]
date: 2026-04-07
---
# Dify - db_client_node UNION 쿼리 Invalid operation type 에러

## 핵심
- Dify 플러그인 `spance/db_client_node`(PostgreSQL Client)는 **최상위 `UNION`/`UNION ALL`/`INTERSECT`/`EXCEPT`를 지원하지 않음**
- 원인: 플러그인이 `sqlglot`으로 SQL을 파싱한 뒤 **AST의 루트 노드 클래스**로만 DML 타입을 판별. `expressions.Select`만 SELECT로 인정하고, `expressions.Union`은 `UNKNOWN`으로 떨어져 `ValueError("Invalid operation type")` 발생
- 해결: 집합 연산을 **서브쿼리로 감싸서** 루트를 `Select`로 만든다 (NL2SQL 프롬프트 룰로 강제)

## 상세

### 증상

NL2SQL Agent가 다음 같은 SQL을 생성하면 플러그인에서 에러:

```sql
-- 케이스 1: UNION ALL
SELECT 'Total' AS period, SUM(monthly_revenue) AS revenue
FROM agent_demo.analytics_monthly_trend WHERE order_year=2025
UNION ALL
SELECT (order_month||'월') AS period, monthly_revenue
FROM agent_demo.analytics_monthly_trend WHERE order_year=2025
ORDER BY period

-- 케이스 2: WITH(CTE) + UNION ALL
WITH yearly AS (SELECT SUM(monthly_revenue) AS revenue FROM ...),
     months AS (SELECT CONCAT(order_month,'월') AS period, ... FROM ...)
SELECT 'Total' AS period, yearly.revenue FROM yearly
UNION ALL
SELECT period, monthly_revenue FROM months
ORDER BY period
```

에러 메시지:
```
PluginInvokeError: An error occurred in the spance/db_client_node/db_client_node,
error type: RuntimeError,
error details: <class 'ValueError'>: Invalid operation type
```

반면 단순 SELECT는 정상:
```sql
SELECT order_year, order_month, monthly_revenue
FROM agent_demo.analytics_monthly_trend
WHERE order_year=2025 ORDER BY order_month
```

### 원인 — 플러그인 소스 분석

[spance/db-client-node](https://github.com/spance/db-client-node)의 `tools/api.py`:

```python
from sqlglot import expressions

class SQLType(Enum):
    UNKNOWN = 0
    SELECT = 1
    DELETE = 2
    INSERT = 3
    UPDATE = 4

def typeOf(obj) -> SQLType:
    if isinstance(obj, expressions.Select):
        return SQLType.SELECT
    elif isinstance(obj, expressions.Insert):
        return SQLType.INSERT
    elif isinstance(obj, expressions.Delete):
        return SQLType.DELETE
    elif isinstance(obj, expressions.Update):
        return SQLType.UPDATE
    else:
        return SQLType.UNKNOWN
```

`tools/pg_node.py`의 `_invoke()`:

```python
sql_type, sql_exp = self._check_query(query, parameters)
...
match sql_type:
    case SQLType.SELECT:
        cursor.execute(sql_exp, parameters)
        ...
    case SQLType.INSERT | SQLType.UPDATE | SQLType.DELETE:
        ...
    case _:
        raise ValueError("Invalid operation type")  # ← 여기!
```

**문제 지점**:
- `sqlglot.parse_one("SELECT a UNION ALL SELECT b")`는 루트가 `expressions.Union`(좌/우에 Select 두 개)
- `sqlglot.parse_one("WITH x AS (...) SELECT ... UNION ALL SELECT ...")`도 WITH가 Union 전체를 감싸므로 루트가 여전히 `Union`
- `typeOf()`는 `Union` 처리가 없어서 `UNKNOWN` 반환
- `match`문이 default `case _:` 로 떨어져 → `Invalid operation type`

같은 이유로 `INTERSECT`, `EXCEPT`도 깨질 거임 (각각 `expressions.Intersect`, `expressions.Except`).

단순 CTE만(`WITH x AS (SELECT...) SELECT * FROM x`) 사용하는 건 루트가 `Select`라서 통과한다.

### 해결 방법

#### 1. 서브쿼리 래핑 (즉시, 권장)

집합 연산을 서브쿼리로 감싸면 루트가 `Select`가 되어 통과:

```sql
SELECT * FROM (
    SELECT 'Total' AS period, SUM(monthly_revenue) AS revenue
    FROM agent_demo.analytics_monthly_trend WHERE order_year=2025
    UNION ALL
    SELECT (order_month||'월') AS period, monthly_revenue
    FROM agent_demo.analytics_monthly_trend WHERE order_year=2025
) t
ORDER BY period
```

NL2SQL Agent instruction에 다음 룰을 박아두면 LLM이 자동으로 따른다:

> **금지**: 최상위 `UNION`/`UNION ALL`/`INTERSECT`/`EXCEPT` 사용 금지.
> 집합 연산이 필요하면 반드시 서브쿼리(`SELECT * FROM (... UNION ALL ...) t`)로 감쌀 것.

#### 2. 플러그인 패치 (근본 해결)

`tools/api.py`의 `typeOf()`에 Union 계열을 SELECT로 분류:

```python
def typeOf(obj) -> SQLType:
    if isinstance(obj, (expressions.Select, expressions.Union,
                        expressions.Intersect, expressions.Except)):
        return SQLType.SELECT
    elif isinstance(obj, expressions.Insert):
        return SQLType.INSERT
    ...
```

포크해서 직접 빌드하거나 spance에 PR. 한 줄 수정이라 비용 작음.

### 핵심 교훈

- **AST 루트 타입만으로 DML 분류는 위험**: SQL 표준 set operation(`UNION`/`INTERSECT`/`EXCEPT`)은 sqlglot에서 별도 expression class를 가짐. SELECT로 분류하려면 명시적으로 추가해야 함
- **Dify 플러그인 디버깅 패턴**: 에러 메시지가 `An error occurred in the spance/db_client_node...` 형태로 모호하면 → marketplace 플러그인 GitHub 가서 `tools/` 폴더 소스 까보면 거의 다 보임
- NL2SQL 시스템에서 LLM이 생성할 수 있는 SQL 패턴을 **사전 제약**으로 좁히는 게 fallback 처리보다 안정적. instruction에 "금지 패턴" 명시하는 게 cheap & effective

## 관련 노트
- [[LLM - JSON Over-escape 버그와 복구 패턴]]
- [[Dify Agent - gpt-oss vLLM Function Calling 트러블슈팅]]
- [[Dify - DSL YAML 작성 및 관리]]
- [[SQL - % 와일드카드 이스케이프]]

## 참고 자료
- [spance/db-client-node GitHub](https://github.com/spance/db-client-node)
- [tools/api.py (typeOf 함수)](https://github.com/spance/db-client-node/blob/main/tools/api.py)
- [tools/pg_node.py (_invoke의 match문)](https://github.com/spance/db-client-node/blob/main/tools/pg_node.py)
- [sqlglot expressions 문서](https://sqlglot.com/sqlglot/expressions.html)
