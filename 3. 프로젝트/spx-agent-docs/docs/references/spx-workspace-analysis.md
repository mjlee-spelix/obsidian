---
title: Workspace 11p 전수 재검토
phase: Phase 4 / 클러스터 A / A4
status: 완료 (2026-06-04), 이사님 결정 2·5·6 반영 (2026-06-04 갱신)
audience: spx-agent 사용자 매뉴얼 작성자 (Phase 4 챕터 집필자)
purpose: |
  Dify 원본 Workspace 섹션 11p에 대해 (1) 5/29 매트릭스 비고 정정,
  (2) 2026-06-02 신설 전역 규칙 #1~#5 적용, (3) A1~A3 인용 위임 위치
  확정, (4) 2026-06-04 회의 결정 일괄 적용(API Extension 3p 삭제 확정,
  team-members-management 변환 박제, KC 추상화). 본 문서는 신규 챕터
  분석이 아니라 **원본 페이지 처리 결정의 정밀화**다.
status_history:
  - 2026-06-04 초안: 5/29 가정 정정 2건 + "유지·번역" 2건 격상 + API Extension 컨펌 대기 분리
  - 2026-06-04 갱신: 결정 2(API Extension 3p outbound 일방향 재분류 후 삭제 확정) + 결정 5(KC 추상화) + 결정 6(team-members-management 변환)
base_documents:
  - spx-app-permissions-analysis.md     # A1 권한 모델 코어
  - spx-knowledge-permissions.md        # A2 지식 도메인 특이사항
  - spx-departments-management.md       # A3 사용자/부서 운영 (team-members-management 변환 정전)
source_documents:
  - scope-mapping.md §전역 규칙·§8 Workspace 표
related_decisions:
  - 결정 2 (2026-06-04 회의) — 외부 연결 제거 (outbound), api-extension 3건 포함
  - 결정 5 (2026-06-04 회의) — KC 추상화 전역 규칙 #6 신설
  - 결정 6 (2026-06-04 회의) — team-members-management → A3 챕터 변환
code_verified_at: 2026-06-04
---

# 1. 본 문서의 위치

본 문서는 클러스터 A의 마지막 산출물. A1~A3에서 정립한 권한 모델·지식 동기화·부서 운영 흐름을 **Workspace 11p에 일관 적용**하는 결정표.

신규 챕터를 만들지 않는다 — 원본 페이지의 액션(유지/번역/부분 수정/삭제/검토)을 정밀화하고, A1~A3로 위임할 콘텐츠를 명확히 한다. Phase 4 본격 작성 시 본 결정을 보고 원본 한 페이지씩 처리한다.

## 1.1 2026-06-04 이사님 결정 반영 결과

본 갱신에서 다음 3건의 결정이 본 산출물에 흡수됨:

