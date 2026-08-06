# 주요 결정 기록

> 형식: 시간순. 결정 사유와 대안도 같이 적어서 추후 재논의 시 base 확보.

## 2026-05-29 — 프로젝트 출범

### 소스 리포 채택
- **결정**: `langgenius/dify-docs` (CC BY 4.0)
- **사유**: Dify 공식 docs 원본, 자유 수정·배포 가능, Mintlify 빌드 호환
- **대안 검토**: 자체 작성 → 비용 큼·일관성 ↓·이사님 지시 ("dify docs 포팅") 미충족

### 범위 — Use Dify만
- **결정**: Use Dify 섹션만 포팅, Getting Started/Self-host/Plugin Dev/API Reference 제외
- **사유**: 5/29 이사님 지시 명시
- **영향**: docs.json 네비게이션 대폭 축소 예정

### 언어 방향
- **결정**: 영어 → 한국어 자연스럽게 (직역 X)
- **사유**: 5/29 이사님 지시 명시
- **구현**: `ko/` 폴더 신설 + 수동 번역으로 시작 (Phase 2 파일럿), 챕터 30개 초과 시 자동화 파이프라인 검토

### 차감/추가 원칙
- **차감**: spx-agent에 없는 기능 (Marketplace·Team 멤버 초대 등)
- **추가**: spx-agent 추가 기능 (대시보드·감사로그·RBAC·KC SSO 등)
- 상세 매핑: [[scope-mapping]]

### 작업 위치
- **결정**: 코드(`Projects/spx-agent-docs/`) + 메모(옵시디언 `3. 프로젝트/spx-agent-docs/`) 분리
- **사유**: 옵시디언이 MDX 대량 인덱싱하면 무거움 + 코드는 git, 메모는 옵시디언이 자연
- **연결**: 코드 폴더에 `.claude` junction → 옵시디언 폴더 미러

### 빌드 도구
- **결정**: 우선 **Mintlify CLI**로 시작 (`mintlify dev` 로컬 미리보기)
- **사유**: 원본이 Mintlify 기반, 변환 비용 0
- **호스팅 결정 보류**: Phase 3 이사님 컨펌 시 (Mintlify Cloud / Docusaurus 변환 후 사내 nginx / mintlify dev 공유 중 택1)

### 사내 git 푸시 시점
- **결정**: Phase 1~2 (탐색·파일럿) 동안은 로컬만, Phase 3 컨펌 후 사내 git 푸시
- **사유**: 형식·범위 흔들리는 단계에서 사내 git에 올리면 노이즈, 컨펌 후 본격화

### Junction 처리
- **결정**: `Projects/spx-agent-docs/.claude` junction → 옵시디언 `3. 프로젝트/spx-agent-docs/` 로 연결
- **충돌 처리**: 원본 fork에 이미 `.claude/skills/` (Dify 작성 가이드) 있음 → 옵시디언 폴더로 이관 통합 결정
- **최종 구조**: 옵시디언 폴더 = `README.md` / `docs/` (우리 메모) / `skills/` (Dify 보존) → junction으로 코드 폴더 `.claude/`에 미러
- **임시 wrapper 제거**: `.claude-dify-original/` 폴더는 skills 이관 후 삭제 (잔존 `CLAUDE.md` 1개만 코드 루트로 떨어짐)

### 베이스라인 커밋 — Dify 1.13.3 정합
- **결정**: 옵션 B — dify-docs `5c1c3a4c` (2026-03-30) 커밋을 baseline으로 fork
- **사유**:
  - spx-agent는 Dify 1.13.3 고정 사용 → docs도 해당 버전에 정합 필요
  - dify-docs는 패치 버전(1.13.1~1.13.3) 별 태그 없음, 마이너 사이클(v1.13.0, v1.14.0)만 태깅
  - `5c1c3a4c` 커밋 메시지에 명시: "1.13.0~1.13.3 변경 일괄 sync" → 정확한 1.13.3 docs 상태 보장
- **대안 검토**:
  - 옵션 A (v1.13.0 태그): 1.13.1~1.13.3 변경 누락
  - 옵션 D (v1.14.0 태그): 1.14 신기능 docs 섞임 → 차감 작업 ↑
- **upstream sync 포기**: 1.13.3 고정이라 신규 patch/minor 변경 받아올 필요 X

### 히스토리 정책 — orphan
- **결정**: `5c1c3a4c` 시점 워킹트리를 `--orphan` 브랜치로 만들어 단일 root 커밋으로 시작
- **사유**:
  - 1.13.3 고정 = upstream sync 포기 = 원본 히스토리 보존 가치 약함
  - GitLens 노이즈 제거, 사내 git 푸시 시 경량화
  - 라이선스 의무는 NOTICE.md + LICENSE 파일로 충족 (히스토리 의존 X)
- **출처 추적**: 본 결정 + NOTICE.md에 upstream 커밋 SHA(`5c1c3a4c`) 명시로 사슬 보존
- **upstream 리모트**: 정보 참조용으로 유지 (가끔 특정 커밋 cherry-pick 가능성)

### 백업
- **결정**: 사내 git 백업 push 안 함 (사용자 결정 — 로컬 작업만)
- **사유**: 탐색 단계라 사내 git에 일단 안 올림. Phase 3 컨펌 시점에 함께 결정

### CLAUDE.md 배치 — 원본 보존 + fork overrides 분리
- **결정**:
  - 루트 `CLAUDE.md` = **원본 Dify 컨트리뷰터 가이드 그대로** (CC BY 4.0, git tracked, 수정 금지)
  - `.claude/CLAUDE.md` = **spx fork overrides** (옵시디언 junction, gitignored, 원본 일부 규칙 덮어씀)
- **사유**:
  - 원본 가이드(MDX 규칙·writing-guides·스킬 패턴 등)는 fork에서도 그대로 유효
  - 원본 파일 미수정 = CC BY 4.0 출처 명확성 ↑, 라이선스 의무 단순화
  - fork 한정 컨텍스트는 옵시디언 측에서 관리 (편집 자유도 ↑)
