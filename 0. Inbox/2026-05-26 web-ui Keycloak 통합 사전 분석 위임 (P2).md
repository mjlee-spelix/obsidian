---
tags: [Inbox, 위임, keycloak, web-ui, P2]
date: 2026-05-26
status: 위임 준비 완료 (옵시디언 Claudian에서 실행) — 승랑님 PR 번호만 옵션
environment: 옵시디언 Claudian (vault root에서 실행)
---

# 위임 프롬프트 — web-ui Keycloak 통합 사전 분석 (P2)

> 작성일: 2026-05-26
> 작업 위치: web-ui 프로젝트(`C:\Users\Administrator\Projects\n8n\poc\web-ui\dify-chat`) + spx-agent(`C:\Users\Administrator\Projects\spx-agent\`)
> 위임 범위: **조사만 — 코드 수정 0건**. 산출물 = 통합 전략 결정 + 다음 액션 리스트
> 위임 환경: **옵시디언 Claudian** (vault root에서 실행 — 외부 컨텍스트 폴더로 web-ui / spx-agent 절대경로 접근)
> 선행: ✅ **후속 ② 완료** — `C:\Users\Administrator\Projects\spx-agent\.claude\docs\references\keycloak-sync.md` (133 lines, § 1~§ 8) 박혀 있음. SESSION_HISTORY § 2026-05-26 § docs/references/keycloak-sync.md 신설 참조
> 출처: 5/22 협의 18항 ([[1. Daily/2026-05-22.md#web-ui Keycloak 연결 사전 분석 18항 (오늘 협의)]])

---

## 목표

데모 사이트 `web-ui` (옛날 만든 LLM 데모 사이트, 로그인 화면 UI만 있고 동작 stub)에 **spx-agent의 Spelix realm + 194 Keycloak 인프라 재활용해서 실제 로그인 동작 붙이기**.

**본 위임은 조사만**. 통합 전략 결정 + 다음 액션 PR 순서까지만 산출. 실제 통합 PR은 본 위임 결과 받은 후 별도 트랙.

---

## 인풋

| 출처 | 경로/위치 | 상태 |
|---|---|---|
| **web-ui 코드** | `C:\Users\Administrator\Projects\n8n\poc\web-ui\dify-chat` | ✅ 박힘 |
| spx-agent 코드 | `C:\Users\Administrator\Projects\spx-agent\` | ✅ 박힘 |
| **Keycloak 동기화 base (필독)** | `C:\Users\Administrator\Projects\spx-agent\.claude\docs\references\keycloak-sync.md` — § 1~§ 8 박힌 완성본. 특히 § 2 인스턴스 분리 / § 3 테이블 4종 / § 4 매칭 우선순위(UUID → code → 신규) / § 8 후속 트리거(본 P2가 후속) | ✅ 완료 (5/26) |
| 5/22 협의 18항 박제 | [[1. Daily/2026-05-22.md]] L168~202 | ✅ 박힘 |
| spx-agent 관련 SESSION_HISTORY | `C:\Users\Administrator\Projects\spx-agent\.claude\SESSION_HISTORY.md` § 2026-05-22 Keycloak 부서 그룹 동기화 + § 2026-05-26 references/keycloak-sync.md 신설 | ✅ 박힘 |
| 승랑님 참고 PR/커밋 | `<PR 번호 또는 커밋 해시 — 미박이면 본 위임에서 git log로 식별>` | 옵션 |
| 194 Keycloak 인스턴스 | `192.168.10.194:8080` (realm: `Spelix`) | 운영 환경 |

> **본 위임은 옵시디언 Claudian에서 실행** — vault root(`C:\Users\Administrator\Documents\Obsidian\Daily\`)가 cwd. 외부 컨텍스트 폴더로 web-ui / spx-agent 절대경로 접근. spec 진입점 지도(`C:\Users\Administrator\Projects\spx-agent\.claude\CLAUDE.md`) L121 `docs/references/keycloak-sync.md` 행도 인덱스로 활용.

---

## 작업 원칙

- **읽기만, 쓰지 않음** — 코드/스키마/Keycloak 모두 조사만. 실제 통합 PR은 본 위임 결과 받은 후 별도
- **결정과 사유 박제** — "X로 가야 합니다" 단독 보고 금지, "Y 사실 발견 → 따라서 X" 사슬로 박음
- **리스크 사전 식별** — 18항 § 리스크 두 건(승랑님 패턴 부정합 / 라이브러리 버전 충돌)은 본 위임에서 사전 점검 의무
- **5/22 박제 활용** — 18항을 재조사하지 말 것. 18항을 기준으로 "확정" / "미확정 → 조사 필요" 분류

---

## 톱 4 우선 조사 (1 → 3 → 5~7 → 8 순)

### ① web-ui 프로젝트 위치/스택/로그인 구현 수준

- 어느 리포 / 어디 경로 / 마지막 커밋 시점
- 프레임워크 — **Next.js / React / Vue 중 무엇** (라우팅 방식, SSR 유무 결정에 직결)
- 빌드 도구 / 패키지 매니저 / 노드 버전 (lib 호환성 사전 점검 — 리스크 ⑱)
- 현재 로그인 화면 구현 수준 — **form UI만? submit 핸들러 stub? 부분 동작?**
- 라우팅 가드 패턴 — 비로그인 페이지 접근 제한 박혀 있나?

**산출**: web-ui 프로젝트 한 줄 요약 (예: `Next.js 13.4 / React 18 / pnpm / Node 18 / form UI만 + submit 미구현 / 가드 미구현`)

### ③ 백엔드 유무 (인증 흐름 분기점)

- web-ui가 LLM을 **직접 호출**(브라우저 → LLM)? 아니면 **자체 API 게이트웨이 경유**(브라우저 → web-ui API → LLM)?
- Next.js라면 **API route 사용 중**인지 (`pages/api/*` 또는 `app/api/*`)
- 백엔드 있음 → SSR + httpOnly cookie 가능 (보안 ↑)
- 백엔드 없음 → SPA-only PKCE + localStorage 강제

**산출**: 백엔드 유무 + 경로 + 통신 패턴 도식. 통합 전략(SPA-only PKCE vs SSR+httpOnly cookie) 자동 결정

### ⑤~⑦ + 승랑님 패턴 — spx-agent Keycloak 통합 base

- **⑤ spx-agent의 Keycloak 통합 코드 위치** — OIDC 흐름 / 토큰 발급·저장·리프레시 구현 파일 (`web/` 또는 `api/` 안)
- **⑥ 사용 라이브러리** — `keycloak-js` / `next-auth` / 자체 PKCE 구현 중 무엇 + 버전
- **⑦ api 측 토큰 검증 미들웨어** — 프론트 검증만으로 충분한지, 백엔드 검증도 박혀 있는지 (어느 미들웨어/decorator)
- **승랑님 참고 PR/커밋** — 통합 base가 된 PR 식별 + 핵심 파일 목록
- `account_service.py:209-365` 매칭 우선순위 (`.claude/docs/references/keycloak-sync.md` § 4) 그대로 적용 가능한지 사전 검토

**산출**: spx-agent Keycloak 통합 패턴 요약표 (위치 / 라이브러리 / 버전 / 검증 위치 / 토큰 흐름 도식). web-ui로 이식 가능 여부 판정

### ⑧ Spelix realm 분리 결정

- web-ui 전용 **client 신설** vs spx-agent와 **공유**
- redirect URI 다르면 신설 필수 (보통 다름 → 신설 거의 확정)
- realm 자체는 같이 쓸지 (계정/그룹 공유 의도면 같이) → **거의 공유 채택 예상**
  - 공유 시 `.claude/docs/references/keycloak-sync.md` § 4 매칭 우선순위 그대로 적용
  - `spx_accounts.sub` ↔ `user_entity.id` 단일 진실 유지

**산출**: realm/client 결정 + 신설 시 박을 설정값 초안 (client_id, root_url, redirect_uris, web_origins, access_type)

---

## 후속 14항 (톱 4 완료 후 자동/순차 결정)

| 항 | 내용 | 결정 방식 |
|---|---|---|
| 2 | 현재 로그인 화면 구현 수준 | ① 안에 흡수 — 코드 정독으로 즉시 |
| 4 | 세션/상태 관리 + 라우팅 가드 | ① 스택 결정 후 자연 결정 |
| 9 | 데모 계정 발급 (`DEMO` 그룹 신설 vs 단일 공유) | **PM 결정 동반** — 이사님 컨펌 후보 |
| 10 | redirect URI 사전 등록 (dev/배포 URL 모두) | ⑧ 결정 후 자동 — URL 목록 확정만 |
| 11 | 토큰 보관 (localStorage vs httpOnly cookie) | ③ 백엔드 유무로 자동 |
| 12 | LLM 호출 토큰 첨부 + 검증 위치 | ③ + ⑦로 자동 |
| 13 | 로그아웃 흐름 (Keycloak `/logout` + 로컬 정리) | 표준 패턴, ⑥ 라이브러리 따라감 |
| 14 | 토큰 만료/리프레시 (silent refresh vs 재로그인) | ⑥ 라이브러리 따라감 |
| 15 | 환경 변수 (`KEYCLOAK_URL` / `REALM=Spelix` / `CLIENT_ID` / `REDIRECT_URI` / dev·배포 분기) | ⑧ 결정 후 자동 |
| 16 | CORS — 194 Keycloak이 web-ui origin 허용 | 운영 작업 (194 서버 설정 변경 동반) |
| **17** | **리스크: 승랑님 패턴 ↔ web-ui SSR 차이/API route 유무 부정합** | ⑤~⑦ + ① 결과 교차 검증 — 부정합 발견 시 별도 보고 |
| **18** | **리스크: web-ui 옛 코드 라이브러리 버전 충돌 (keycloak-js ↔ React 버전)** | ① 스택 + ⑥ 라이브러리 선택 후 사전 호환성 점검 — `package.json` peerDependencies 교차 확인 |

---

## 출력 산출물 (보고 형식)

### A) 톱 4 조사 결과

```
### ① web-ui 스택
- 프레임워크: <Next.js 13.4 / ...>
- 노드/패키지: <Node 18 / pnpm 8.x / ...>
- 로그인 화면: <form UI만 / submit stub / ...>
- 라우팅 가드: <있음/없음>

### ③ 백엔드 유무
- 구조: <SPA-only / Next.js API route / 별도 백엔드>
- LLM 호출 위치: <브라우저 직접 / API 경유>
- 통합 전략 자동 결정: <SPA-only PKCE / SSR+httpOnly cookie>

### ⑤~⑦ spx-agent 패턴
- Keycloak 통합 파일: <경로 목록>
- 라이브러리: <keycloak-js x.x / next-auth x.x / 자체 PKCE>
- 백엔드 검증: <미들웨어 위치>
- 승랑님 base PR: <PR 번호 + 핵심 파일>

### ⑧ realm/client
- realm: Spelix 공유 (예상)
- client: <web-ui 전용 신설 — client_id 후보: ...>
- redirect_uris: <dev URL + 배포 URL 후보 목록>
```

### B) 통합 전략 결정 (자동 도출)

```
### 통합 전략: <SPA-only PKCE / SSR+httpOnly cookie> 채택
- 사유: ③ 백엔드 <유/무>, ⑥ 라이브러리 <X>, ⑦ 검증 <Y>
- 토큰 보관: <localStorage / httpOnly cookie>
- 리프레시: <silent refresh / 재로그인>
- 로그아웃: <Keycloak /logout 호출 → 로컬 정리>
```

### C) 호환성 리스크 리포트 (⑰·⑱)

```
### ⑰ 승랑님 패턴 부정합
- 부정합 발견: 0건 / N건
- (있으면) 부정합 위치 + 우회/적응 방안

### ⑱ 라이브러리 버전 충돌
- web-ui 현재 React 버전: <x.x>
- 선택 라이브러리 peerDependency: <X 요구>
- 충돌 여부: 없음 / 있음 (해결 옵션 N개)
```

### D) 다음 액션 PR 순서

본 위임 끝나면 시작할 통합 작업 PR 순서:

```
1. <Keycloak client 신설 (194 admin UI)>
2. <web-ui .env 등록 + 환경변수 추가>
3. <라이브러리 설치>
4. <로그인 페이지 submit 구현 + redirect>
5. <콜백 처리 + 토큰 저장>
6. <LLM 호출 토큰 첨부>
7. <라우팅 가드>
8. <로그아웃>
9. <CORS 설정 (194 admin)>
10. <dev/배포 redirect URI 추가 등록>
```

각 단계의 의존성 + 예상 변경 파일 박음.

### E) PM 컨펌 후보

```
- ⑨ 데모 계정 발급 방식 — DEMO 그룹 신설 vs 단일 공유 → 이사님 컨펌 필요
- (기타 발견 시)
```

---

## 가드 (delegation-standard 적용)

### § 1 — drift 발견 시 작업 중단
- web-ui 코드 ↔ 5/22 협의 18항 박제 사이 새 사실 발견 시 즉시 중단 + 사용자 보고
- spx-agent Keycloak 통합 코드 ↔ 본 위임 가정(라이브러리 X 사용 중) 불일치 시 보고
- 자동 보정 금지 — 18항 박힌 가정과 실측이 다르면 사용자 결정 요청

### § 2 — 자가 검증 거짓 가능성
- 각 조사 결과에 **출력 인용 첨부 의무**:
  - `package.json` 본문 (스택 / 라이브러리 / 버전)
  - `grep -rn "keycloak" web-ui/` 출력 (이미 통합 시도 흔적 있나)
  - `grep -rn "keycloak" spx-agent/` 출력 (통합 코드 위치)
  - `git log --oneline -20` (승랑님 PR base 식별)
  - Keycloak admin UI 또는 `curl /admin/realms/Spelix` 응답 (client 목록)
- "X라고 알고 있습니다" 단독 보고 금지

### § 5 — 한글 인코딩
- 본 위임은 코드 수정 0건이라 직접 함정 없음. 단 한글 경로/주석 inspect 시 Edit/Read tool 사용

---

## 제약 (STRICT)

- **코드 수정 금지** — web-ui / spx-agent / Keycloak 어느 쪽도 수정 0건. 조사만
- **Keycloak admin 변경 금지** — client 신설 / redirect URI 등록 / CORS 변경 모두 본 위임 후 별도 트랙
- **SESSION_HISTORY / CLAUDE.md / spec 미터치** — 본 위임은 결과 리포트만 (옵시디언 트랙에서 별도 박제)
- **18항 외 신규 분석 요청 금지** — 18항 안에서만 톱 4 + 후속 14
- **승랑님 PR 본문 변경/comment 금지** — 읽기만

---

## 후속 트리거 (보고 끝)

```
- ⏳ 통합 PR 트랙 — 본 위임 D) 다음 액션 PR 순서 따라 별도 트랙으로 진행
- ⏳ ⑨ 데모 계정 발급 방식 — 이사님 PM 컨펌 필요 (별도 트랙)
- ⏳ Keycloak 운영 변경 (client 신설 / redirect URI / CORS) — 통합 PR 1단계에서 처리
- ⏳ 본 위임 산출물 → SESSION_HISTORY 박제 (옵시디언 트랙)
```

---

## 위임 표준 가드 (필수 인지)

본 위임은 `hdd/delegation-standard.md`의 § 1·§ 2·§ 5 가드 적용 대상 (§ 3·§ 4 마이그레이션/컨테이너 범위 외):

1. **drift 발견 시 작업 중단** (§ 1) — 18항 박제 ↔ 실측 불일치 시 자동 보정 금지
2. **자가 검증 거짓 가능성** (§ 2) — 각 조사 결과에 grep/curl/cat 출력 첨부 의무
3. **한글 인코딩** (§ 5) — Edit/Read tool 사용

상세: `.claude/docs/hdd/delegation-standard.md` 참조

---

# 위임 시 박을 체크리스트

위임 트리거 누르기 전:
- [x] **web-ui 코드 경로 박힘** — `C:\Users\Administrator\Projects\n8n\poc\web-ui\dify-chat`
- [x] **`.claude/docs/references/keycloak-sync.md` 생성 완료** (후속 ② 5/26 끝남, 133 lines, § 1~§ 8) — `spx-agent\.claude\docs\references\keycloak-sync.md` 직접 인용 가능
- [x] **위임 환경 결정** — 옵시디언 Claudian (vault root cwd, 외부 컨텍스트 폴더 접근)
- [ ] 승랑님 참고 PR/커밋 번호 박힘 (옵션 — 미박이면 본 위임에서 `git log --oneline -30 | grep -i keycloak` 자체 식별)
- [ ] (옵션) 5/26 SESSION_HISTORY § 신설 작업의 미세 drift 1건 인지: `scripts/sync_mock_users_keycloak.sh` (.sh 변종)이 추가 존재 — 5/22 박제는 .py만 언급. 본 위임 § ⑤~⑦ 조사 시 두 변종 모두 확인 후 보고
- [ ] `.claude/docs/references/keycloak-sync.md` 생성 완료 (후속 ② 끝남) — 안 끝났으면 SESSION_HISTORY § 2026-05-22 § Keycloak 부서 그룹 동기화 직접 인용
- [ ] 위임 받을 환경 (옵시디언 Claudian vs VSCode Claude Code vs 별도) 결정
