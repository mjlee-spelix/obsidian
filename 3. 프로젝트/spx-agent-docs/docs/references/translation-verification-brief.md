# 번역 검증 브리프 (레이어2 에이전트 공통 기준)

> ko/ 번역 문서 검증용. 각 검증 에이전트는 본 브리프 + (원본 en + 번역본 ko) + 도메인 i18n을 대조해 findings를 반환한다.
> 원리 한 줄: **i18n(도메인에 맞는 파일)이 권위, 글로서리는 1차 조회용 캐시**. 충돌 시 i18n 승.

## 검증 4기준

1. **의미 일치 (①)** — 원문 대비 의미 변질·왜곡 없음 (원본 있는 파일만)
2. **자연스러움 (②)** — 번역본만 통독했을 때 문장 연결·흐름이 자연스러움 (전 파일)
3. **규칙 준수 (③)** — 어미·용어·링크·MDX·frontmatter 규칙 (전 파일)
4. **정보 누락/추가 (④)** — 표·콜아웃·코드블록·이미지 통째 누락 or 무단 추가 (원본 있는 파일은 대조, 신규는 내적 일관성)

## 심각도

- **상**: 의미 변질·잘못된 사실·정보(표/콜아웃/코드블록) 통째 누락 → 반드시 수정
- **중**: 용어 권위 라벨 위반(i18n 확인됨)·어미 정책 위반·broken link → 수정 권장
- **하**: frontmatter·헤딩 레벨·문체 개선 여지·grep 후보(맥락 판단 필요) → 확인

## 용어 권위 규칙 (가장 중요 — 오탐 방지)

우선순위: **1) 해당 도메인 i18n 파일 > 2) dify-audit audit-meta.ts(감사로그) > 3) spx 자체 추가 용어 > 4) 글로서리(캐시) > 5) writing-guides 표준번역**.

**중/상 등급 용어 findings는 보고 전 반드시 "해당 파일 도메인의 i18n"으로 재확인**한다. 글로서리만 보고 위반 판정 금지 (글로서리가 엉뚱한 도메인 라벨을 캐시했을 수 있음).

⚠️ **실제 오탐 사례**: 노드 문서의 "재정렬 모델"을 글로서리(L358, 지식 설정 도메인 인용)만 보면 "재순위로 고쳐야 함"으로 오판. 그러나 노드 도메인 권위는 `workflow.json` = "재정렬 모델"이라 **정상**. 같은 영어 단어도 i18n 파일마다 번역이 다름(재순위/재랭크/재정렬). 반드시 도메인 파일 확인.

### i18n 파일 위치

```
C:\Users\Administrator\Projects\spx-agent\web\i18n\ko-KR\*.json
```
grep 예: `grep -niE "rerank|재순위|재정렬" web/i18n/ko-KR/workflow.json`

### 챕터 도메인 → i18n 파일 매핑

| ko 경로 | 우선 i18n 파일 |
|---|---|
| getting-started/ | common.json, login.json, layout.json |
| nodes/ (워크플로 노드) | **workflow.json** (Tools 노드는 tools.json) |
| build/ | workflow.json |
| debug/ | workflow.json (debug.* 키) |
| publish/ | share.json, app.json, app-api.json |
| monitor/ | app-debug.json, app-log.json, app-annotation.json, app-overview.json |
| knowledge/ | dataset.json, dataset-creation.json, dataset-documents.json, dataset-hit-testing.json, dataset-settings.json, pipeline.json, dataset-pipeline.json |
| workspace/ | common.json, layout.json, oauth.json (멤버·역할: common.json members.*) |
| analytics-audit/audit-log (신규) | dify-audit `src/lib/audit-meta.ts` (별도 앱) |
| analytics-audit/dashboard (신규) | layout.json + spx 자체 추가 키 |
| workspace/permissions·departments (신규) | common.json, layout.json + spx 자체 추가 키 |

