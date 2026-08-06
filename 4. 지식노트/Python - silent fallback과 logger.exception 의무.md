---
tags: [Python, 개발, CS, 디버깅]
date: 2026-05-07
---
# Python — silent fallback과 logger.exception 의무

## 핵심

`try/except: return []` 같은 fallback 패턴에 `logger.exception` 누락하면 **에러가 발생해도 로그에 안 찍혀 디버깅 불가능**. 화면은 빈 데이터로만 보이고, 코드에서 진짜 원인을 찾는데 grep/SQL/컨테이너 검사로 우회 디버깅 1시간+ 소요.

## 메커니즘

```python
# ❌ Silent fallback — 에러 정보 완전 소실
def get_dept_counts():
    try:
        return db.session.execute(...).fetchall()
    except Exception:
        db.session.rollback()
        return []  # 에러 났어도 빈 리스트만 반환, 로그 0건
```

이 코드가 SQL 실패 / 컬럼명 미스 / Pydantic ValidationError 등 어떤 이유로든 except 진입하면:
- 함수는 빈 리스트 반환
- 호출자는 "데이터 없음"으로 처리
- 화면은 "데이터가 없습니다" 또는 빈 차트 표시
- **로그엔 아무것도 안 찍힘** → 디버깅 시점에 단서 0개

## 표준 패턴

```python
import logging
logger = logging.getLogger(__name__)

def get_dept_counts():
    try:
        return db.session.execute(...).fetchall()
    except Exception:
        db.session.rollback()
        logger.exception("Failed to fetch dept counts — falling back to empty")
        return []
```

`logger.exception`은 traceback 자동 첨부 (= `logger.error` + `exc_info=True`). 진단 시 `docker logs api | grep ERROR` 한 줄로 진짜 원인 즉시 노출.

## 검증 명령

기존 코드에 silent fallback 잔존 검사:

```bash
grep -rln 'except Exception' api/services/ | xargs grep -L 'logger.exception'
```

결과 0건이면 OK. 출력된 파일은 보강 필요.

## 진단 흐름의 메타 학습

"에러가 안 보임" 두 가지 경우 분리:
1. **정말 에러가 없는 상태** — 코드/데이터/SQL 모두 정상
2. **에러는 있는데 로깅이 묻혀있는 상태** — silent fallback 함정

디버깅 첫 5분에 (2)를 점검하면 1시간 헤매기 회피 가능. *"있는 줄 알고 찾는데 없음"* 시나리오의 본질.

## 관련 결함 / 컨벤션

- ruff `TRY400` — `logger.error` → `logger.exception` 자동 fix
- spx-agent `conventions.md § 1` Fallback 패턴 + `logger.exception` 의무 하위 섹션
- 5/7 사례: service 4종 명명 정정 후 화면 빈 데이터 → silent fallback 때문에 1시간+ 디버깅 → conventions에 검증 명령 추가

## 자동화 후보

- pre-commit hook 또는 CI lint에 silent fallback 검출 자동화
- 룰: `except Exception` 블록 안에 `logger.exception` 또는 `raise` 둘 중 하나 의무

## 관련 노트

- [[4. 지식노트/Python - logger.error vs logger.exception (ruff TRY400)]]
- [[4. 지식노트/PostgreSQL - InFailedSqlTransaction과 SQLAlchemy fallback 패턴]]
