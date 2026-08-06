---
tags: [개발, Python, 로깅, ruff]
date: 2026-05-06
---
# Python - logger.error vs logger.exception (ruff TRY400)

## 핵심
- **except 블록 안에서는 `logger.error()` 대신 `logger.exception()`을 써야 한다**
- `.exception()`은 `.error()` 동작에 **현재 스택 트레이스(traceback)를 자동 첨부**
- ruff `TRY400` 룰이 이 위반을 자동 검출 + `--fix`로 자동 치환 가능

## 차이

```python
import logging
logger = logging.getLogger(__name__)

# ❌ Bad — 메시지만 남고 traceback이 사라짐
try:
    do_something()
except Exception as e:
    logger.error(f"failed: {e}")
    # 로그: ERROR    failed: ConnectionError('timeout')
    # → 어디서 터졌는지 알 수 없음

# ✅ Good — traceback 자동 첨부
try:
    do_something()
except Exception:
    logger.exception("failed")
    # 로그:
    # ERROR    failed
    # Traceback (most recent call last):
    #   File "main.py", line 42, in handler
    #     do_something()
    #   File "service.py", line 17, in do_something
    #     raise ConnectionError('timeout')
    # ConnectionError: timeout
```

## 왜 중요한가 — 운영 환경 디버깅

| 상황 | `.error()` | `.exception()` |
|------|------------|----------------|
| 메시지만 본다 | OK | OK |
| 어디서 터졌는지 안다 | ❌ | ✅ |
| 호출 체인 안다 | ❌ | ✅ |
| 같은 에러 메시지 + 다른 발생 위치 구별 | ❌ | ✅ |

운영 중 알람 보고 **5초 안에 원인 위치 파악**이 가능한지가 갈림. `.error()`만 박힌 코드는 prod에서 디버깅 자체가 안 됨.

## 동등한 표현

`logger.exception("msg")` 는 사실 다음과 동일:

```python
logger.error("msg", exc_info=True)
```

즉 `.exception()` = `.error()` + `exc_info=True` 단축. 그래서 ruff는 except 블록의 `.error()` 호출을 **모두 잡아서** `.exception()`으로 변환 권장.

## except 블록 *밖*에선 어떻게?

except 밖에선 `.error()`가 정상이고 `.exception()`은 의미 없음 (스택 없음). ruff TRY400은 except 블록 안만 검사함.

```python
# except 밖 → .error() OK
if not state or state != session_state:
    logger.error("State mismatch: request=%s session=%s", state, session_state)
    return {"error": "Invalid state"}, 400
```

## 자동 fix

```bash
# 우리 영역만 검사
ruff check --select TRY400 services/admin/

# 자동 수정
ruff check --fix --select TRY400 services/admin/
```

대부분의 경우 **ruff `--fix`로 안전 자동 치환** (`logger.error` → `logger.exception` + `as e` 캡처가 단순한 경우). 복잡한 케이스(에러 객체를 메시지에 포함)는 수동 검토 필요.

## 함정 — 메시지에 `{e}` 포함하던 코드

```python
# Before
except Exception as e:
    logger.error(f"failed: {e}")

# After (ruff fix 후)
except Exception as e:
    logger.exception(f"failed: {e}")
```

→ 동작은 OK지만 **메시지에 `{e}` 중복**이 됨 (traceback에 이미 같은 정보 있음). 깔끔하게 가려면:

```python
except Exception:
    logger.exception("failed")
```

`as e` 자체도 더 이상 필요 없음 (traceback이 자동으로 들어감).

## 실전 — 발견 사례

2026-05-06 백엔드 commit 시 `auth/keycloak.py`에서 다음 패턴 다수 발견:

```python
except Exception:
    logger.error("Failed to exchange authorization code with Keycloak")
except Exception as e:
    logger.error("Keycloak token verification failed: %s", str(e))
```

→ 모두 `.exception()` 변환 대상. 같은 파일 내에 `.exception()`도 일부 사용되고 있어 **혼재 상태** = 누군가 일관성 없이 작성. ruff hook이 활성화된 환경에서만 사후 발견됨 (개발자별 hook 활성화 시점 차이).

## 관련 노트
- [[Python - datetime.UTC alias (Python 3.11+, ruff UP017)]]
- [[husky - .husky 폴더 패턴과 install 시점]]
- [[Git - core.hooksPath와 Hook 위치 추적]]
- [[1. Daily/2026-05-06.md]]