## Distilled 글로서리 (1차 조회 — 충돌 시 i18n 승)

### 핵심 개념
Workflow=워크플로우 · Chatflow=채팅 플로우(~~챗플로우~~) · Agent=에이전트 · knowledge base=지식("지식베이스"X) · workspace=워크스페이스 · WebApp=웹앱 · Multimodal=멀티모달 · Streaming=스트리밍 · token=토큰 · prompt=프롬프트 · LLM=LLM(유지)

### 워크플로 노드 (workflow.json 검증됨)
사용자 입력(User Input) · LLM · 지식 검색(Knowledge Retrieval) · 답변(Answer) · 출력(Output) · 에이전트(Agent) · 질문 분류기(Question Classifier) · 조건 분기(IF/ELSE, UI는 영어) · 코드 실행(Code) · 템플릿 변환(Template) · HTTP 요청 · 변수 집계자(Variable Aggregator, ~~변수 집약기~~) · 변수 할당자(Variable Assigner, ~~변수 할당기~~) · 반복(Iteration) · 루프(Loop) · 매개변수 추출기(Parameter Extractor, ~~파라미터 추출기~~) · Doc 추출기(~~문서 추출기~~) · List 연산자(~~리스트 처리~~) · 사람 입력(Human Input, ~~사람 개입~~) · 일정 트리거(Schedule, ~~예약 트리거~~) · 웹훅 트리거 · 도구(Tool)

### 지식·검색
chunk=청크("세그먼트"X) · retrieval=검색 · indexing=인덱싱 · 벡터 검색 · 전체 텍스트 검색(~~전문 검색~~) · 하이브리드 검색 · Top K(유지) · 점수 임계값 · 검색 테스트(Retrieval Testing) · 인덱스 방법 · 검색 방법 · 청크 중첩 · 일반/부모-자식/Q&A 모드 · 고품질/경제적 · 역인덱스 · 지식 파이프라인 · 오케스트레이션 · 데이터 소스 · 아카이브/아카이브 해제
> ⚠️ Rerank: 도메인별 다름 — 노드(workflow.json)="재정렬", 지식설정(dataset-settings)="재순위", 모델제공자(common)="재랭크". **도메인 i18n 확인 필수**

### UI 라벨·탭
변수 검사(Variable Inspector, ~~변수 인스펙터~~) · 실행 기록(Run History, ~~실행 이력~~) · 모두 초기화(~~전체 초기화~~) · 오케스트레이트(~~편성~~) · API 액세스(~~API 접근~~) · 모니터링 · 조회(Hit, ~~적중~~) · 일치(Match, ~~매칭~~) · 점수 임계값(~~유사도 임계값~~) · 게시 · 미리보기 · 테스트 실행

### 워크스페이스 역할 (common.json members.*)
소유자(Owner) · 관리자(Admin) · 편집자(Editor) · 빌더(Builder) · 일반(Normal, ~~일반 멤버~~) · 지식 관리자(datasetOperator)

### spx 전용 (코드에 한글 키 없음 → 글로서리 권위)
대시보드(워크스페이스 단위, 앱"모니터링"과 구분) · 부서(Department) · 가시성 범위(비공개/부서 공개/사용자 지정/워크스페이스 공개) · ACL(접근 제어 목록) · 주체(Principal) · 감사 로그 · RBAC/SSO(유지) · Keycloak(본문은 "외부 시스템(예, Keycloak)"으로 추상화)

### 가시성 4단계 / 액션 8종 / 감사 카테고리
가시성: private=비공개, department=부서 공개, custom=사용자 지정, workspace=워크스페이스 공개
액션: view=조회, edit=편집, delete=삭제, execute=실행, publish=발행, duplicate=복제, manage_permission=권한 관리, transfer=소유권 이전
감사 카테고리: admin=관리자, user=사용자, security=보안. 행위자: account=관리자, end_user=앱 사용자, api=API, system=시스템
> rate_limit_exceeded 라벨은 audit-meta.ts로 확인 (글로서리 캐시는 "Rate Limit 초과")