| 결정 | 본 문서 적용 위치 | 요약 |
|------|----------------|------|
| **결정 2** — 외부 연결(outbound) 제거 | §2 결정표 (8/9/10), §4 (전면 갱신) | API Extension 3p(`api-extension.mdx` / `external-data-tool-api-extension.mdx` / `moderation-api-extension.mdx`)는 컨펌 대기에서 **삭제 확정**으로 이동. 양방향 표기를 outbound 일방향으로 재분류 |
| **결정 5** — KC 추상화 (전역 규칙 #6) | §3.2 Personal Settings | 본 문서의 KC 관련 표기는 A3 [[references/spx-departments-management#0-표기-가이드-—-kc-추상화-전역-규칙-6]] 따름. Personal Settings의 SSO 관련 본문은 "외부 시스템(예, Keycloak)" 표기 |
| **결정 6** — team-members-management 변환 | §3.3 Team Members | "삭제 → 부분 수정"이었던 5/29 정정이 **변환**으로 재확정. A3 [[references/spx-departments-management]] §1·§8.2가 변환 정전, 본 산출물은 그 인용처 |

## 1.2 액션 분포 변화

| 시점 | 유지·번역 | 부분 수정 | 삭제 | 컨펌 대기 | 변환 | 합계 |
|------|---------|---------|------|---------|------|------|
| 5/29 매트릭스 | 2 | 2 | 4 | 3 (검토) | — | 11 |
| 6/4 초안 | — | 5 | 3 | 3 | (3=부분 수정에 흡수) | 11 |
| **6/4 갱신** | — | **4** | **6** | **0** | **1** (Team Members) | **11** |

> **6/4 초안**: 5/29의 "유지·번역" 2건(App Management·Model Providers)이 SaaS 분기 잔재 정리 필요로 **부분 수정**으로 격상되어 유지·번역 컬럼이 0이 됨 → 합계는 부분 수정에 흡수되며 11 유지.
>
> **6/4 갱신**: Team Members가 "부분 수정"에서 **변환**으로 분리되어 부분 수정 5→4로 감소. API Extension 3p가 **컨펌 대기→삭제**로 이동하여 삭제 3→6, 컨펌 대기 3→0. 합계 11 유지.

> **인접 분석**: A1·A2·A3, B1 [[references/spx-audit-log-analysis]]

---

# 2. 원본 11p 처리 결정표 (확정, 6/4 갱신)

> 액션 4종: **유지·번역** / **부분 수정** / **삭제** / **변환** (Dify 원본 페이지를 신규 챕터로 대체)
>
> 5/29 매트릭스 또는 6/4 초안에서 변경된 항목은 **변경** 컬럼에 사유 명시.

| # | 원본 페이지 | 경로 | 5/29 결정 | 6/4 초안 | **6/4 갱신** | 변경 / 사유 |
|---|-----------|------|----------|---------|------------|------------|
| 1 | Workspace Overview | `workspace/readme` | 부분 수정 P1 | 부분 수정 P1 | **부분 수정 P1** | 동일. §3.1 처리 지침 박제 |
| 2 | Personal Settings | `workspace/personal-account-management` | 부분 수정 P2 — "KC SSO 기반 계정" | 부분 수정 P2 (정정) | **부분 수정 P2** | 5/29 가정 정정 유지. 결정 5(KC 추상화) 표기 적용. §3.2 |
| 3 | Manage Members | `workspace/team-members-management` | **삭제** — "UI 멤버 관리 없음" | 부분 수정 P1 (변경) | **변환 P1 (재변경)** | 결정 6 적용 — "부분 수정"이 아니라 **변환**. Dify 원본 자리를 A3 [[references/spx-departments-management]]로 대체. §3.3 |
| 4 | Manage Apps | `workspace/app-management` | 유지·번역 P1 | 부분 수정 P1 (변경) | **부분 수정 P1** | 6/4 초안 유지. SaaS 분기·DSL 버전 SaaS/Community 분기 정리. 권한 절은 A1로 위임. §3.4 |
| 5 | Model Providers | `workspace/model-providers` | 유지·번역 P2 | 부분 수정 P2 (변경) | **부분 수정 P2** | 6/4 초안 유지. "System Providers" SaaS 개념 삭제·Custom Providers 일반화. §3.5 |
| 6 | Plugins | `workspace/plugins` | **삭제** | 삭제 확정 | **삭제** 확정 | Marketplace/Plugin 시스템 미보유 (결정 1) |
| 7 | Subscription Mgmt | `workspace/subscription-management` | **삭제** | 삭제 확정 | **삭제** 확정 | SaaS 요금제 없음 (전역 규칙 #1) |
| 8 | API Extension Overview | `workspace/api-extension/api-extension` | 검토 P3 | 컨펌 대기 | **삭제** (재변경) | **결정 2 적용** — outbound 일방향으로 재분류 후 삭제 확정. §4 |
| 9 | External Data Tool | `workspace/api-extension/external-data-tool-api-extension` | 검토 P3 | 컨펌 대기 | **삭제** (재변경) | 결정 2 적용 (동일) §4 |
| 10 | Moderation Extension | `workspace/api-extension/moderation-api-extension` | 검토 P3 | 컨펌 대기 | **삭제** (재변경) | 결정 2 적용 (동일) §4 |
| 11 | Cloudflare Worker | `workspace/api-extension/cloudflare-worker` | **삭제** | 삭제 확정 | **삭제** 확정 | SaaS/Cloudflare 전용 |

## 결정 변동 요약

- **5/29 → 6/4 갱신 변경**: 2 (KC 추상화 적용), 3 (삭제→변환), 4 (유지→부분 수정), 5 (유지→부분 수정), 8/9/10 (검토→삭제)
- **5/29 → 6/4 갱신 유지**: 1, 6, 7, 11
- **6/4 초안 → 6/4 갱신 변경**: 3 (부분 수정→변환), 8/9/10 (컨펌 대기→삭제)
- **컨펌 대기 잔여**: **0건** (모두 결정 적용 완료)

## 액션별 페이지 수 분포

| 액션 | 페이지 수 | 페이지 |
|------|---------|-------|
| 부분 수정 | 4 | 1, 2, 4, 5 |
| 삭제 | 6 | 6, 7, 8, 9, 10, 11 |
| 변환 | 1 | 3 (Team Members → A3 챕터로 대체) |
| **합계** | **11** | |

---

# 3. 페이지별 처리 지침

## 3.1 Workspace Overview (`readme`)

원본 핵심: workspace 정의, Dify Cloud vs CE 분기, 다중 workspace, 5종 역할.

**부분 수정 처리**:

| 원본 요소 | 처리 |
|----------|------|
| "Dify는 워크스페이스 중심" 도입 문단 | spx-agent로 교체 (Dify 브랜드 제거, 전역 규칙 #5) |
| 워크스페이스 멘탈 모델 다이어그램 | 유지하되 "Billing" 줄 삭제(전역 규칙 #1) |
| Dify Cloud / CE 분기 — Workspace Creation 절 | **CE 단일 서술로 통합** (전역 규칙 #2). "관리자 이메일·비밀번호는 설치 시 설정" 한 줄만 |
| **Multiple workspaces** 절 | **단일 워크스페이스 정책 명시로 교체**. "spx-agent는 워크스페이스를 한 개 사용합니다" 추가 |
| 5종 역할 요약 | 유지 + 한국어 매핑 적용(소유자/관리자/편집자/일반 멤버/지식 관리자 — [[conventions]]). 권한 상세는 [[references/spx-app-permissions-analysis#7.3 워크스페이스 역할별 capability 매트릭스]]로 위임 한 줄 |
| Workspace Navigation 절 — Billing (Cloud only) 표기 | **삭제**. 사이드바 메뉴는 [[references/spx-departments-management#2 화면 진입]] 인용 |

**추가 1문단**: spx-agent의 부서·권한 모델을 한 줄로 요약(A1·A3 챕터 링크).

## 3.2 Personal Settings (`personal-account-management`) — 5/29 정정 + 결정 5 KC 추상화

원본 핵심: 멀티 워크스페이스 가정, Cloud vs Community 로그인 분기.

**5/29 가정 정정**:
> 5/29 비고: "KC SSO 기반 계정 → 프로필 설정만"
>
> A4 확정: spx-agent 로그인은 **systemFeatures 플래그로 4종 중 활성화** — 이메일+비밀번호 / 이메일+코드 / 소셜 / SSO. 외부 IdP는 SSO 옵션을 통해 통합되며 항상 강제되는 것이 아님. 배포 환경 설정에 따라 옵션 노출.

**부분 수정 처리**:

| 원본 요소 | 처리 |
|----------|------|
| **Multi-Workspace Access** 절 | **삭제** 또는 "spx-agent는 단일 워크스페이스" 한 줄로 축소 |
| 워크스페이스 selector 안내 | **삭제** (단일 워크스페이스 정책) |
| **Login Methods by Edition** 표 (Community vs Cloud) | **삭제** 또는 spx-agent 실제 4종 + systemFeatures 플래그 안내로 교체 |
| **Account Linking** (Dify Cloud 전용) | **삭제** |
| **Security** 절의 Cloud vs Community 분기 | **삭제** — 단일 서술로 통합 |
| Profile / Display Name / Email / Language | 유지·번역 |

**추가 1문단** (결정 5 KC 추상화 적용):
> "SSO로 로그인한 경우 사용자명·이메일은 **외부 시스템(예, Keycloak)** 이 관리합니다. spx-agent에서 수정해도 다음 로그인 시 외부 시스템 값으로 동기화됩니다." → A3 [[references/spx-departments-management#5.6-챕터-본문-추상화-표기-예시]] 인용.

**KC 추상화 적용 가이드**:
- 본 페이지의 SSO 관련 본문에서 "Keycloak" 직접 노출 금지 — A3 §5.6 추상화 톤 그대로 사용
- 보존 예외(분석본 영역): 본 §3.2의 코드 식별자(`normal-form.tsx`, `systemFeatures.sso_enforced_for_signin` 등)는 그대로 유지 — 챕터 본문에는 옮기지 않음

> **확인 필요 (Phase 4 본격 작성 시)**: Personal Settings UI 진입점이 워크스페이스 설정 사이드바인지 별도 사용자 메뉴인지. `account-setting/index.tsx`의 `ACCOUNT_SETTING_TAB`에는 Personal Account 탭이 없음(워크스페이스 그룹 + 일반 그룹의 Language만). 헤더 사용자 메뉴에서 별도 다이얼로그로 노출될 가능성 — 코드 추가 검증 필요.

## 3.3 Manage Members (`team-members-management`) — 결정 6 변환 확정

원본 핵심: Team Size Limits (Free/Pro/Team/Enterprise), 5종 역할 상세, 초대 절차.

**처리 결정 경위**:
> 5/29: **삭제** ("UI 멤버 관리 없음" 가정 오류)
>
> 6/4 초안: **부분 수정** (A3로 80% 위임하는 형태로 처리하려 함)
>
> **6/4 갱신 (결정 6 적용): 변환** — Dify 원본 페이지를 신규 챕터 [[references/spx-departments-management]]로 대체. 부분 수정처럼 원본 페이지에 한국어를 얹는 게 아니라, **원본 자리에 A3 챕터를 통째로 배치**한다.

### 변환 처리 정전 — A3 §8.2 인용

본 §3.3은 원본 페이지의 변환 위치 명시만 담당. 실제 변환 결과는 [[references/spx-departments-management#8.2-결정-6-변환-처리-—-매뉴얼-구성-영향]]에 정전화되어 있다. 본 문서는 그 인용처.

| 원본 요소 | 변환 후 위치 |
|----------|------------|
| **Team Size Limits** 절 (Free/Pro/Team/Enterprise) | A3로 옮기지 않음 — 전역 규칙 #1로 삭제 |
| **Workspace Roles** 절 (5종 Accordion) | A1 [[references/spx-app-permissions-analysis#7.3-워크스페이스-역할별-capability-매트릭스]] |
| **Adding Team Members** 절 (이메일 초대) | A3 §5.6 — 외부 IdP 그룹 동기화로 안내 (KC 추상화 적용) |
| **Member Management** 절 | A3 §4 멤버 페이지·§3.5 멤버 관리 모달 |
| **Multiple Workspaces** | A3로 옮기지 않음 — 단일 워크스페이스 정책으로 삭제 |
| **Access Patterns** 절 | A1 [[references/spx-app-permissions-analysis]] |

### Phase 4 본격 작성 시 적용 방식

1. `en/use-dify/workspace/team-members-management.mdx` 자리에 신규 챕터 배치 (한국어 본문은 `ko/use-spx-agent/workspace/departments/readme.mdx`로 작성됨)
2. `sidebars.js` Option α 적용(7번 작업, 2026-06-04 완료) — 단일 "워크스페이스" 카테고리 안에 사용자·부서 관리 → 권한 설정 → 개인 계정 순서로 등록
3. A1 [[references/spx-app-permissions-analysis]] §7.3 역할 매트릭스를 한 절로 인용하거나 별도 챕터(권한 설정)로 위임

→ 결과적으로 사용자가 워크스페이스 그룹에서 "사용자/부서 관리" 챕터를 만나면 그것이 원본 Team Members의 spx-agent 변환판이라는 매뉴얼 구성.

## 3.4 Manage Apps (`app-management`)

원본 핵심: 앱 편집/복제/DSL import-export/삭제.

**부분 수정 처리** (5/29 "유지·번역"에서 격상):

| 원본 요소 | 처리 |
|----------|------|
| 도입 — "Dify provides..." | "spx-agent" 또는 일반화 |
| Edit / Duplicate / Import & Export / Delete 4종 카드 | 유지·번역 |
| **Export DSL Version compatibility** — "SaaS users / Community users" 분기 | **삭제** (전역 규칙 #2). CE 단일 서술 |
| Knowledge base connections | 유지하되 권한 영향 한 줄 추가 — [[references/spx-knowledge-permissions]] 링크 |
| Secret 환경 변수 export 경고 | 유지 |
| **권한 관련 절 신규 추가 (1문단)** | "앱을 삭제·복제하려면 해당 앱에 대한 권한이 필요합니다. 자세한 권한 모델은 [권한 설정] 챕터를 참조하시기 바랍니다." → A1 챕터 링크 (구 명칭 "앱 권한 설정", 6/4 rename) |

## 3.5 Model Providers (`model-providers`)

원본 핵심: System vs Custom Providers, 설정 절차, 지원 모델 목록.

**부분 수정 처리** (5/29 "유지·번역"에서 격상):

| 원본 요소 | 처리 |
|----------|------|
| **System Providers** 절 (Dify 운영 사전 설정 모델) | **삭제** — SaaS 개념. Custom Providers만 본문에 남김 |
| 모델 제공자 목록 (OpenAI, Anthropic, Google, Cohere, Ollama) | 유지·번역. Dify 브랜드 제거 |
| Settings → Model Providers 진입 동선 | 유지 (A3 §2 사이드바 메뉴 인용) |
| **권한** 한 줄 — "Only workspace admins and owners can configure" | 유지·번역 + 한국어 매핑 (관리자·소유자 — [[conventions]]) |
| **"로드 밸런싱" 절** | **삭제** — spx-agent 배포에서 **기본 비활성** (전역 규칙 #1). 아래 코드 검증 참조 |

> **🔴 로드 밸런싱 = CE 기본 비활성 (2026-06-15 코드 검증)**. 활성화 조건은 두 경로뿐: (1) env `MODEL_LB_ENABLED` — 기본값 `False` (`api/configs/feature/__init__.py:663`), (2) billing API 응답 — `BILLING_ENABLED`(기본 `False`)가 켜진 **SaaS 환경에서만** 적용 (`api/services/feature_service.py:281, 348`). spx-agent 실 배포(`docker/.env`·`.env.example`·docker 전체)에 `MODEL_LB_ENABLED` 설정 **없음** → 코드 기본값 `False` = 사용자가 접근 불가. ⚠️ "엔터프라이즈 전용"이 아니라 **env 플래그로 꺼진 기능** — 배포에서 `MODEL_LB_ENABLED=true`로 켜면 그때 본문 복원. 원문이 유료 SaaS/엔터프라이즈로 안내하므로 현재는 본문에서 **"## 로드 밸런싱" 절 삭제** (규칙 #1·#8). 출처: [[1. Daily/2026-06-15]].

→ 본 페이지에서 vLLM·로컬 모델 등 spx-agent 자주 사용 패턴 추가 보강 후보 — Phase 4 본격 작성 시 결정. [[scope-mapping]] Phase 1.3 재검토에서 "nodes/llm, workspace/model-providers vLLM 언급 제거"가 적용되므로 vLLM은 일반화.

## 3.6 삭제 확정 6p (6/4 갱신 — 결정 2 추가 적용)

5/29부터 삭제 확정이던 3p에 결정 2 적용으로 API Extension 3p가 추가되어 총 6p.

| 페이지 | 삭제 사유 |
|-------|---------|
| Plugins | Marketplace/플러그인 시스템 전체 제거 (결정 1) |
| Subscription Management | SaaS 요금제 없음 (전역 규칙 #1) |
| Cloudflare Worker | SaaS/Cloudflare 전용 |
| API Extension Overview | outbound 일방향 (결정 2) — §4 |
| External Data Tool | outbound 일방향 (결정 2) — §4 |
| Moderation Extension | outbound 일방향 (결정 2) — §4 |

**처리 절차**: `docs.json`(원본 보존) 또는 `sidebars.js`에서 항목 제거. `en/` 원본 파일은 보존(원본 보존 원칙).

---

# 4. API Extension 3p — 결정 2 적용 (삭제 확정)

## 4.1 결정 경위

- **5/29**: 검토 P3 — "사내 API 확장 가능 여부" 미결
- **6/4 초안**: 컨펌 대기 — 사내망 외부 노출 정책 결정 필요로 슬롯 마련
- **6/4 갱신 (결정 2 적용)**: **삭제 확정** — 양방향(bidirectional) 표기로 보였으나 실제로는 **outbound 일방향**(spx-agent가 외부 API를 호출) 으로 재분류되어 결정 2 외부 연결 제거 범위에 포함

> 결정 2 원문: "spx-agent에서 외부 SaaS·서비스로 outbound 호출하는 기능 모두 제거. 영향 페이지 ... workspace/api-extension/* 3건 (양방향 표기 정정 → outbound 일방향)"

## 4.2 outbound 일방향 재분류 근거

원본 코드(`workspace/api-extension/api-extension.mdx`)의 API 사양:

```
POST {Your-API-Endpoint}
Authorization: Bearer {api_key}
```

→ spx-agent가 **사용자가 운영하는 외부 엔드포인트로 POST 요청을 보내는** 흐름. 호출 방향은 spx-agent → 외부. 사내 시스템이 spx-agent를 호출하는 inbound가 아님(inbound는 결정 4의 `publish/developing-with-apis` 등 3건이 해당).

→ 폐쇄망 가정에서 outbound 의존을 늘리는 페이지라 결정 2 범위에 정확히 들어맞음.

## 4.3 삭제 처리 (3p 일괄)

| # | 페이지 | 삭제 사유 |
|---|-------|---------|
| 8 | `api-extension/api-extension` (Overview) | outbound 일방향 (결정 2) |
| 9 | `api-extension/external-data-tool-api-extension` | outbound 일방향 (결정 2) |
| 10 | `api-extension/moderation-api-extension` | outbound 일방향 (결정 2). 사내 모더레이션이 필요해도 별도 챕터로 신설하는 게 깔끔 — 본 페이지는 일단 삭제 |

11 (Cloudflare Worker)은 SaaS/Cloudflare 전용으로 5/29부터 삭제 확정이라 본 결정과 독립.

## 4.4 UI 잔존 (코드 자체는 활성)

- 워크스페이스 설정 사이드바의 **`API_BASED_EXTENSION` 탭**은 코드에 그대로 존재 (`account-setting/index.tsx` 라벨: `settings.apiBasedExtension`)
- 즉 **사용자 매뉴얼에서 노출만 제거**하는 결정 — 코드 자체를 비활성화하라는 의미는 아님
- 사내에서 실제로 API Extension 기능을 사용하면, 운영자 부록·내부 가이드에서 별도로 다룬다. 일반 사용자 매뉴얼에는 노출 안 함

## 4.5 후속 영향

- `scope-mapping` Workspace 표에서 8/9/10 행을 검토 P3 → 삭제로 갱신 필요 (Phase 4 본격 작성 진입 전)
- 사용자가 워크스페이스 설정에서 API Extension 탭을 발견해 "이게 뭔가요?" 물을 때를 대비, **FAQ 1줄 후보** — Phase 4 본격 작성 시 결정

---

# 5. 전역 규칙 #1~#5 적용 일괄 점검 (11p 공통)

각 페이지 부분 수정·유지·번역 작업 시 다음을 일괄 점검:

| 규칙 | 11p에서 발견된 위치 |
|------|--------------------|
| #1 SaaS 플랜 제거 | Manage Members(Team Size Limits), Subscription(전체), Workspace Overview(Billing 줄) |
| #2 Cloud version 분기 | Personal Settings(Login Methods 표), App Management(DSL Version SaaS/Community), Workspace Overview(Workspace Creation) |
| #3 Sandbox 워크스페이스 분리 | 11p에서 직접 등장 없음 — 확인 필요 |
| #4 자체 호스팅 한정 분기 + env var | Workspace Overview·Personal Settings 일부 — 발견 시 [[references/deployment-config-extracts]]로 추출 |
| #5 Dify 브랜드 + 외부 채널 + Marketplace | 전 페이지 — Dify 텍스트 일괄 교체. Plugins/Marketplace 언급은 페이지 자체 삭제로 회수 |

---

# 6. A1·A2·A3 인용 매핑 (Phase 4 본격 작성 시 사용)

본 클러스터 A 마감으로 Workspace 11p의 권한·부서 관련 콘텐츠는 모두 위임 가능.

| 원본 페이지 위치 | 위임 대상 |
|--------------|---------|
| Workspace Overview의 역할 5종 매트릭스 | A1 [[references/spx-app-permissions-analysis#7.3-워크스페이스-역할별-capability-매트릭스]] |
| Workspace Overview의 사이드바 메뉴 | A3 [[references/spx-departments-management#2-화면-진입]] |
| Personal Settings의 SSO 안내 (결정 5 KC 추상화) | A3 [[references/spx-departments-management#5.6-챕터-본문-추상화-표기-예시]] — "외부 시스템(예, Keycloak)" 톤 그대로 사용 |
| **Manage Members 전체 (변환)** | **A3 [[references/spx-departments-management#1-본-문서의-위치]] + [[references/spx-departments-management#8.2-결정-6-변환-처리-—-매뉴얼-구성-영향]]** — 단순 위임이 아니라 원본 자리를 대체 |
| Model Providers의 권한 한 줄 | A1 §7.3 — "관리자(Admin) 이상" 표기 정합성만 |
| App Management의 권한 한 줄 | A1 [[references/spx-app-permissions-analysis]] |
| KC 표기 정책 (전 페이지 공통) | A3 [[references/spx-departments-management#0-표기-가이드-—-kc-추상화-전역-규칙-6]] — 본 문서·챕터 본문 모두 이 가이드 따름 |

---

# 7. 인터리브 임시 작성 메모 — 대표 1p 검증 (작성 완료)

> Workspace 11p 전수를 1p로 압축하는 건 의미 없음 → 5/29 가정이 가장 크게 변경된 **Personal Settings** 한 페이지에 대해 임시 1p 작성으로 검증.

## 7.1 작성 결과

- **현재 위치** (Option α 적용 완료, 2026-06-04 7번 작업): `ko/use-spx-agent/workspace/personal-settings/readme.mdx`
- (이전 경로) `ko/use-spx-agent/workspace-management/personal-settings/readme.mdx` — Option α 적용으로 이동·`workspace-management/` 폴더 삭제
- 5번 작업(2026-06-04)에서 KC 추상화 적용 완료
- 빌드 통과 (§9.1 마지막 항목 참조)

## 7.2 챕터 골격 (실제 작성본 요지)

> **결정 5 KC 추상화 적용 후 표기**. §3.2 처리 지침과 일관. 챕터 본문 KC 추상화 적용은 progress.md "남은 작업" 별도 항목.

```
# 개인 설정

## 소개
- 본인 프로필·언어·로그인 메커니즘 한눈에

## 진입
- 헤더 우측 사용자 메뉴 → 설정(확인 필요)

## 프로필
- 표시명·이메일·아바타
- SSO 로그인 시 이름/이메일은 외부 시스템(예, Keycloak)이 관리

## 로그인 방식
- spx-agent 배포에 따라 4종 중 활성화 (이메일+비밀번호 / 이메일+코드 / 소셜 / SSO)
- 활성화 여부는 워크스페이스 관리자가 결정

## 언어
- 인터페이스 언어 변경

## 자주 묻는 질문
- "이름을 바꿨는데 적용 안 됩니다" → SSO 사용자는 외부 시스템(예, Keycloak)에서 변경
- "비밀번호를 잊었습니다" → 로그인 방식에 따라 다름 (배포 환경별 안내)
```

> ⚠️ **챕터 본문 KC 추상화 환기** — 위 골격은 추상화 톤으로 박제된 미래 상태. 현재 `personal-settings/readme.mdx` 본문에는 일부 "Keycloak" 직접 노출이 남아 있어, A3 [[references/spx-departments-management#5.6-챕터-본문-추상화-표기-예시]] 톤으로 교체 필요. progress.md "A3 챕터 본문 KC 추상화" 항목과 같은 작업 묶음으로 처리 권장.

본 임시 작성은 §3.2 처리 지침의 실효성 검증 목적. Phase 4 본격 작성 시 확장.

---

# 8. B·C 클러스터가 본 문서를 인용하는 방식

| 후속 산출물 | 본 문서에서 인용할 섹션 |
|------------|----------------------|
| [[references/spx-monitoring-analysis]] (B2) | §3.4 App Management — Monitor Analysis 페이지의 권한 영향 한 줄 |
| [[references/spx-get-started-analysis]] (C1) | §3.2 로그인 방식 4종 — Quick Start "로그인" 절의 정확한 안내 |

---

# 9. 후속 작업 체크리스트 (A4 마감용)

## 9.1 초안 작성 (2026-06-04)

- [x] 원본 11p grep 및 핵심 키워드(SaaS·Cloud version·Marketplace) 식별
- [x] scope-mapping 8 Workspace 표 vs A4 결정 차이 박제
- [x] 5/29 가정 오류 2건(Personal Settings KC SSO 가정 / Team Members 삭제) 정정
- [x] 5/29 "유지·번역" 2건(App Management / Model Providers) → 부분 수정 격상
- [x] 11p별 처리 지침(§3)·전역 규칙 적용 위치(§5)·A1~A3 인용 매핑(§6) 박제
- [x] API Extension 3p를 컨펌 트랙으로 분리(§4 초안)
- [x] 인터리브 — Personal Settings 대표 1p 작성

## 9.2 이사님 결정 2·5·6 반영 갱신 (2026-06-04, "남은 작업 2번")

- [x] frontmatter `status` 갱신·`status_history`·`related_decisions` 신설
- [x] §1.1 결정 2·5·6 반영 위치 표 신설
- [x] §1.2 액션 분포 변화(시점별) 표 신설 — 컨펌 대기 0건 명시
- [x] §2 결정표 — 컨펌 대기 컬럼 → 6/4 초안·6/4 갱신 컬럼 분리, Team Members 변환·API Extension 3p 삭제 확정 반영

### 9.2.1 검토 결과 5건 처리 (2026-06-04 자체 검토)

> progress.md "남은 작업 2번" 라인 172-177에서 식별된 분석본 자체 수정 5건. 실질 2건 + minor 3건.

- [x] §7.2 인터리브 골격 KC 직접 노출 모순 해소 — "Keycloak SSO"·"KC SSO" → "SSO + 외부 시스템(예, Keycloak)" 톤으로 교체, §7 본문 작업 시 personal-settings 챕터 KC 추상화 환기 박스 추가
- [x] §1.2 액션 분포 표 — 5/29 행 합계 9 모순 해소. "유지·번역" 컬럼 추가하여 합계 11 맞춤. 5/29 유지·번역 2건이 6/4 초안에서 부분 수정으로 격상된 경위 각주 추가
- [x] §3.6 삭제 확정 — 5/29부터 3p → 6/4 결정 2 적용으로 6p로 확장 갱신. Plugins 사유를 "전역 규칙 #5" → "결정 1"로 통일 (§2와 일관)
- [x] §7 헤더 "(작성 예정)" stale → "(작성 완료)" 갱신, §9.1 빌드 통과 항목 인용
- [x] §7.1 경로 혼용 정리 — 현재 위치(`workspace-management/personal-settings/`)와 목표 위치(`workspace/personal-settings/`, Option α 이동 후)를 명시 분리. 이후 7번 작업(2026-06-04)에서 목표 위치로 실제 이동 완료, §7.1도 동기화
- [x] §2 결정 변동 요약 갱신 (5/29→갱신·초안→갱신 분리)
- [x] §2 액션별 분포 표 갱신 — 부분 수정 4·삭제 6·변환 1, 컨펌 대기 제거
- [x] §3.2 Personal Settings — 결정 5 KC 추상화 적용 지침 추가, "외부 시스템(예, Keycloak)" 톤 박제
- [x] §3.3 Team Members — 결정 6 변환 확정으로 재구성, A3 §8.2 변환 처리 표 인용
- [x] §4 API Extension — 컨펌 트랙에서 결정 2 적용 (삭제 확정) 전면 갱신, outbound 일방향 재분류 근거 박제
- [x] §6 인용 매핑 — A3 §0·§5.6·§1·§8.2 인용 위치 추가

## 9.3 별도 작업 추적 — 2026-06-04 모두 완료

본 산출물 갱신 후 진행된 일련의 작업(3·4·5·6·7·8번). 사이드바 구조는 Option G → **Option α**로 진화하여 폴더 경로 일부 변경됨.

- [x] **3번 — A2 산출물 갱신** (결정 2·4 외부 연결 패턴 반영) + 자체 검토 5건 + 챕터 본문 연쇄 1건
- [x] **4번 — A1 산출물 §13 정리** (결정 트랙 영향 정리) + 자체 검토 1건
- [x] **5번 — A3 + personal-settings 챕터 본문 KC 추상화** (`workspace-management/.../`의 14개 직접 노출 행 추상화) + 자체 검토 1건
- [x] **6번 — A1 챕터 본문 명칭 갱신 + `app-permissions/` → `workspace/permissions/` 폴더 이동** + 자체 검토 1건
- [x] **7번 — Option α 적용**: A3·personal-settings → `workspace/` 통합 이동 + sidebars 워크스페이스 그룹 통합 + 자체 검토 1건
- [x] **8번 — Option α 통계·감사 그룹 신설**: `dashboard/` → `analytics-audit/dashboard/` 이동 + sidebars 그룹 신설 + 라벨 스왑 오류 정정

## 9.4 Phase 4 본격 작성 시 처리

- [ ] Personal Settings UI 정확한 진입점 확인 (`account-setting` 외 헤더 사용자 메뉴 확인)
- [ ] scope-mapping 8 Workspace 표를 본 결정으로 갱신 (컨펌 대기 → 삭제·변환 반영)
- [ ] API Extension 사이드바 탭이 화면에 남아 있을 때의 FAQ 1줄 추가 여부
- [x] Manage Members 페이지 자리에 A3 챕터 배치 — Option α 사이드바 적용으로 단일 "워크스페이스" 카테고리 안에 사용자·부서 관리 등록(7번 작업, 2026-06-04). 원본 Dify의 Team Members 슬롯 자체는 매트릭스 변환 처리로 흡수됨(결정 6)