- **대안 검토**:
  - (a) 루트 CLAUDE.md 전체 교체 → 원본 가치 손실, 폐기 (시도 후 revert)
  - (b) 루트 CLAUDE.md 상단에 overrides 헤더 prepend → 원본·우리 콘텐츠 혼재, 폐기 (시도 후 revert)
  - (c) 루트 원본 + `.claude/CLAUDE.md` 분리 ← **채택**
- **주의 — `.claude/CLAUDE.md` 자동 로드 불확실**:
  - Claude Code가 `.claude/CLAUDE.md`를 세션 시작 시 자동 로드하는지 공식 보장 없음
  - 세션 시작 시 명시적 로드 권장: `"루트 CLAUDE.md + .claude/CLAUDE.md + .claude/README.md 읽어"`
  - 실제 동작 확인 후 패턴 재조정 가능
- **시도 흔적**: git log에 `d50f368c` (교체) + `02be15fe` (prepend) + `9c25ea90` (revert) 3건 — 추후 squash로 정리 가능 (현재는 시행착오 보존 목적으로 유지)

## 2026-05-29 — spx-agent 분석 후 신규 챕터 재구성

### KC SSO 독립 챕터 삭제
- **결정**: Keycloak SSO 로그인을 독립 챕터로 만들지 않음
- **사유**: 사용자 입장에서 로그인은 단순 동작. Get Started/Quick Start에서 간단히 설명하면 충분
- **대안 검토**: 독립 챕터 → 내용이 얇아 페이지 가치 낮음
- **KC 관련 내용 배치**: Quick Start(로그인 흐름), 사용자/부서 관리(그룹 동기화)에 분산

### RBAC → 권한 설정 + 사용자/부서 관리 분리
- **결정**: RBAC 통합 챕터 대신 사용자 관점으로 2개 분리
  - **권한 설정** (5/29 시점 명칭 "앱 권한 설정", 6/4 rename): "내 자원(앱/지식/도구)의 권한을 어떻게 설정하나?" (가시성, ACL, 소유권)
  - **사용자/부서 관리**: "부서를 어떻게 만들고 멤버를 배정하나?" (부서 CRUD, KC 그룹 연동)
- **사유**: 
  - "RBAC"는 개발자 용어 — 사용자 매뉴얼에서는 기능 중심 명명이 자연스러움
  - spx-agent 코드 분석 결과 `app-permissions/`, `dataset-permissions/`, `tool-permissions/` 컴포넌트가 별도 존재 → 앱 상세 내 permissions 탭으로 접근
  - 부서 관리는 workspace 설정 내 별도 메뉴
- **코드 근거**: `web/app/components/app-permissions/`, `services/rbac/department_service.py`

### spx-agent 코드베이스 분석 완료
- **분석 범위**: 최상위 구조, RBAC 서비스 계층, 프론트엔드 컴포넌트, dify-audit, Keycloak 통합, 배포 구조
- **핵심 발견**:
  - 5개 사이드카 테이블 (`spx_*`) — 부서, 멤버십, 소유권, ACL, 감사로그
  - 권한 판정: Dify role (상위 게이트) + 커스텀 ACL (추가 레이어)
  - 가시성 4단계: private / department / custom / workspace
  - dify-audit: 독립 Next.js 서비스, 13 collectors, PostgreSQL 트리거
  - 관리자 대시보드: KPI, 부서별 리소스, 모델 토큰, 드릴다운
- **산출물**: [[scope-mapping]] 신규 챕터 v2, 코드 매핑 표 추가

## 2026-05-29 — Phase 2 진입 결정 6건

> 출처: `phase2-prereqs.md` (병합 후 삭제). 본 결정사항이 Phase 2(파일럿 챕터)·Phase 4(본격 포팅) 작업 기준.

### writing-guides 처리 — 하이브리드 (영역별 분리)

- **결정**: 원본 `writing-guides/` 3개 파일을 영역별로 분기 처리
  - `formatting-guide.md` (MDX·Mintlify 컴포넌트·헤딩) → **참조만**: `conventions.md`에 "MDX 포맷은 원본 그대로 따름" 한 줄로 위임 (언어 무관, 그대로 유효)
  - `style-guide.md` (문체·톤·콜아웃) → **우리가 다시 작성**: 영어 문체 규칙은 한국어 적용 불가. 톤·존댓말 정책은 Phase 2 파일럿 후 확정. 단 콜아웃(`<Note>`/`<Warning>`/`<Tip>`) 사용 패턴 같은 언어 무관 부분은 참조
  - `glossary.md` (en/zh/ja 표준 용어) → **우리 한국어 글로서리 별도 + 영문 원본은 참조용**: Dify UI 표기 정합 잡을 때 참조
- **사유**: 영역마다 한국어 적용성 다름. 통째로 복붙은 헷갈리고, 통째로 무시는 일관성 손실
- **대안 검토**:
  - (a) writing-guides 참조만 — 형식은 OK인데 톤·용어 영역 공백
  - (b) 우리한테 맞춰 전면 재작성 — 형식 가이드 중복 작성, 노동 ↑
  - (c) 하이브리드 ← **채택**

### `ko/` 폴더 + `docs.json` 한국어 섹션 — 골격 + 챕터별 점진

- **결정**: `ko/` 디렉토리 + `docs.json` ko 섹션을 **파일럿 챕터부터 점진적으로 추가**
  - `ko/use-dify/` 골격만 먼저 생성 + 파일럿 챕터 1~2개 작성
  - `docs.json`에 ko 언어 섹션 신설 + nav는 작성된 페이지만 등록
  - `mintlify dev`에서 언어 전환 UI로 한국어 렌더링 검증
- **사유**: 풀 미러링(102p placeholder)은 노동·UX 비용 큼. 점진식이 차단 없이 가볍게 시작 + 회복 비용 낮음
- **재판단 트리거**: ko/ 골격 만들면서 작업감 가늠 → 풀 미러링 필요성 (전체 nav 일관성 등) 발견되면 그때 전환
- **대안 검토**:
  - (a) 풀 미러링 — 빈 placeholder 100여개, 사용자 클릭 시 혼란
  - (b) 골격 + 점진 ← **채택**
  - (c) 점진 + nav 일괄 등록 — 미작성 페이지 클릭 시 404 → UX 더 나쁨

