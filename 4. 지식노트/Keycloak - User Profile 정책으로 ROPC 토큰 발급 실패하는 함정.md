---
tags: [keycloak, 인증, 트러블슈팅, ROPC]
date: 2026-05-28
---

# Keycloak - User Profile 정책으로 ROPC 토큰 발급 실패하는 함정

> Keycloak Admin UI에서 사용자 만들고 ROPC로 토큰 받으려 하면 `invalid_grant + "Account is not fully set up"` 에러. Required Actions가 비어 있어도 발생할 수 있다 — **User Profile 정책의 필수 attribute 미입력**이 원인.

## 증상

ROPC(Resource Owner Password Grant) 토큰 발급 시도:

```powershell
$body = @{
    grant_type = 'password'
    client_id  = 'demo-dify-chat'
    username   = 'demo-01'
    password   = '123'
}
Invoke-RestMethod -Uri "$kc/realms/Spelix/protocol/openid-connect/token" -Method Post -Body $body
```

응답:
```json
{"error": "invalid_grant", "error_description": "Account is not fully set up"}
```

## 원인 3가지 (체크 순서)

### 1순위 — Realm-level Default Required Actions

**경로**: Admin UI → Authentication → Required actions 탭

- "Default action" 컬럼이 ON인 항목은 신규 사용자에 **자동 부여**됨
- 사용자 상세 화면의 "Required user actions"엔 안 보일 수 있음
- 흔한 default action:
  - `Verify Email`
  - `Update Password`
  - `Verify Profile`
  - `Update Profile`

→ Default action 컬럼 OFF로 변경.

### 2순위 — User Profile 필수 attribute 미입력

Keycloak 21+ 의 **User Profile 정책**이 firstName / lastName / email을 required로 두면, 비어있는 사용자는 "fully set up 안 됨"으로 판정.

- 사용자 상세 → Details 탭
- **First name, Last name, Email** 모두 채워야 함 (가짜 값이라도 OK)
- 예: `Demo / User01 / 01@demo.com`

> ROPC는 비대화형이라 부족한 필드를 입력받을 화면이 없음. 표준 로그인 화면이라면 사용자가 채우면 되지만 ROPC는 토큰 발급 거부.

### 3순위 — 사용자 개별 Required Actions

사용자 상세 → Details → **Required user actions** 박스가 비어있는지 확인.

채워져 있으면 하나씩 X 눌러 제거 → Save.

### 부가 확인

| 항목 | 권장 |
|---|---|
| Enabled | ON |
| Email verified | ON |
| Credentials → Password Temporary | OFF |

## 사례 — demo-dify-chat client (2026-05-28)

1. Required Actions 비어있는데도 에러 발생
2. realm Default Required Actions도 모두 OFF였음
3. → User Profile 필수 필드 입력으로 해결 (`Demo / User01 / 01@demo.com`)

이후 토큰 정상 발급.

## 검증 — 에러 본문 확인

PowerShell에서 catch로 응답 본문 추출:

```powershell
try {
    $resp = Invoke-RestMethod -Uri $tokenUrl -Method Post -Body $body -ContentType 'application/x-www-form-urlencoded'
} catch {
    $reader = New-Object System.IO.StreamReader($_.Exception.Response.GetResponseStream())
    $errBody = $reader.ReadToEnd()
    Write-Host "Status:" $_.Exception.Response.StatusCode
    Write-Host "Body:" $errBody
}
```

또는 admin UI의 사용자 → Events 탭에서 실패 이벤트 상세 확인.

## 핵심 교훈

ROPC는 표준 OIDC 로그인과 달리 **사용자 보완 입력 화면이 없음** → 계정이 100% 완성 상태여야 동작. "Required user actions" 박스만 보면 안 되고, realm Default Action + User Profile 정책 + 필수 필드 입력까지 함께 확인 필요.

## 관련 노트

- [[4. 지식노트/Keycloak - 토큰 만료 시 API CPU 100% 행 패턴]]
- [[4. 지식노트/웹 인증 흐름 - JWT, 쿠키, 세션]]
- [[4. 지식노트/인증 아키텍처 용어 - BaaS, IAM, SSO, 미들웨어]]
- 적용 사례: [[3. 프로젝트/web-ui/Keycloak 통합 구현 설계]] PR #1 트러블슈팅
