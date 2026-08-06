---
tags: [JWT, 인증, 운영, 디버깅]
date: 2026-05-28
---

# JWT - 토큰 lifespan 정합 확인 (exp - iat)

> JWT 발급 시 IdP의 토큰 lifespan 설정 ↔ 앱 측 cookie max-age / 환경변수 설정이 어긋나면 세션 유실 또는 401 폭증. 매번 토큰 받자마자 `exp - iat`로 IdP 실측값 확인이 디버깅 기본.

## 왜 중요한가

| 불일치 방향 | 결과 |
|---|---|
| cookie max-age **<** 토큰 lifespan | cookie가 먼저 만료 → 유효한 토큰인데 브라우저가 안 보냄 (세션 유실) |
| cookie max-age **>** 토큰 lifespan | cookie는 보내지만 토큰이 무효 → 401 응답 (refresh 동작 안 하면 강제 로그아웃) |
| 환경변수 `EXPIRE` **≠** IdP 설정 | 문서/코드는 60분이라 박혀있는데 실제는 다른 값 → 운영 오해 |

## 검증 — `exp - iat` 한 줄

JWT payload의 두 claim:
- `iat` (issued at): 발급 시각 (Unix epoch)
- `exp` (expires): 만료 시각 (Unix epoch)

차이 = lifespan 초 단위.

### PowerShell

```powershell
# 토큰 받은 후 ($resp.access_token)
$payload = $resp.access_token.Split('.')[1].Replace('-', '+').Replace('_', '/')
$pad = (4 - ($payload.Length % 4)) % 4
$payload = $payload + ('=' * $pad)
$claims = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($payload)) | ConvertFrom-Json

Write-Host "Lifespan:" ($claims.exp - $claims.iat) "초"
```

### Bash + jq

```bash
echo "$ACCESS_TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq '{exp, iat, lifespan: (.exp - .iat)}'
```

### Python

```python
import jwt, json
decoded = jwt.decode(token, options={"verify_signature": False})
print(f"Lifespan: {decoded['exp'] - decoded['iat']}초")
```

## 사례 — Keycloak Spelix realm (2026-05-28)

PR #1 검증 시 토큰 받고 두 가지 확인:

| 토큰 | `exp - iat` | 설계 값 | 정합? |
|---|---|---|---|
| access_token | **3600초** (60분) | `ACCESS_TOKEN_EXPIRE_MINUTES=60` | ✅ |
| refresh_token | **7200초** (2시간) | `REFRESH_TOKEN_EXPIRE_DAYS=30` (=2,592,000초) | ❌ **큰 차이** |

→ refresh도 30일로 알고 있었으나 실제는 2시간. 코드에서 cookie max-age=30일로 박아도 토큰은 2시간 후 무효 → 강제 로그아웃 발생.

대응:
- Keycloak realm settings → Sessions의 SSO Session Idle/Max 늘리기 (realm 전체 영향)
- 또는 client별 Advanced 탭에서 override (해당 client만 영향)

## 운영에서 확인할 lifespan 4종

| 종류 | Keycloak 위치 | 영향 |
|---|---|---|
| Access Token Lifespan | Realm → Tokens 탭 | 매 API 호출 |
| SSO Session Idle | Realm → Sessions 탭 | refresh 가능 기간 (사용 시 갱신) |
| SSO Session Max | Realm → Sessions 탭 | refresh 가능 절대 한도 |
| Client Session Idle/Max | Client → Advanced 탭 | client-level override |

토큰 종류별로 어떤 설정이 영향 주는지:
- access의 `exp - iat` = Access Token Lifespan
- refresh의 `exp - iat` = SSO Session Idle (또는 Client override)

## 점검 자동화 (선택)

배포 후 헬스체크에 토큰 lifespan 확인 추가:

```bash
# CI/배포 후 smoke test
curl ... /token | jq '.access_token' | base64 decode | jq '.exp - .iat'
# 기대값과 비교, 다르면 alert
```

## 핵심 교훈

IdP 설정값을 코드/문서로만 믿지 말 것. **실제 토큰 받아서 디코드하는 게 진실**. 5분이면 끝나는 검증이 운영 사고 예방.

## 관련 노트

- [[4. 지식노트/웹 인증 흐름 - JWT, 쿠키, 세션]]
- [[4. 지식노트/웹 인증 - authFetch 래퍼 패턴 (401 intercept + auto refresh)]]
- [[4. 지식노트/Keycloak - 토큰 만료 시 API CPU 100% 행 패턴]]
- 적용 사례: [[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]] — 5/28 refresh lifespan mismatch 발견