### `ja/`, `zh/` 처리 — nav 비활성 (파일 보존)

- **결정**: `ja/`, `zh/` **파일은 그대로 보존**, `docs.json` 네비게이션에서만 제외 + 한국어 디폴트 언어 설정
- **사유**: 회복 비용 비대칭 — 삭제는 비가역(orphan이라 git 히스토리 복원 불가), nav 비활성은 가역. 파일 자체가 작업에 방해 안 됨 (우리는 ko/에만 작성)
- **운영 효과**: mintlify dev UI에서 한국어만 보임 → 데모/배포 시점 깔끔
- **대안 검토**:
  - (a) 지금 삭제 — 비가역, 잘못 삭제 시 복구 불가
  - (b) Phase 4 일괄 삭제 — 시점만 늦춤, 본질 동일
  - (c) 유지 + nav 비활성 ← **채택**
- **재검토 트리거**: 한 달 작업 후 `ja/`, `zh/` 폴더가 실제로 거슬리면 그때 삭제 결정

### 번역 워크플로 — B안 (ko/ 직접 작성)

- **결정**: `en/`은 Dify 원본 그대로 보존, **`ko/`에 직접 한국어 작성**
- **사유**:
  - 5/29 이사님 지시 1차 목표는 사내 한국어 매뉴얼
  - 영문은 긴급도 낮음 (현재 외부 영문 데모 요청 없음)
  - A안(en/ 수정 → ko/ 번역)은 작업량 2배
- **대안 검토**:
  - A안 (2단계, en/ 먼저 수정 후 ko/ 번역) — 영문 문서 같이 생성되지만 노동 2배
  - B안 (ko/ 직접 작성) ← **채택**
- **재검토 트리거**: Phase 4 중반에 영문 매뉴얼 필요성 부상하면 그때 ko/ 기준으로 역방향 작성

### 파일럿 챕터 — Knowledge Overview (1차) + 대시보드 (2차)

- **결정**:
  - **1차 파일럿**: Knowledge Overview (`en/use-dify/knowledge/readme`) — 원본 번역 케이스
  - **2차 파일럿**: 대시보드 — 신규 작성 케이스
- **사유**:
  - 원본 번역과 신규 작성 두 패턴 다 검증해야 형식·톤 기준 확정 가능
  - Knowledge Overview는 짧고 핵심 기능이라 톤 검증에 적합
  - 대시보드는 5/28 이사님 관심 (대시보드 의미 설명), spx-agent 차별화 기능
- **목표**: 1·2차 파일럿 통과 후 `conventions.md`(톤·MDX 규칙)와 `scope-mapping.md`(워크플로) 확정 → Phase 3 이사님 컨펌 자료로 사용

### spx-agent 실행 환경 활용 — 텍스트 우선, 스크린샷은 Phase 5

- **결정**:
  - Phase 2 파일럿은 **텍스트 중심** 작성 (스크린샷 placeholder 또는 임시 캡처)
  - Phase 5에서 spx-agent 화면으로 일괄 교체
- **사유**: 텍스트·구조와 이미지를 분리하면 각 단계 효율 ↑. 텍스트 톤 확정 전에 이미지 확정하면 재작업 위험
- **실행 환경 보유 확인됨**: UI 동작 확인 가능 (정확도 검증용)
- **Phase 5 사전 작업 후보**: 화면 캡처 가이드라인(해상도·언어·계정·디스플레이 모드) 정의 필요

## 2026-06-02 — Docusaurus 전환 확정 (호스팅 방향)

### 결정

- **빌드·호스팅 도구**: **Docusaurus**로 전환 (Mintlify 옵션 전면 폐기)
- Phase 3.5(전환 실행)를 Phase 4(본격 포팅) 앞에 신설 → [[progress]]
- 작업량 산정·세부 작업: [[references/mintlify-to-docusaurus]] 감사 보고서 그대로 채택 (~4~5.5일)

### 사유

1. **사내망/외부 GitHub 의존 회피** — Mintlify Cloud는 GitHub 연동이 사실상 필수이고 사내 git(Gitolite)과 양방향 동기화 불가. 사내망 정책상 외부 SaaS 의존 최소화가 안전
2. **안정성** — 2026-06-01 Mintlify Cloud 배포 시 navigation 구조 호환성 에러 발생, 디버깅 불투명. Docusaurus는 오픈소스라 빌드·배포 전 과정 가시
3. **비용** — Mintlify Enterprise 별도 라이선스 비용 vs Docusaurus 0원

### 대안 검토

| 옵션 | 채택 여부 | 사유 |
|------|---------|------|
| Mintlify Cloud (`spelix.mintlify.app`) | ❌ 폐기 | 외부 GitHub 의존, 사내 git 연동 불가, navigation 호환성 에러 미해소 |
| Mintlify Enterprise | ❌ 폐기 | 라이선스 비용 + 외부 의존성 여전 |
| `mintlify dev` 임시 공유 | ❌ 폐기 | 빌드 산출물 정적 호스팅 불가, 데모/배포에 부적합 |
| **Docusaurus + 사내 nginx** | ✅ **채택** | 자체 호스팅, 정적 빌드, 검색 오프라인 가능, 비용 0 |

### 후속 작업 (Phase 3.5 진입 시 처리)

- [ ] Phase 3.5 8개 체크박스 진행 → [[progress]]
- [ ] `en/` 원본 보존 원칙 유지 — 변환 스크립트는 `ko/use-spx-agent/`만 대상
- [ ] `docs.json` 원본은 NOTICE 차원에서 read-only로 보존, 변환 산출물 `sidebars.js`는 별도 관리 (`CLAUDE.md` fork overrides 보존 대상 표 갱신 필요)
- [ ] `conventions.md` 갱신 — Mintlify 컴포넌트 표기 → MDXComponents 래퍼 문법, 내부 링크 형식 재정의
- [ ] `README.md` 갱신 — Phase 흐름 6→7단계, 기술 스택 Docusaurus로
- [ ] `NOTICE.md` 갱신 — CC BY 4.0 변경 사실 명시(빌드 도구 교체 + ko/ 신설 등 누적 변경)

