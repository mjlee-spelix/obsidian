---
tags: [지식, 개발, CS, SQL]
date: 2026-03-26
---
# SQL - % 와일드카드 이스케이프

## 핵심
- Python SQL 드라이버(psycopg2 등)에서 `%`를 변수 바인딩 기호로 오인해 `TypeError` 발생
- 쿼리 내 `%`는 `%%`로 이스케이프 처리해야 함

## 상세

### 문제 상황
```
TypeError: dict is not a sequence
```
SQL 쿼리에 `ILIKE '%Bikes%'` 같은 와일드카드가 포함될 때 Python SQL 드라이버가 `%`를 변수 바인딩 기호(`%s`)로 오인해 충돌 발생.

### 해결 방법
쿼리 내 모든 `%`를 `%%`로 이스케이프 처리

```sql
-- 잘못된 예
WHERE category ILIKE '%Bikes%'

-- 올바른 예
WHERE category ILIKE '%%Bikes%%'
```

### 적용 대상
- `psycopg2` (PostgreSQL)
- `pymysql` (MySQL)
- Python의 `cursor.execute()` 방식으로 쿼리 직접 전달 시 모두 해당

### Dify 활용 시
- 프롬프트에 "와일드카드 사용 시 `%%` 형태로 이스케이프하라"는 지침 명시

## 관련 노트
- [[Dify Agent - LLM 카테고리명 환각 대응]]
