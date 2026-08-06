---
tags: [개발, Python, datetime, ruff]
date: 2026-05-06
---
# Python - datetime.UTC alias (Python 3.11+, ruff UP017)

## 핵심
- **Python 3.11+** 부터 `datetime.UTC` 단축 alias 추가됨
- 기존 `datetime.timezone.utc`와 **완전 동일 객체** (`datetime.UTC is timezone.utc` → `True`)
- ruff `UP017` 룰이 구식 표현을 자동 검출 + `--fix`로 단축

## 차이

```python
# ❌ 구식 (Python 3.10 이하 호환, 3.11+에서도 작동은 함)
from datetime import datetime, timezone
now = datetime.now(timezone.utc)

# ✅ 신식 (Python 3.11+ 전용, 더 간결)
from datetime import datetime, UTC
now = datetime.now(UTC)
```

## 왜 추가됐나

Python 표준 라이브러리에서 UTC 다루는 코드가 매우 흔한데 매번 `timezone.utc` 두 단어 적는 게 장황하다는 피드백. 3.11에서 단축 alias 도입.

> "The most commonly used reference for `timezone.utc` is now also accessible as `datetime.UTC`." — Python 3.11 release notes

## 검증 — 정말 같은 객체인가?

```python
>>> from datetime import timezone, UTC
>>> UTC is timezone.utc
True
>>> id(UTC) == id(timezone.utc)
True
```

같은 객체를 가리키는 두 이름. 동작 차이 없음, **완전한 단축**.

## 호환성 주의

| Python 버전 | `timezone.utc` | `UTC` |
|------------|----------------|-------|
| 3.10 이하 | OK | ❌ ImportError |
| 3.11 이상 | OK | ✅ |

→ **3.11+ 의존이 확정된 프로젝트**에서만 `UTC` 사용. 라이브러리 코드라면 호환성 위해 `timezone.utc` 유지가 안전할 수도.

## ruff 자동 fix

```bash
# UP017만 검사
ruff check --select UP017 .

# 자동 수정
ruff check --fix --select UP017 .
```

- import 경로도 자동 갱신 (`from datetime import timezone` → `from datetime import UTC`)
- 사용처 모두 치환

## 실전 — 발견 사례

2026-05-06 SPX-Agent 백엔드 작업 중 본인 코드에서 ruff가 잡음:

```python
# Before
from datetime import datetime, timezone
created_at = datetime.now(timezone.utc)

# After (UP017 자동 fix)
from datetime import datetime, UTC
created_at = datetime.now(UTC)
```

Dify 프로젝트가 Python 3.11+ 요구하므로 `UTC` 사용이 표준. **신규 코드는 `UTC` 직접 import이 더 깔끔**.

## 같이 자주 만나는 ruff 룰

| 룰 ID | 의미 | 자동 fix |
|-------|------|---------|
| UP017 | `timezone.utc` → `UTC` | ✅ |
| TRY400 | except 블록 `logger.error` → `logger.exception` | ✅ |
| COM812 | 함수 인자/dict 마지막 trailing comma 강제 | ✅ |
| E501 | 줄 길이 초과 (보통 120자) | ❌ 수동 |

`UP` prefix는 **pyupgrade** 룰셋. Python 신버전 문법으로 자동 현대화.

## 다른 datetime 단축들도 있나?

3.11에서 추가된 datetime 관련 신기능:
- `datetime.UTC` (이 노트)
- `datetime.fromisoformat()` 확장 (Z 접미사 지원 등)
- `datetime.timestamp()` 정밀도 개선

→ 3.11+ 의존이면 `fromisoformat()`도 더 적극 활용 가능. 별도 라이브러리(`dateutil`) 의존 줄임.

## 관련 노트
- [[Python - logger.error vs logger.exception (ruff TRY400)]]
- [[Pydantic - Python 데이터 검증 라이브러리]]
- [[1. Daily/2026-05-06.md]]