### Mintlify Cloud 시도 흔적 (참조)

- 2026-06-01: `spelix.mintlify.app` 연결, navigation 호환성 에러 발생 → 본 결정으로 종결
- 데모 URL은 Docusaurus 빌드 후 사내 호스팅으로 대체

## 2026-06-02 — Phase 1.3 매트릭스 재검토 결과

### 배경

5/29 작성된 매트릭스를 6/2 라이브 docs(localhost:3001 = 1.13.3 baseline) + spx-agent 코드 분석을 병행하면서 9개 섹션 전수 재검토. 핵심 발견은 "전역적으로 적용해야 할 패턴"이 다수 존재했고, 페이지별로 매번 재판단하는 비용을 줄이기 위해 **전역 규칙**으로 박았다는 점.

### 전역 적용 규칙 5개 신설

페이지 액션과 무관하게 모든 페이지 번역 시 일괄 적용:

1. **Dify Cloud/SaaS 플랜 언급 제거** — Free/Pro/Team/Enterprise plan, Subscription, Billing, Seat, Quota
2. **"Cloud version에서는..." 분기 콜아웃** → CE 기준 단일 서술로
3. **Sandbox/Production 워크스페이스 분리(SaaS)** → CE 단일 환경 기준 + ⚠️ "sandbox" 용어 모호성 명확화 (SaaS 테스트 워크스페이스 vs `langgenius/dify-sandbox` 보안 격리 컨테이너 — 후자는 보존)
4. **"자체 호스팅 한정" 분기 + 환경 변수/배포 설정** → 분기 제거 + env var 본문은 `references/deployment-config-extracts.md`로 별도 추출. "관리자" 표현은 모호(워크스페이스 vs 시스템)하므로 회피
5. **외부 Dify 리소스 링크·언급** — GitHub `langgenius/*` 즉시 제거 + 공식 community/forum/Discord 제거 + Marketplace는 결정 보류(이사님 컨펌 대기) + 기능 확장 안내는 `references/feature-extension-extracts.md`로 추출

추가 규칙: 페이지 액션이 "유지·번역"이어도 위 항목이 본문에 있으면 번역 시 제거/추출. **매트릭스 액션은 변경 없음** (전역 규칙 적용은 페이지별 평가가 아닌 본문 처리).

### 매트릭스 변경사항

**확정 삭제 5건 → 7건**:
- ➕ `publish/webapp/web-app-access` — Dify Enterprise 전용 (`frontmatter tag: "ENTERPRISE"`). spx-agent RBAC은 *Studio 내 앱 권한* 영역으로 별개. Enterprise 미보유 → 페이지 삭제, 권한 모델은 신규 챕터 "권한 설정"으로 대체
- ➕ `knowledge/knowledge-request-rate-limit` — Dify Cloud 전용 quota. spx-agent 미보유

**부분 수정 격상**:
- `knowledge/create-knowledge/introduction`, `knowledge/knowledge-pipeline/create-knowledge-pipeline`, `knowledge/manage-knowledge/introduction` — 권한 설정 섹션 추가 + 신규 챕터 "권한 설정" 링크. Phase 4 진입 전 spx-agent 코드/UI 분석 필요(데일리 노트 deferred)

**다운그레이드** (5/29 분석 시점 오버스펙):
- `nodes/llm`, `workspace/model-providers` 부분 수정 → 유지·번역 — "vLLM 고정" 가정은 사용자 환경마다 백엔드 다를 수 있어 부정확

**비고 정밀화**:
- Get Started 3개 — 어제 가정 "KC SSO 로그인 흐름" 부정확 (실제는 사용자명/비번 폼) → 실제 화면 기준으로 정정
- `monitor/analysis` — Dify docs 페이지 제목 "Analysis"와 UI label "Monitoring" 분리 발견 → 한국어판은 UI 우선 "모니터링"으로 통일
- `publish/README` — Marketplace 처리 보류로 톤다운 (5/29 시점은 즉시 제거 명시 → 6/2 보류 정책 반영)

**검토 묶음 4 → 6 세분화**:
- 5/29 묶음: MCP / 플러그인 트리거 / 사내 망 외부 연동 / API Key
- 6/2 분해: 외부 SaaS observability 송출(Monitor integrations 7건) / 사내 망 기타 외부 호출 / 외부 데이터 import (지식 패턴 a) / 외부 KB read-only 연결 (지식 패턴 b) / 외부 노출 API (패턴 c) / API Key 발급

### 신규 챕터 5 → 4 (설정 사이드바 제거)

- ~~설정 사이드바 (KAN-29)~~ → **삭제** (2026-06-02)
- **사유**: Dify 원본에도 존재하는 설정 다이얼로그에 토글 기능만 추가된 수준 → 별도 챕터 거리 아님. 필요 시 기존 페이지(`web-app-settings`, `personal-account-management` 등) 부분 수정에서 한 줄 언급으로 충분
- **신규 합계**: 9~14p → **8~12p**

### 패턴 (a)/(b)/(c) 분류 도입 — 지식 외부 연결

검토 묶음의 "외부 연결" 카테고리는 실은 **방향이 3가지로 다름** — 묶어두면 컨펌 시 혼동 → 패턴별 분리:

- **(a) 외부 → 내부 import (데이터 복사)**: `sync-from-notion`, `sync-from-website`, `authorize-data-source` (Pipeline)
- **(b) 외부 KB read-only 연결 (복사 X)**: `connect-external-knowledge-base`, `external-knowledge-api`
- **(c) 내부 → 외부 노출 (API)**: `maintain-dataset-via-api`

→ 이사님 컨펌 시 패턴별로 분리해서 질의해야 명확

### 용어 글로서리 확장

