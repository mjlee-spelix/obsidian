---
title: Get Started 3p — 부분 수정 처리 분석 (간소화)
phase: Phase 4 / 클러스터 C / C1
status: 완료 (2026-06-04)
audience: spx-agent 사용자 매뉴얼 작성자 (Phase 4 챕터 집필자)
purpose: |
  Get Started 그룹 3p(Introduction / Quick Start / Key Concepts)의 "부분 수정"
  처리 지침을 박제. 신규 코드 분석 없이 A1(권한 모델)·A3(부서 운영)의 결과를
  인용해 각 페이지에 들어갈 한 줄·정의·티저의 정확한 위치와 표현만 정한다.
  클러스터 A·B에서 키워드·정의가 안정된 뒤 마무리하는 후행 작업(낮은 시급도).
base_documents:
  - spx-app-permissions-analysis.md   # A1 — 권한 모델 코어, 역할 매트릭스 §7.3
  - spx-departments-management.md      # A3 — 부서 정의, 외부 시스템 그룹 동기화 §5.4/§5.6
  - spx-dashboard-analysis.md          # 대시보드 티저 근거
  - spx-audit-log-analysis.md          # 감사로그 티저 근거
related_decisions:
  - 결정 1 (2026-06-04) — Marketplace/플러그인 전체 제거
  - 결정 3 (2026-06-04) — MCP 유지
  - 결정 5 (2026-06-04) — KC 추상화 (전역 규칙 #6)
source_refs:
  - en/use-dify/getting-started/introduction.mdx
  - en/use-dify/getting-started/quick-start.mdx
  - en/use-dify/getting-started/key-concepts.mdx
code_verified_at: 불필요 — A1·A3·대시보드·감사로그 분석본 인용 (Get Started 자체는 신규 코드 분석 없음)
---

# 0. 본 문서의 위치 — 간소화 사유

Get Started 3p는 모두 **부분 수정**이며, 차이의 본질은 두 가지뿐이다:

1. **rebranding·전역 규칙 적용** (Dify 브랜드·외부 채널·Cloud/SaaS 개념 제거) — 모든 페이지 공통
2. **spx-agent 신규 기능을 가리키는 한 줄·정의·티저 삽입** — 그 내용은 이미 A1·A3에 정전화되어 있음

따라서 C1은 **신규 코드 deep dive가 불필요**하다. 본 문서는 "어디에, 무엇을, 어떤 표현으로" 넣을지만 박제하고, 실제 개념 정의·동작은 A1·A3로 위임한다. progress.md 클러스터 C 비고("거의 완료 — 간소화 가능")와 일치.

> **선행 안정화 확인**: 본 작업은 A1~A4·B1 완료 후 진행(2026-06-04). 인용하는 용어(부서·RBAC·가시성·ACL·소유권·외부 시스템 추상화)가 모두 확정된 상태라 표현 흔들림 없음.

---

# 1. 처리 개요

| 원본 페이지 | 경로(원본) | 액션 | 핵심 변경 | 인용 |
|------------|----------|------|----------|------|
| Introduction | `getting-started/introduction` | 부분 수정 | Dify 소개 → spx-agent 소개 재작성, CardGroup 재구성(외부 채널 제거), **신규 챕터 4종 키워드 티저** | A1·A3·대시보드·B1 |
| Quick Start | `getting-started/quick-start` | 부분 수정 | Cloud/Sandbox/credits/플러그인 설치 의존 정리, **로그인=사용자명/비밀번호 폼**(+로그인 시 부서 자동 배정 한 줄), 앱 생성 단계 **부서·권한 한 줄** | A3 §5.4/§5.6, A1 §4·§9 |
| Key Concepts | `getting-started/key-concepts` | 부분 수정 | 공통 개념 유지·번역, **부서·RBAC 정의만 추가** (가시성·ACL·소유권은 권한 설정 챕터 위임) | A1 §7.3, A3 §1·§5 |

> 신규 챕터 4종 = 대시보드 / 감사로그 / 권한 설정 / 사용자·부서 관리. Get Started는 이들의 **진입 티저**만 담당하고, 본문은 각 챕터가 책임진다.

---

# 2. Introduction — 처리 지침

## 2.1 본문 재작성

원본 첫 문단("Dify is an open-source platform…")을 spx-agent 소개로 교체. 톤은 "오픈소스 플랫폼" 강조 대신 **사내 에이전트·워크플로우 구축 플랫폼**으로. 전역 규칙 #5에 따라 "Dify" 브랜드·"Do It For You" 어원 `<Info>`(원본 line 32-34)는 **제거**.

권장 도입문(예):

> spx-agent는 에이전트형 워크플로우를 구축하는 플랫폼입니다. 프로세스를 시각적으로 정의하고, 기존 도구·데이터와 연결하여, 실제 업무 문제를 푸는 AI 애플리케이션을 배포할 수 있습니다.

## 2.2 CardGroup 재구성 — 외부 채널 제거 (전역 규칙 #5)

원본 6개 카드 중 외부 Dify 채널·범위 외 항목을 제거하고 신규 챕터 티저로 대체:

| 원본 카드 | 처리 |
|----------|------|
| Quick Start (`/quick-start`) | **유지** — 링크만 ko 경로로 |
| Concepts (`/key-concepts`) | **유지** — 링크만 ko 경로로 |
| Self Host (`/self-host/...`) | **제거** — Use Dify 범위 외 (Self-host 섹션 미포팅) |
| Forum (`forum.dify.ai`) | **제거** — Dify 공식 외부 채널 (전역 규칙 #5) |
| Changelog (`github.com/langgenius/...`) | **제거** — Dify 공식 GitHub (전역 규칙 #5) |
| Tutorials (`/tutorials/customer-service-bot`) | **유지** — 링크만 ko 경로로 |

→ 빈 자리는 **신규 챕터 키워드 티저 카드**로 채운다(§2.3). 결과적으로 "유지 3장 + 신규 4장" 구성.

## 2.3 신규 챕터 키워드 티저 (4종)

각 카드는 한 줄 설명 + 해당 챕터 링크. Introduction 위치가 `ko/use-spx-agent/getting-started/`이므로 상대경로는 `../<그룹>/<챕터>` 형태.

| 티저 카드 | 한 줄 설명(권장) | 링크(상대경로) | 근거 |
|----------|----------------|--------------|------|
| **권한 설정** | 앱·지식·도구의 가시성 범위와 접근 권한을 설정합니다 | `../workspace/permissions` | A1 §1·§3 |
| **사용자·부서 관리** | 부서를 만들고 멤버를 배정하며, 외부 시스템 그룹과 자동으로 연동합니다 | `../workspace/departments` | A3 §3·§5 |
| **대시보드** | 부서별 리소스·모델 토큰 사용량을 워크스페이스 단위로 조망합니다 | `../analytics-audit/dashboard` | 대시보드 분석본 |
| **감사로그** | 사용자 활동·관리 작업·보안 이벤트를 조회·필터·내보내기합니다 | `../analytics-audit/audit-log` | B1 §8·§10 |

> **표기 주의**(전역 규칙 #6): 사용자·부서 관리 티저에서 "Keycloak" 직접 노출 금지 — "외부 시스템" 표현 사용. A3 §5.6 톤.
>
> **티저 = 키워드 수준**: Introduction은 "이런 게 있다"만 알린다. 가시성 4단계·ACL·mirror 동기화 같은 메커니즘은 각 챕터 본문이 다룸 (점진적 공개, conventions 스타일 패턴).

## 2.4 이미지·frontmatter

- 원본 대표 이미지(line 9)는 Phase 5에서 spx-agent 화면으로 교체. Phase 4에선 placeholder 또는 원본 참조 유지.
- frontmatter: `mode: "wide"`는 Mintlify 전용 키 → Docusaurus 전환 시 제거(conventions/migration 기준). `icon`은 sidebars.js 관리이므로 본문 frontmatter에서 불필요. `title`만 유지.

---

# 3. Quick Start — 처리 지침

> 원본은 "30-Minute Quick Start" — Dify Cloud 가입부터 시작하는 멀티플랫폼 콘텐츠 생성기 워크플로우 튜토리얼. 워크플로우 빌드 본문(Step 1~4)은 **교육용 핵심 자료라 유지·번역**하되, 진입부("Before You Start")와 환경 의존 표현을 spx-agent 기준으로 정리한다.

## 3.1 "Before You Start" — 전역 규칙 적용 (가장 큰 변경)

원본 3 Step(Cloud 가입 / 모델 제공자 / 기본 모델)을 다음과 같이 정리:

| 원본 Step | 처리 | 사유 |
|----------|------|------|
| **Sign in to Dify Cloud** (Sandbox plan, 200 AI credits) | **로그인 절차로 교체** — §3.2 | 전역 규칙 #1·#3 (Cloud/SaaS·Sandbox·credits 제거) |
| **Set Up the Model Provider** (OpenAI 플러그인 설치) | **부분 수정** — §3.3 | 전역 규칙 #1 (Marketplace/플러그인 제거 → "설치" 표현 조정) |
| **Configure the Default Model** | **유지·번역** | spx-agent 동일 (Settings → Model Provider → Default Model) |

## 3.2 로그인 = 사용자명/비밀번호 폼 (+ 부서 자동 배정 한 줄)

scope-mapping 비고 정전: **현재 개발된 화면 기준 로그인은 사용자명/비밀번호 폼**이며 KC SSO 리다이렉트가 아니다. 단, 로그인 시점에 외부 시스템 그룹 → 부서 자동 동기화가 일어난다(A3 §5.4).

권장 본문(예):

> spx-agent에 접속하여 부여받은 계정으로 로그인합니다. 로그인하면 워크스페이스로 이동합니다.
>
> <Info>
> SSO로 로그인하는 경우, 로그인 시점에 외부 시스템(예, Keycloak)의 그룹에 따라 소속 부서가 자동으로 배정됩니다. 자세한 내용은 [사용자·부서 관리](../workspace/departments)를 참고하세요.
> </Info>

- **인용**: A3 §5.4(동기화 시점 — 로그인 콜백), §5.6.1·§5.6.5(추상화 표기). "Keycloak"은 §5.6 톤으로 **첫 등장 1회 "외부 시스템(예, Keycloak)"** 후 "외부 시스템".
- **주의**: 사용자명/비밀번호 폼과 SSO는 systemFeatures 플래그에 따라 공존 가능(A4 §3.2, 로그인 옵션 4종). 따라서 "로그인은 무조건 폼"이라고 단정하지 말고, **기본 화면은 폼**으로 안내하고 SSO 환경은 `<Info>`로 분기. 5/29 매트릭스의 "KC SSO 아님" 표기는 *디폴트 화면* 기준이라는 점을 본문에서 단정형으로 굳히지 않는다.

## 3.3 모델 제공자 — "플러그인 설치" 표현 조정 (전역 규칙 #1)

원본은 "OpenAI 플러그인을 설치(install the OpenAI plugin)"하고 Sandbox credits로 API 키 없이 사용. spx-agent는 Marketplace/플러그인 관리가 제거(결정 1)되었으므로:

- "플러그인 설치" → **"모델 제공자 설정"**으로 표현 전환. (모델 제공자 페이지 자체는 유지·번역 — scope-mapping workspace/model-providers)
- Sandbox credits·"no API key required" 문장 **제거** → spx-agent는 관리자가 구성한 모델 제공자/API 키를 사용한다는 톤.
- 예시 모델명(`gpt-5.2`)은 사내에서 접근 가능한 모델로 일반화하거나 자리표시. 본격 작성 시 실제 사용 가능 모델 확인 후 확정(deferred).

> ⚠️ Quick Start 본문 Step 2(워크플로우 빌드) 곳곳에 `gpt-5.2` 모델명이 박혀 있음. 모델명 일괄 치환 정책은 본 문서 범위 밖 — Tutorials AI Image Generation 처리(scope-mapping)와 동일하게 "접근 가능한 모델로 적용" 원칙 따름.

## 3.4 앱 생성 단계 — 부서·권한 한 줄

원본 "Step 1: Create a New Workflow"(Studio → Create from blank → Workflow)에 spx-agent 차이를 **한 줄 + 링크**로 추가. 가시성·ACL·소유권 메커니즘은 권한 설정 챕터가 책임지므로 여기선 **포인터만**.

권장 본문(예, Step 1 말미):

> <Note>
> 앱을 만들면 생성자가 소유자가 되고, 소속 부서가 소유 부서로 지정됩니다. 다른 부서·멤버에게 공개 범위나 권한을 부여하려면 [권한 설정](../workspace/permissions)을 참고하세요.
> </Note>

- **인용**: A1 §3(도메인 모델 — 생성자=소유자), §4(가시성 — 부서 공개 시 소유 부서), §6(소유권). A3 §6(부서 이동 시 권한 변화)은 본 한 줄에는 불필요(과함).
- **범위 경계**: Get Started에서 가시성 4단계·ACL 부여 절차를 설명하지 **않는다**. scope-mapping 비고 "한 줄 언급" 준수.

## 3.5 Cloud/SaaS 잔여 표현 정리 (전역 규칙 #1~#3)

본문 전체 grep 대상:
- "Dify Cloud", "Sandbox plan", "AI credits", "cloud.dify.ai" → 제거/CE 단일 서술
- "Publish & Share"(Step 4) — 게시 자체는 유지. Marketplace 게시 언급 있으면 제거(결정 1).
- 외부 링크(Jinja2 docs `jinja.palletsprojects.com`)는 기술 문서 링크라 유지 가능(전역 규칙 #5는 Dify 채널 한정).

---

# 4. Key Concepts — 처리 지침

## 4.1 유지·번역 개념 (rebranding만)

원본 개념 블록은 그대로 번역하되 "Dify" 브랜드만 정리:

| 원본 개념 | 처리 | 비고 |
|----------|------|------|
| Dify App | 유지·번역 → "앱" | "Dify App" → "앱". Studio에서 워크플로우·챗플로우 구축. MCP 서버 링크 **유지**(결정 3, ko 경로로) |
| Workflow | 유지·번역 | User Input / Trigger 시작 노드 설명 유지 |
| Chatflow | 유지·번역 | |
| Dify DSL | 유지·번역 → "DSL" | "Dify's own DSL" → "자체 DSL". 앱 이식·공유. (App Management의 SaaS/Community 분기는 A4 위임 — Key Concepts에선 단순 개념만) |
| Variables | 유지·번역 | 입력/출력/환경 변수/대화 변수. `sys.*` 표는 그대로 |
| Variable Referencing | 유지·번역 | |

> 글로서리 매핑: 워크플로우/챗플로우/DSL/변수 → conventions 핵심 개념·노드 표 기존 항목 사용. 신규 용어 없음.

## 4.2 추가: 부서(Department) + RBAC 정의만

scope-mapping 정전: **부서·RBAC 개념만 추가**. 소유권·가시성·ACL은 신규 "권한 설정" 챕터에서 다루므로 Key Concepts에서 **정의하지 않고 한 줄 링크만**.

권장 신규 절(예, 개념 블록 말미에 H3 2개 추가):

### 부서 (Department)

> spx-agent는 워크스페이스 멤버를 **부서**로 묶어 관리합니다. 부서는 권한 부여의 단위가 되어, 부서 단위로 앱·지식·도구 접근을 제어할 수 있습니다. 한 멤버는 여러 부서에 속할 수 있으며, 외부 시스템(예, Keycloak)의 그룹과 자동으로 연동됩니다. 자세한 내용은 [사용자·부서 관리](../workspace/departments)를 참고하세요.

- **인용**: A3 §1(부서가 권한 단위), §5(외부 시스템 동기화 — §5.6 추상화 톤), A1 §2 C1(다중 멤버십 허용).

### RBAC (역할 기반 접근 제어)

> 워크스페이스 멤버에게는 역할이 부여됩니다 — **소유자 / 관리자 / 편집자 / 일반 멤버 / 지식 관리자**. 역할은 앱·지식·도구에 대한 기본 권한(조회·실행·편집·소유권 이전 등)을 결정합니다. 역할 위에 부서·개별 권한 부여가 더해지는 방식은 [권한 설정](../workspace/permissions)에서 다룹니다.

- **인용**: A1 §7.3(역할별 capability 매트릭스 — 5종 역할). 매트릭스 표 자체는 권한 설정 챕터로 위임하고, Key Concepts에선 **역할 5종 나열 + 한 줄 의미**만.
- 글로서리: 소유자/관리자/편집자/일반 멤버/지식 관리자 = conventions "워크스페이스 역할" 표 기존 항목. RBAC = 약어 유지.

## 4.3 위임 경계 (넣지 말 것)

| 개념 | 위치 | Key Concepts 처리 |
|------|------|------------------|
| 가시성 4단계(private/department/custom/workspace) | 권한 설정 챕터 | 정의 X — 링크만 |
| ACL 부여/취소 | 권한 설정 챕터 | 정의 X |
| 소유권·소유권 이전 | 권한 설정 챕터 | 정의 X |
| mirror 동기화 동작 | 사용자·부서 관리 챕터 | 정의 X — "자동 연동" 한 줄만 |

→ A1 분석본 §1.1 결정 원칙 일관: "부서·RBAC는 Key Concepts, 소유권/가시성/ACL은 권한 설정 챕터".

---

# 5. 전역 규칙·결정 적용 체크리스트 (3p 공통)

| 규칙/결정 | Introduction | Quick Start | Key Concepts |
|----------|:---:|:---:|:---:|
| #1 Cloud/SaaS 플랜 제거 | — | ✅ (Sandbox·credits) | — |
| #1 Marketplace/플러그인 제거 (결정 1) | — | ✅ (플러그인 설치 표현) | ✅ (앱 타입 설명 내 플러그인 언급 점검) |
| #3 MCP 유지 (결정 3) | — | — | ✅ (MCP 서버 링크 유지) |
| #5 Dify 브랜드·외부 채널 제거 | ✅ (CardGroup·어원 Info) | ✅ (Dify Cloud 링크) | ✅ ("Dify App/DSL" 명칭) |
| #6 KC 추상화 (결정 5) | ✅ (부서 티저) | ✅ (로그인 부서 자동 배정) | ✅ (부서 개념) |

> #2(외부 연결 제거)·#4(자체 호스팅 분기)는 Get Started 3p에 해당 본문 없음.

---

# 6. 인용 매핑 — A1·A3가 어디에 쓰이나

| 인용원 | 인용 섹션 | 사용 위치 (본 3p) |
|--------|----------|------------------|
| A1 [[references/spx-app-permissions-analysis]] | §3 도메인 모델·§4 가시성·§6 소유권 | Quick Start §3.4 앱 생성 한 줄 |
| A1 | §7.3 역할 매트릭스(5종) | Key Concepts §4.2 RBAC 정의 (역할 나열) |
| A1 | §2 C1 다중 멤버십 | Key Concepts §4.2 부서 정의 ("여러 부서에 속할 수 있음") |
| A3 [[references/spx-departments-management]] | §5.4 동기화 시점(로그인 콜백) | Quick Start §3.2 로그인 부서 자동 배정 |
| A3 | §5.6 추상화 표기 예시 | Introduction §2.3 티저·Quick Start §3.2·Key Concepts §4.2 (KC → "외부 시스템") |
| A3 | §1 부서=권한 단위 | Key Concepts §4.2 부서 정의 |
| 대시보드 [[references/spx-dashboard-analysis]] | (워크스페이스 KPI 요지) | Introduction §2.3 대시보드 티저 |
| B1 [[references/spx-audit-log-analysis]] | §8 영역 구분·§10 라벨 | Introduction §2.3 감사로그 티저 |

> A3 §10에서 C1이 §5.4·§5.6을 인용한다고 이미 명시(역방향 일치 확인).

---

# 7. 챕터 본문 골격 (선택적 인터리브)

> 클러스터 C는 후행이라 인터리브 1p는 **선택**. 본격 작성 시 아래 골격대로 3p를 작성하면 됨. 경로: `ko/use-spx-agent/getting-started/{introduction,quick-start,key-concepts}.mdx` (원본 fork 시 이미 존재 — 부분 수정).

## 7.1 introduction.mdx
```
# 소개 (title)
- spx-agent 한 문단 소개 (오픈소스 강조 X, 사내 플랫폼 톤)
- CardGroup: Quick Start / 개념 / 튜토리얼 (유지 3)
                + 권한 설정 / 사용자·부서 관리 / 대시보드 / 감사로그 (신규 티저 4)
- (Dify 어원 Info 제거, Self Host/Forum/Changelog 카드 제거)
```

## 7.2 quick-start.mdx
```
# 빠른 시작 (title)
## 시작하기 전에
- 로그인 (사용자명/비밀번호 폼; SSO 시 부서 자동 배정 Info)
- 모델 제공자 설정 (플러그인 설치 표현 X)
- 기본 모델 설정
## Step 1 앱 생성  ← 소유자·소유 부서 Note + 권한 설정 링크
## Step 2~4  워크플로우 빌드 (유지·번역, 모델명 일반화)
```

## 7.3 key-concepts.mdx
```
# 핵심 개념 (title)
## 앱 / 워크플로우 / 챗플로우 / DSL / 변수 / 변수 참조  (유지·번역)
## 부서 (Department)   ← 신규, A3 인용, 외부 시스템 추상화
## RBAC (역할 기반 접근 제어)  ← 신규, A1 §7.3 역할 5종 나열
(가시성·ACL·소유권은 권한 설정 챕터 링크만)
```

---

# 8. 후속 체크리스트

## 8.1 본 분석 (2026-06-04)
- [x] 원본 3p grep + 전역 규칙·결정 적용 지점 식별
- [x] A1·A3 인용 위치 확정 (역방향 A3 §10 일치 확인)
- [x] 신규 챕터 4종 티저 한 줄·링크·근거 박제
- [x] Introduction CardGroup 6장 → 3 유지 + 4 신규 재구성 표
- [x] Quick Start 로그인(폼 + SSO 부서 자동 배정 분기)·모델 제공자·앱 생성 한 줄 박제
- [x] Key Concepts 부서·RBAC 정의 권장문 + 위임 경계 박제

## 8.2 인터리브 챕터 작성 (2026-06-04 완료)
- [x] `ko/use-spx-agent/getting-started/introduction.mdx` — 소개 재작성 + 시작 카드 3 + 관리 기능 티저 카드 4 (CardGroup 2개). 외부 채널 제거
- [x] `ko/use-spx-agent/getting-started/quick-start.mdx` — 시작하기 전에(로그인+SSO 부서 자동 배정, 모델 제공자, 기본 모델) + 1단계 앱 생성(소유자·소유 부서 Note). 2단계 워크플로우 빌드는 유지·번역 요약(P2)
- [x] `ko/use-spx-agent/getting-started/key-concepts.mdx` — 앱·워크플로우·챗플로우·DSL·변수·변수 참조 번역 + 부서·RBAC 정의(역할 5종 표). 가시성·ACL·소유권은 권한 설정 챕터 위임
- [x] `sidebars.js` "시작하기" 카테고리 신설(최상단) + 3p 등록
- [x] 빌드 통과 — 깨진 링크는 미작성 페이지(`publish/publish-mcp`·`nodes/*`·`tutorials/*`)만, 신규 3p의 워크스페이스·통계·감사 링크는 정상 해소

## 8.3 Phase 4 본격 작성 시 (deferred)
- [ ] Quick Start 2단계 워크플로우 빌드 본문 전체 번역 (현재 노드 구성 요약만)
- [ ] Quick Start `gpt-5.2` 모델명 일괄 치환 정책 확정 (접근 가능 모델 확인 후)
- [ ] 로그인 화면 디폴트(폼) vs SSO 분기를 systemFeatures 실제 플래그로 재확인 (A4 §3.2 연동)
- [ ] 신규 챕터(nodes·publish/mcp·tutorials) 작성 후 티저·개념 링크 anchor 최종 검증
- [ ] introduction 대표 이미지 spx-agent 화면 교체 (Phase 5)