## 문체·규칙 체크 (③)

- **어미(합쇼체)**: `~합니다`/`~할 수 있습니다`/`~해야 합니다`. 행동 안내 문장 표준 어미 **`~하시기 바랍니다`** (`~하세요`·단독 `~하십시오` 금지)
- **영문 병기**: 전문 용어 첫 등장 시 `한국어(English)`, 이후 한국어만
- **frontmatter**: `title` 필수. `description`은 **원본 en에 있을 때만** (en에 없는데 ko에 있으면 위반)
- **헤딩**: H2→H3→H4 (H2 없이 H3 시작은 하-등급 지적). 한국어 Title Case 불필요
- **링크**: 마크다운 링크=상대경로 / `<Card href>`=절대경로. `/readme`·`/index` 슬러그 금지 (`.mdx` 포함 `readme.mdx#…`도 broken 후보)
- **코드블록**: 언어 태그 필수. **이중 빈 줄 금지**

### 직역체 마커 (②③)
`~을 위한 N`(→~하는 N) · 무생물 주어(이 기능은~제공) · noun-of-noun(~의 ~의) · `~위에서 동작`(→~으로) · `~로부터 생성`(→~로) · `~를 통해` 남용(→~로/생략) · 괄호 한자어 동격 · 피동 남용(~되어집니다/~에 의해) · `~할 수 있게 해주는 N`(→~하는 N)

### ⭐ 전역 규칙 (decisions.md 2026-06-02) — ④ 판정의 전제. 누락 ≠ 위반
아래에 해당하면 **누락은 의도적이며 위반 아님(오탐)**:
1. **Dify Cloud/SaaS 플랜**(Free/Pro/Team/Enterprise·Subscription·Billing·Seat·Quota) → 제거
2. **"Cloud version에서는…" 분기 콜아웃** → CE 단일 서술로 (분기 자체 제거)
3. **Sandbox/Production 워크스페이스 분리(SaaS)** → 제거 (단 `dify-sandbox` 보안 컨테이너는 보존)
4. **"자체 호스팅 한정" 분기 + 환경변수/배포 설정** → 분기 제거 + **env var 본문은 `references/deployment-config-extracts.md`로 추출**. 즉 env var가 본문에서 빠진 건 **정상**
5. **외부 Dify 리소스**(GitHub langgenius/*·community/forum/Discord·Marketplace) + 기능 확장 안내 → 제거 + `references/feature-extension-extracts.md`로 추출
6. **이미지/스크린샷**: Dify 원본 캡처는 **Phase 5에서 spx 화면으로 교체** → Phase 2~4 시점 누락/placeholder는 **정상(deferred)**, ④ 위반 아님
7. **플러그인 트리거 노드**: 결정 1로 **삭제 대상** → 관련 서술 누락은 정상

### 정보 누락 판단 (④) — 절차
1. 위 **전역 규칙 1~7에 해당하면 위반 아님(오탐)**.
2. env var/배포 설정이면 `deployment-config-extracts.md`에 **이미 추출됐는지 grep 확인**: 추출됨 → 🟢 정상 / 미추출 → 🟠 "추출 갭"(본문 수정 아님, **추출 파일에 추가** 권고. severity=중)
3. 어디에도 안 걸리는데 빠진 것 = **진짜 누락**: 표 데이터·연산자 예시·코드블록·env 아닌 동작 제약·문장 의미 → ④ 위반
4. 판단 어려우면 "하(검토)"로 낮추고 사람 판단에 맡김

## 출력 형식

findings 배열. 각 항목: `criterion(①②③④) / severity(상중하) / location(라인/섹션) / issue / authority_check(i18n 확인 결과) / recommendation`. 위반 없으면 빈 배열 + "이상 없음".