`conventions.md` Glossary 보강:
- `Monitoring → 모니터링` 행에 **"Dify docs 페이지 제목 'Analysis'도 동일 영역, 한국어판은 UI label 우선 '모니터링'"** 명시
- `Dashboard → 대시보드` 신규 추가 (spx-agent 전용 섹션). **앱별 모니터링(Monitoring 탭)과 워크스페이스 KPI(신규 챕터 대시보드)는 별개 영역** 명시 — 한국어 작성 시 혼동 주의

### 부수 발견 (소급 박제)

- **WebApp 인증 모델 검증** (spx-agent 코드 분석): WebApp의 4가지 접근 모드는 Dify Enterprise 한정. spx-agent는 backend에서 3 모드만 허용(public/private/private_all) + enterprise microservice 외부 의존. **결론: 사실상 WebApp Access Control 기능은 spx-agent에 노출 안 됨** → web-app-access 페이지 삭제 근거
- **Tutorial: AI Image Generation**: 모델 보유 여부 의존 X (튜토리얼은 연결 방법 가이드, 사용자가 접근 가능한 모델로 적용) → 검토 → 유지·번역 다운그레이드

### 영향 / 후속

- Phase 3.5 / Phase 4 진입 시 본 매트릭스가 기준
- Workspace 섹션 전체는 **재검토 미완** — 2026-06-02 신설 규칙 적용 + 소스 분석 필요 (deferred)
- Marketplace / Plugin Trigger / MCP / 외부 호출 정책 등 컨펌 묶음은 이사님 컨펌 시 패턴별로 질의

## 2026-06-04 — Dify 브랜드 전체 제거 (rebranding 강화)

### 결정

전역 규칙 #5를 "외부 Dify 리소스 링크·언급 제거"에서 **"Dify 브랜드 자체를 사용자 매뉴얼에서 전면 제거"**로 강화. spx-agent를 독립 제품으로 표기.

### 적용 범위

- Dify 브랜드명·로고 → spx-agent로 교체 또는 통째 제거
- Dify 공식 외부 채널 (GitHub, community, forum, Discord, Twitter, LinkedIn 등) 전부 제거
- Dify 공식 docs/사이트 링크 (dify.ai, docs.dify.ai 등) 제거
- "Powered by Dify" 류 헤더·푸터 제거
- Marketplace는 보류 (별도 결정)
- 기능 확장 안내는 추출 보관 (Marketplace 결정과 묶음)

### 예외 (보존)

- 루트 `NOTICE.md` CC BY 4.0 출처 표기 — 라이선스 의무
- 내부 코드 컨테이너·서비스명 (`langgenius/dify-sandbox`, `dify-audit` 등) — 운영 식별자, 사용자 매뉴얼 노출 안 됨

### 사유

- spx-agent = 독립 제품 포지셔닝 (Dify 기반이지만 fork 후 자체 브랜드)
- 사용자가 "Dify"라는 단어를 보면 혼란 (어디서 도움 받지? 마켓플레이스 어디 있지? 등)
- 사내 매뉴얼에서 외부 Dify 리소스 안내는 의미 없음 (이미 5/29 시점 부분 적용, 2026-06-04 강화)

### 영향

- Phase 4 포팅 시 모든 페이지에서 Dify 브랜드 스캔 필요 (전역 규칙 #5 적용)
- 이미지·로고 교체는 Phase 5에서 일괄 (`<Frame>` 컴포넌트 alt·caption도 정리 대상)

## 2026-06-04 — 이사님 회의 결정 일괄 박제

### 결정 1: Marketplace + 플러그인 시스템 전체 제거

- **결정**: Dify의 Marketplace UI와 플러그인 시스템을 spx-agent에서 비활성화
- **사유**:
  - 사내(폐쇄망) 환경에서 외부 Marketplace 접근 어려움
  - 필요한 기능은 사내에서 미리 소스로 심어둘 예정 (사용자가 install하는 모델 아님)
  - 모델 제공자 설치는 **유지** (사용자 환경마다 백엔드 다를 수 있음 — vLLM / Ollama / 상용 모델 등)
- **영향**:
  - Plugin Trigger 노드(langbot, lark, telegram 등) → 매트릭스 ❌ 삭제
  - Marketplace 페이지·안내·기능 확장 안내 → 삭제
- **남기는 것**: 워크스페이스 > 모델 제공자 (사용자 환경 다양성 가정)

### 결정 2: 외부 연결 제거 (outbound)

- **결정**: spx-agent에서 외부 SaaS·서비스로 outbound 호출하는 기능 모두 제거
- **사유**: 폐쇄망 환경 가정 (사내망에서 외부 SaaS 의존 최소화)
- **영향 페이지** (매트릭스 ❌ 삭제, 총 14건):
  - Monitor integrations 7건 (LangSmith / Langfuse / Opik / Weave / Arize / Phoenix / Aliyun)
  - Knowledge 외부 import 3건 (Sync from Notion / Sync from Website / Authorize Data Source)
  - Knowledge 외부 KB 연결 2건 (Connect External KB / External Knowledge API)
  - Twitter Chatflow
  - workspace/api-extension/* 3건 (양방향 표기 정정 → outbound 일방향)

### 결정 3: MCP 유지

- **결정**: Model Context Protocol (build/mcp, publish/publish-mcp) 페이지 유지·번역
- **사유**: MCP는 사내·사외 관계없이 연동 가능 (사내 MCP 서버 운영 또는 사외 MCP 서버 모두 적용 가능)

### 결정 4: 검토 대기 inbound 3건 유지

- **결정**: 외부에서 spx-agent를 호출하는 inbound 영역 3건 유지·번역
- **대상**: publish/webapp/embedding-in-websites, publish/developing-with-apis, knowledge/.../maintain-dataset-via-api
- **사유**: 사내 다른 시스템에서 spx-agent를 호출하는 시나리오 가정 가능

### 결정 5: KC 추상화 전역 규칙 (#6 신설)

- **결정**: 모든 페이지 본문에서 Keycloak 같은 외부 인증·관리 시스템 명칭을 추상화 표기
- **표기**: "외부 시스템(예, Keycloak)" 또는 "관리 시스템"
- **사유**: spx-agent 사용 기업마다 IdP가 다를 수 있음 → 특정 제품 명시는 오해 유발
- **적용 범위**: 사용자·부서 관리 챕터 한정 X, **모든 페이지**

### 결정 6: workspace/team-members-management → 사용자·부서 관리 변환

- **결정**: ❌ 삭제 → 🔄 변환 (5/29 결정 정정). Dify 원본 team-members-management 자리를 신규 챕터 "사용자·부서 관리"로 대체
- **사유**:
  - 5/29 매핑 시점에는 spx-agent의 사용자·부서 관리 기능이 새 챕터로만 인식
  - 6/4 재검토: 실제로는 Dify 원본 team-members-management의 **확장판** (KC 동기화 + 부서 기능 추가)
  - 변환으로 처리하는 게 의미 일관성 ↑
- **영향**: 매트릭스에서 team-members-management를 별도 "변환" 카테고리로 이동

### 결정 7: CI/CD 표시 가이드

- **결정**: 개발/운영 환경 분리 기능은 별도 마커로 명시
- **사유**: spx-agent에 개발/운영 두 환경 띄울 때 앱 이관용 CI/CD가 있음 (현재 다른 분 작업 중, 미보유)
- **적용**: 향후 CI/CD 기능 추가 시 해당 페이지에 "이 기능은 개발 환경 전용입니다" 같은 표기

### 결정 8: 문서는 전체 사용자 대상, CI/CD만 환경 구분

- **결정**: spx-agent docs는 개발/운영 환경 구분 없이 전체 사용자 대상으로 작성
- **예외**: CI/CD처럼 개발 환경에만 있는 기능은 별도 마커

### 결정 9: "앱 권한 설정" → "권한 설정" 챕터명 변경

- **결정**: 신규 챕터 "권한 설정"을 **"권한 설정"**으로 rename
- **사유**:
  - A1 분석([[references/spx-app-permissions-analysis]]) 결과 챕터 범위가 **앱·지식·도구 공통 권한 모델 코어**로 확정
  - "앱" 한정 표기는 도메인 한정 오해를 유발 → A2(Knowledge 권한)에서 본 챕터 인용 시 어색
- **위치 결정 (Option G)**: 워크스페이스 그룹 안 `workspace/permissions/`
- **A1 작성된 1p는 그대로 활용** — 챕터 구조(소개·권한탭·가시성·자원권한·FAQ)는 변경 없음

### 결정 10: 사이드바 구조 Option α 채택 (Workspace 흡수 + 통계·감사 별도 그룹)

> 2026-06-04 같은 날 Option G에서 진화. Option G 시점의 "Monitor 안 워크스페이스 단위 sub-group" 작명이 직관적이지 않다는 우려로 별도 top-level 그룹(통계·감사) 신설로 정정.

- **결정**: Workspace 흡수(권한 설정·부서 관리) + **통계·감사 별도 top-level 그룹 신설**(대시보드·감사로그). 모니터링 그룹은 앱 단위 조회만 평면 구조
- **구조**:
  - 워크스페이스: 기본 설정 + 앱 관리 + 사용자·부서 관리(신규) + 권한 설정(신규) + 개인 계정 (8~10p)
  - 모니터링: Analysis / Logs / Annotation Reply (3p, 평면 — sub-group 없음)
  - **통계·감사** 🆕: 대시보드 + 감사로그 (4~5p, sub-group으로 응집)
- **사유**:
  - **설정 도메인(워크스페이스) vs 조회 도메인(모니터링·통계·감사) 분리**가 멘탈 모델 자연
  - 모니터링(앱 단위 조회)과 통계·감사(워크스페이스 단위 조회)를 top-level에 나란히 두면 "조회" 도메인이 scope별로 정렬
  - 그룹명 "통계·감사" = 대시보드(통계) + 감사로그(감사) 1:1 매핑으로 콘텐츠 즉시 파악 가능
  - team-members-management 자리에 사용자·부서 관리가 자연 변환
- **Option G에서 α로 변경 사유**:
  - "워크스페이스 단위" 같은 scope 라벨이 사용자에게 직관적이지 않음
  - 워크스페이스 그룹 = 설정 중심, 통계·감사 = 조회 중심 → 의미상 별도 도메인
  - 신규 그룹 1개 추가 부담 있지만 sub-group(대시보드/감사로그)으로 응집되어 비어 보이지 않음
  - 대시보드는 일반 사용자도 조회 가능 (admin only 아님) — 따라서 "관리자 분석" 같은 청자 기반 작명은 부적합 → 콘텐츠 기반 "통계·감사" 채택
- **대안**: Option A~G 검토 후 α(G의 정정형)가 최적
- **폴더 경로**:
  - `ko/use-spx-agent/workspace/permissions/` (기존 app-permissions/ 이동)
  - `ko/use-spx-agent/workspace/departments/` (신규)
  - `ko/use-spx-agent/monitor/` (평면 — Analysis / Logs / Annotation Reply)
  - `ko/use-spx-agent/analytics-audit/dashboard/` (기존 dashboard/ 이동)
  - `ko/use-spx-agent/analytics-audit/audit-log/` (신규 — B1 인터리브 1p)
- **이동 비용**: 기존 작성된 페이지 2건 이동 (app-permissions/, dashboard/)

### 결정 11: 데이터 조회·시각화 챕터 서술 원칙 확정 (대시보드·감사로그·모니터링)

> 2026-06-08. 5/28 이사님 지시("그림 화려하게보다 의미 설명에 초점, 로그 항목들이 어떤 건지 설명, 어떻게 업무에 적용·활용하는지")가 그동안 conventions §이미지 처리에 한 줄로만 있어 본문 작성 기준으로 작동하지 못함. 대시보드 분석본이 "차트 나열"에 그친 점을 계기로 본문 서술 원칙으로 명문화.

- **결정**: 화면을 "보는" 챕터는 차트·카드·컬럼을 나열·정의하는 데서 멈추지 않고, 각 요소를 **①무엇을 보여주는가(의미) → ②어떤 질문에 답하나/왜 중요한가 → ③업무에 어떻게 활용하나** 3단으로 서술
- **적용 대상**: 대시보드, 감사로그, 모니터링(앱별 통계 = Analysis·Logs 한정). **Annotation Reply는 설정 기능이라 비대상**
- **사유**:
  - 분석본(`references/spx-*-analysis.md`)은 화면·API 구조 박제용이라, 그대로 옮기면 "차트 나열" 문서가 됨 — 사용자에게 의미·활용이 전달 안 됨
  - 5/28 지시의 핵심은 "의미·업무 활용 초점", 이를 모든 데이터 화면 챕터에 일관 적용
- **박제 위치**: [[conventions]] §데이터 조회·시각화 챕터 서술 원칙(정본) / [[chapter-writing-checklist]] §2.9·§4.6·§4.10 / [[scope-mapping]] 보강·신규 챕터 표 비고 / [[references/spx-dashboard-analysis]]·[[references/spx-audit-log-analysis]] 헤더 안내
- **후속**: 대시보드·감사로그 본문 작성 시 본 원칙 적용. 모니터링 Analysis 작성 시에도 통계 지표에 적용

### 결정 12: 전역 규칙 #8 "문장 적합성 필터" 신설

> 2026-06-09. Monitor 리뷰에서 키워드 기반 전역 규칙(#1~#6)으로 걸러지지 않는 SaaS 맥락 문장이 번역에 포함되는 문제 발견. 예: "로그 익명화를 검토하시기 바랍니다"(제품에 없는 기능 권고), "해당 지역의 데이터 보호 규정"(SaaS 글로벌 서비스 맥락).

- **결정**: 원본 문장마다 "spx-agent 사용자가 이 안내대로 실행할 수 있는가?" 판단. 제품에 없는 기능을 권고하는 문장, 자체 호스팅 환경과 무관한 운영 맥락은 번역하지 않고 삭제
- **사유**:
  - 원본이 SaaS + CE 혼합 독자 대상이라, 키워드(Free, Pro plan 등) 없이도 SaaS 맥락이 스며든 문장이 존재
  - #1~#7은 키워드로 잡을 수 있지만, "~를 검토하시기 바랍니다" 류 일반론은 키워드가 없어 통과됨
  - 같이 추가: i18n 검증 범위를 "UI 라벨"에서 "UI 라벨 + 도메인 개념어"로 확장 (적중→조회, 매칭→일치 사례)
- **박제 위치**: [[scope-mapping]] 전역 규칙 #8 / [[conventions]] §피해야 할 패턴 + §i18n 검증 범위 / [[chapter-writing-checklist]] §2.1 + §2.2

### 결정 13: 노드명·부서 용어 확정 (번역 검증 하 P1 처리 중)

> 2026-06-15. 번역 검증 리포트 하 P1 17건 i18n 대조 처리 중 발견한 라벨 충돌 2건을 이사님이 결정.

- **노드명 "템플릿"으로 통일** (기존 "템플릿 변환" 폐기): i18n `blocks.template-transform`="템플릿"이 권위. `nodes/template.mdx`·`iteration.mdx`·`loop.mdx`의 "템플릿 변환" → "템플릿" 일괄 변경. [[conventions]] 글로서리 Template 행 갱신.
- **User Input 노드 = "시작"**: 캔버스 노드명·변수 프리픽스 모두 "시작"(`blocks.start`). "사용자 입력"은 노드 추가 시 노드 피커에서만 노출(2026-06-15 실제 UI 확인). 문서도 "시작" 사용. quick-start finding #7 원복 완료, [[conventions]] User Input 행 갱신, 메모리 [[project_user_input_node_label]] 박제.
- **다중 부서 멤버십 = 가능으로 확정**: `workspace/departments/readme.mdx`의 `<Info>`("한 사용자가 여러 부서에 속할 수 있습니다") **유지**. 근거: [[references/spx-departments-management]] §4.3(부서 셀 다중 선택)·§11("멤버십 set은 다중, owner_department_id는 단일"), code-verified 2026-06-04. 제품 모달 문구(`membersModal.description`="한 명은 한 부서에만…")와 부서 셀 UX가 상충하나, 데이터 모델·부서 셀 기준 다중이 정답.
- **departments 용어 분리**: 부서 스코프 멤버 관리 UI는 i18n이 "구성원"(`rbac.department.*`), 워크스페이스 멤버 탭은 "사용자 관리"(`settings.members`). 둘을 구분해 반영. "## 멤버 페이지" 등 일반 서술 헤딩은 구조 변경 risk로 "멤버" 유지. [[conventions]] §UI 라벨에 박제.

### 결정 14: 번역 검증 하 P2 백로그 4건 처리 방침 확정

> 2026-06-16. 하 P2 백로그(검증 필요분)에 대한 이사님 결정.

- **`파라미터` → `매개변수` 전역 적용**: 문서 독자가 개발자·비개발자 혼재이므로 글로서리(매개변수)를 일괄 적용. node 도메인뿐 아니라 HTTP/도구/모델 매개변수 등 일반 'parameter' 전부 포함. 13개 파일 일괄 변경 완료(`매개변수`·`파라미터` 모두 모음 종결이라 조사 안전). [[conventions]] 글로서리 Parameter=매개변수 권위.
- **다중 자격 증명 = spx 보유 기능 확정**(2026-06-16 코드 조사): `model-provider-page/model-auth/`(`manage-custom-model-credentials.tsx`, `credential-selector.tsx`)에 **게이트 전무** — `IS_CE_EDITION`·라이선스·업그레이드 체크 없음, `useCustomModels` 기반 렌더링만. → 환경 분리·비용 최적화·모델 테스트 3종 모두 유효. 누락됐던 **"비용 최적화" 시나리오 1건 복원**.
- **로드 밸런싱 = 제외 유지(유료/Enterprise 기능)**: ⚠️ 사유 정정. 처음엔 i18n `upgradeForLoadBalancing`만 근거로 했으나, 코드 조사 결과 프런트 게이트는 `!modelLoadBalancingEnabled && !IS_CE_EDITION`(`IS_CE_EDITION = EDITION==='SELF_HOSTED'`)라 **self-hosted에서는 클라우드 업그레이드 CTA만 숨김** — 커뮤니티에 기능을 여는 게 아님. **정확한 근거는 en 원문**(`workspace/model-providers.mdx` L161-163)의 명시: *"Load balancing is a paid feature — paid SaaS subscription **or Enterprise license**"*. self-hosted 유료 경로 = Enterprise 라이선스. **이사님 확인(2026-06-16) "CE 버전이야" → spx-agent = 커뮤니티 self-hosted 확정**이므로 로드 밸런싱 미보유 **확정**(보수적 추정 아님). 전역 규칙 #1 적용해 로드 밸런싱 섹션·"Default Config" Info 미작성 유지(정상). 에디션·기능 게이트 판정법 정전: [[references/spx-edition-feature-gating]].
- **거부(DENY) 행 = 매뉴얼 비노출**: [[references/spx-app-permissions-analysis]] §결론(L120·122·472) — DENY는 일반 UI에 없고 관리자 API로만 등록. "유저 친화적이지 못함" 판단으로 `workspace/permissions` FAQ에서 DENY 행 직접 노출 제거, "관리자가 의도적으로 접근을 제한했을 수 있으니 확인 요청"으로 완화.
- **Match(매칭)→일치 / cleaning(정제) 유지**: Match는 글로서리에 일치(~~매칭~~) 있음 → 자연스럽게 reword하여 일치 적용. cleaning(정제)은 글로서리에 없고 "정제"가 자연스러운 한국어라 유지(원문 의미 통함). 원칙: **글로서리 있으면 글로서리, 없으면 어색하지 않은 자연어**.

## 2026-06-17 — 이미지 교체/제거 작업 설계 결정

> ko 문서의 Dify 원본 이미지(전부 영문 UI)를 spx-agent 화면으로 교체/제거하는 작업. 설계 전문: [[image-replacement-plan]].

### 결정 15: 판별 축 = 단일 이분(교체/제거), 텍스트 스캔 주력

- 세션 논의로 판단 축이 단일 이분으로 붕괴: **본문이 이미지를 필요로 하는가** → 필요하면 교체(재캡처), 아니면 제거.
- 근거 전제 3가지: ① 기존 이미지 전부 Dify 원본(사용자 의도 삽입 아님) → **파일명 분류 폐기**, ② 전부 영문 UI → "언어무관 유지" 예외 없음, ③ 본문이 이미 spx 기준 → 기능 게이트는 텍스트에 반영 완료, 코드 대조 불요.
- **텍스트 스캔이 비전 스캔보다 효율·정확**(필요성은 본문의 속성이지 기존 이미지 내용이 아님). 313장 전수 비전 열람 폐기.
- **제거는 비가역**이라 가드 적용: 본문 명시적 참조("아래/그림/표/처럼") + alt 텍스트 프록시 + 잔여 소수 비전 스팟체크. 비전은 제거 후보에만 좁게 사용.

### 결정 16: 작업 운용 방침 4건 (2026-06-17 이사님 결정)

- **신규 챕터 분리, A 먼저**: 기존 이미지 교체/제거(A 스트림) 먼저, 이미지 0개인 spx 전용 신규 챕터(대시보드·권한·부서·감사로그) 캡처(B 스트림)는 별도. (확인: 신규 챕터 3곳 이미지 0개)
- **애매 시 review 대신 권장 표시**: 등급화에서 애매한 행을 빈 `review`로 미루지 않고, **교체/제거 중 권장값을 추정해 `suggested` 플래그로 표시** → 사람은 권장값을 확인·뒤집기만. (별도 review 버킷 제거)
- **파일럿 먼저**: 1~2개 챕터로 인벤토리→등급화→캡처→삽입 전 과정을 검증한 뒤 전체 확대.
- **수동 캡처 + 에이전트 후처리**: 사람이 캡처(관리자 화면 SSO·시드데이터 포함), 에이전트가 리네임·배치·삽입·빌드검증. Playwright 자동 캡처는 도입 안 함(셋업 비용 회피). → 후속 "화면 캡처 가이드라인" 항목 부분 해소.

### 결정 17: 신규 이미지 배치 = `static/images/`만, 루트 `images/` 사본 안 함 (2026-06-17)

- Docusaurus는 `/images/...` 마크다운 참조를 **`static/images/`** 에서 서빙(빌드 필수). 루트 `images/`는 Mintlify 시절 원본 보존 사본.
- quick-start 파일럿에서 루트 `images/`에만 넣었다가 **빌드 실패** → `static/images/`로 해결.
- **결정**: 신규 캡처는 `static/images/`에만 둔다. 루트 `images/`엔 사본을 두지 않는다(원본 보존 영역과 신규 자산 분리). 이미 넣었던 quick-start 15장 루트 사본은 제거 완료.
- 절차 반영: [[image-replacement-plan]] §4·§5·체크리스트.

## 후속 결정 대기 (Phase 3 이사님 컨펌)

- ~~호스팅 방식 확정 (Mintlify Cloud / Docusaurus / 임시)~~ → **2026-06-02 Docusaurus 확정** (위 참조)
- 사내 git namespace 결정 (`spx/dify-docs-spx`?)
- IBM 문서 참조 범위 (5/28 회의 단서, 구체 미정)
- 번역 톤 (존댓말 / 평어체) 파일럿 후 합의
- 영문 병기 정책 확정
- 화면 캡처 가이드라인 (해상도·언어·계정)
- ~~신규 챕터 우선순위 (대시보드 / 감사로그 / RBAC / KC SSO 중)~~ → 재구성 완료 (위 참조)
- ~~검토 항목 4가지 질문 확정 (MCP / 플러그인 트리거 / 사내 망 외부 연동 / API Key 정책)~~ → **2026-06-04 일괄 해소** (위 결정 1~4 참조)
