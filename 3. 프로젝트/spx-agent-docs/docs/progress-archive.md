# 진행 상황 — 작업 로그 아카이브 (시간순)

> 본 파일은 [[progress]]에서 분리한 **시간순 작업 로그**입니다. 과거 작업 내역 보존용.
> 라이브 진행 상태·다음 작업·Phase 진척은 [[progress]] 본문을 참조하세요.
> 분리 시점: 2026-06-09 (progress.md 비대화 → 라이브/이력 계층 분리).

## 작업 로그 (시간순)

### 2026-06-15 (워크스페이스 검수 반영 + 완료 그룹 소급 QA 패스)

> 워크스페이스 그룹 검수 피드백([[review/08-워크스페이스]]) 반영에서 출발 → 신설 규칙(G1~G7)을 완료 그룹에 소급 적용. 방법론 박제: [[references/qa-pass-method]].

- **워크스페이스 6p 검수 반영** — readme·app-management·model-providers·personal-settings·permissions·departments. G1(없는 기능 삭제)·G2(UI 라벨 i18n 대조)·G3(모달/다이얼로그 제거)·W1~W5(메뉴 상단·대시보드 역할무관·설정 진입) 적용. personal-settings는 W5로 프로필·로그인 방식 절 삭제 → 언어·시간대만으로 축소(account-setting 코드 `ACCOUNT_SETTING_TAB` 검증). 빌드 SUCCESS
- **제품 코드 검증 2건**:
  - **로드 밸런싱 = CE 기본 비활성** (`MODEL_LB_ENABLED` 기본 `False`, billing 분기는 SaaS 전용, docker/.env 미설정) → model-providers "## 로드 밸런싱" 절 삭제. 근거 [[references/spx-workspace-analysis]] §3.5 + [[scope-mapping]] #1 박제
  - **web-app-settings 법적 필드 3종** (`copyright`/`privacy_policy`/`custom_disclaimer`) 확인 → "데이터 처리 설정·이용약관"(실재 안 함) → 면책 조항으로 교체
- **규칙 박제** — **G1~G7·W1~W5** → [[conventions]] §검수 발견 전역 규칙 신설. 모든 그룹 공통 적용 명시
- **소급 grep 패스 (G3/G1/#8)** — 완료 8그룹 81파일, **fan-out 5 서브에이전트**(읽기 전용) 전수 점검. 종합 16건 → **17건 적용**(적대적 검토자 발견 departments 모달 1건 추가). #8 거버넌스 3건(logs 데이터보호 정책·web-app-settings 규정준수)은 [[references/translation-verification-report]]가 못 잡은 키워드리스 SaaS 맥락 — 본 패스 핵심 성과. 적대적 검토 서브에이전트로 회귀·누락 0 확인
- **정성 패스 (결정11/G6)** — fan-out 3 서브에이전트. **결정11**: 대시보드·감사로그·logs 충족(다른 세션 확장분이 결정11 준수), analysis 워크플로우 토큰 사용량 1건만 ②③ 보강. **G6**: 13건 플래그됐으나 "시스템이" 남발은 군더더기 → 과잉보고 필터로 1건만 적용(knowledge/permissions "검색 권한 반영이")
- **방법론 문서 신설** — [[references/qa-pass-method]] (fan-out·빌드 게이트·적대적 검토·과잉보고 필터·중복 제외·재실행 체크리스트). 다른 세션 단독 실행 가능
- **검증 방식**: 전 단계 fan-out(컨텍스트 격리) + `npm run build` 게이트 + 적대적 검토 — Claude Code best-practices 적용
- **빌드**: 각 적용 단계 SUCCESS, 신규 broken 0건. 커밋은 사용자 직접 (워킹트리 정리 상태)

### 2026-06-12 (통계·감사 그룹 본문 확장)
- **대시보드·감사로그 임시 1p → 본문 확장 완료** (통계·감사 그룹, §B 신규 챕터 + §C 데이터 시각화 적용)
- 작업 절차: 사용자 지시로 직역체 검수(§2.4)·i18n 사후 검증·전역 규칙 잔존 grep(§2.5) 생략, 그룹 마감은 빌드 검증만 수행
- **§1 그룹 사전 점검**: scope-mapping(신규 챕터→§B+§C)·decisions(결정 10 사이드바·결정 5 KC 추상화)·i18n(`audit-meta.ts` 라벨 글로서리 기반영)·sidebars 등록 확인(line 157-158)·분석본 2종([[references/spx-dashboard-analysis]]·[[references/spx-audit-log-analysis]]) 확인
- **대시보드** (`analytics-audit/dashboard/readme.mdx`): 라벨 나열 → §C 3단 해설(의미→왜 중요→업무 활용) 전면 재서술. KPI 4종 읽는 법·행동 신호, 종합 뷰 3요소 인사이트(자원 편중·비용 집중·부서 ROI), 드릴-스루 "누가/어디가 그 숫자를 만들었나" 일관 프레이밍. `sidebar_label` "개요"→"대시보드" 교정(플랫 항목 일관성)
- **감사로그** (`analytics-audit/audit-log/readme.mdx`): 기존본 충실 → §C 보강. 이벤트 조회 표 "읽는 법" 열 추가, 카테고리 행위/위험 관점 해설, **활용 시나리오 절 신설**(변경 추적·보안 점검·책임 소재·권한 감사·정기 보고), 필터 조합 `<Tip>`. 내보내기 건수 상한 단정 회피(분석본 §9 코드 검증 반영) 유지, KC 추상화·데이터셋=지식 안내 유지
- **빌드**: `npm run build` SUCCESS. 신규 broken 0건 (검출 2건은 기존 deferred — knowledge-retrieval `#create-knowledge`, variable-assigner `#variables`)

### 2026-05-29
- 옵시디언 폴더 골격 5개 파일 생성 ([[README]], [[scope-mapping]], [[conventions]], [[progress]], [[decisions]])
- references 2개 (`dify-docs-structure`, `ibm-research`)
- 원본 리포 fork: `Projects/spx-agent-docs/` (git clone, CC BY 4.0 확인)
- 옵시디언 폴더 ↔ 코드 폴더 junction 연결 (`.claude` mirror)
- **Phase 1 완료**:
  - docs.json 분석 → Use Dify dropdown 식별 (4개 중 1개만 대상)
  - en/use-dify/ 디렉토리 구조 파악 (9그룹, ~102 pages)
  - 3-way 매핑표 1차 완성: 유지·번역 66 / 부분 수정 9 / 삭제 5 / 검토 23
  - 신규 챕터 5종 확정 (대시보드·감사로그·RBAC·KC SSO·설정), ~10p 추정
  - 검토 항목 4가지 질문으로 클러스터링 (MCP 지원 / 플러그인 트리거 / 사내 망 외부 연동 / API Key 정책)
  - **spx-agent 코드베이스 분석 완료** → 신규 챕터 재구성:
    - ~~KC SSO~~ 삭제 → Quick Start에 로그인 흐름 흡수
    - ~~RBAC~~ → **권한 설정** + **사용자/부서 관리**로 분리
    - 신규 챕터 ↔ spx-agent 코드 매핑 표 추가
    - 부분 수정 보강 사항 9건 식별 (nodes/llm 모델 선택 등)
    - 용어집에 spx-agent 신규 용어 11건 추가
- **Phase 2 진입 결정 6건 기록** → [[decisions]] "Phase 2 진입 결정" 섹션
  - writing-guides: 하이브리드 (영역별 분기 — formatting 참조 / style 우리 작성 / glossary 우리 한국어 별도)
  - ko/ + docs.json: 골격 + 챕터별 점진 (풀 미러링은 작업 중 재판단)
  - ja/zh: nav 비활성 (파일 보존, 가역적 선택)
  - 번역 워크플로: B안 (`en/` 미수정, `ko/` 직접 작성)
  - 파일럿: 1차 Knowledge Overview (번역) → 2차 대시보드 (신규)
  - 실행 환경: 텍스트 우선, 스크린샷은 Phase 5
- `phase2-prereqs.md` 삭제 (결정 기록 완료, 임시 문서 역할 종료)
- **Phase 2 사전 셋업 완료**:
  - `conventions.md` 재구성: formatting-guide 참조 위임, 한국어 톤 placeholder, 용어집 대폭 확장 (노드 23종, UI 레이블, 지식·검색 용어, 워크스페이스 역할)
  - `ko/use-spx-agent/` 골격 생성: `knowledge/readme.mdx` + `dashboard/readme.mdx` placeholder
  - `docs.json` 수정: ko 언어 추가(디폴트), zh/ja nav 제거(파일 보존), "spx-agent 사용" dropdown
  - `mintlify dev` 검증 통과: ko 리다이렉트·렌더링 정상, en 유지, zh/ja nav 미노출

### 2026-06-01
- **Phase 2 파일럿 완료**:
  - 1차 Knowledge Overview 번역 + 톤 검증 → 합쇼체 확정
  - UI/브랜딩 정리 12항목 (로고, nav, Changelog, Studio, footer, GA4, redirects 등)
  - 2차 대시보드 신규 작성 (spx-agent admin 컴포넌트 전수 분석 기반)
  - conventions.md 최종 확정
- **파일 경로 변경**: `ko/use-dify/` → `ko/use-spx-agent/`
- **GitHub 리포 정리**: origin → `mjlee-spelix/spx-agent-demo`, upstream → 참조용
- **Mintlify Cloud 연결**: `spelix.mintlify.app` — navigation 구조 호환성 에러 해소 중
- **호스팅 옵션 비교표** 작성 (데일리 메모)
- **대시보드 분석 결과** → [[references/spx-dashboard-analysis]] 정리
- **Phase 3 사전 조사 완료 — Mintlify → Docusaurus 전환 감사**:
  - en/use-dify/ 106 MDX 파일 대상 전수 조사
  - 컴포넌트 14종 ~769회 (Frame 191, Info 127, Steps 122, Tip 60 등)
  - 프론트매터 6종 키 (Mintlify 전용: icon, sidebarTitle, tag, mode)
  - 링크 348건 (내부 62 절대경로 + 외부 286), 이미지 384건 (로컬 47%, CDN 47%)
  - MDXComponents 글로벌 래퍼 전략 채택 → MDX 본문 수정 최소화
  - 작업량: ko/ 만 1~1.5일, ko/+en/ 4~5.5일
  - 검색: @easyops-cn/docusaurus-search-local 권장 (한국어 오프라인)
  - 산출물: [[references/mintlify-to-docusaurus]]

### 2026-06-10 (독립 검토 선행 2건 처리)

- **readme/index 링크 폴더경로 교체** — Docusaurus가 `readme.mdx`를 폴더 인덱스로 처리하여 `/readme`로 끝나는 링크가 실제 broken. 5건 교체: introduction 2(`./import-text-data/`, `../permissions/`), import-text-data 1(`../../knowledge-pipeline/`), 개요 1(`./knowledge-pipeline/`), analysis 1(`../analytics-audit/dashboard/`). conventions에 "readme.mdx/index.mdx는 폴더 슬러그" 규칙 박제(사용자 직접 편집)
- **#8 보존 기준 적용** — setting-indexing-methods 재순위 모델 설명에서 "추가 토큰이 소비됩니다" 3건 복원 (벡터·전체텍스트·하이브리드). pricing page 참조만 삭제하고 제품 실재 정보(토큰 소비)는 보존. conventions "유효 정보 동반 삭제" 피해야 할 패턴 추가(사용자 직접 편집)
- 빌드 SUCCESS. introduction 발 broken link 0건 확인

### 2026-06-10 (Knowledge 1차 배치 create-knowledge 4p + 미니 체크포인트)

- **create-knowledge 4p 작성** (introduction · import-text-data · chunking · setting-indexing-methods)
  - introduction(부분수정 §A): Notion/Website 링크 삭제(결정2), 권한 설정 `<Tip>` 추가 + permissions 링크
  - import-text-data: SaaS batch 유료플랜 `<Info>` 삭제(#1), self-hosted env var `<Tip>` 2개 삭제(#4), env var 4건 deployment-config-extracts 박제
  - chunking: "Available for self-hosted deployments only" 분기 제거(#4), Tabs(일반/부모-자식) 구조 보존, 비교 표 번역
  - setting-indexing-methods: "Dify's knowledge base" 2회 삭제(#5), pricing page 참조 3회 삭제(#8), "self-hosted only" 분기 제거(#4). 벡터/전체텍스트/하이브리드 검색 3종 + 경제적 탭 전체 번역
- **검증**
  - 직역체 grep: 1건 수정 ("검색 결과를 제공합니다" → M2 교정)
  - 전역 규칙 grep: 0건 (Dify/pricing/SaaS 완전 제거)
  - 빌드 SUCCESS. readme.mdx 링크 오탐 가능성 — deferred
- **보완 (체크리스트 미이행 항목 사후 처리)**
  - §2.4 직역체 3패스: 4p 페이지별 원문 대조 완료, 누락 0건
  - §2.2 글로서리: 11건 추가 (구분 기호, 최대 청크 길이, 청크 중첩, 일반, 부모-자식, 재순위 모델, 고품질, 경제적, 역인덱스, Q&A 모드, Delimiter). Full-Text Search 중복 1건 제거
  - §2.5 i18n 사후 검증: 14건 대조 전부 일치, "부모-자식" 공백 차이 기록
  - §2.5 env var: 4건 deployment-config-extracts 박제
- **🚦 미니 체크포인트 결과**: #8 발동 4건·오탐 0건 / i18n §1.3 테이블 효과적 / 링크 상대경로 통일 / 규칙 보정 불필요 → 나머지 ~8p 진행 가능

### 2026-06-10 (Knowledge 파일럿 2p 작성 + 선행 보정)

- **Test Retrieval** (`ko/use-spx-agent/knowledge/test-retrieval.mdx`) 신규 작성
  - i18n `dataset-hit-testing.json` 전수 검증: 검색 테스트·레코드·소스 텍스트·질의 내용 등 라벨 확인
  - 전역 규칙 #1~#8 스캔: 해당 없음 (깨끗한 페이지)
  - 직역체 3패스 통과, 전역 규칙 grep 0건
  - deferred: `./create-knowledge/setting-indexing-methods` dangling link (타겟 미생성)
- **Knowledge 개요** (`ko/use-spx-agent/knowledge/readme.mdx`) 소급 QA
  - #8 적용: "외부 지식 베이스 연결" 항목 삭제 (결정 2 — 외부 연결 제거)
  - 용어 교정: "지식 베이스"→"지식" (글로서리 기준, 5개소)
  - i18n 교정: "인덱스 방식"→"인덱스 방법", "검색 전략"→"검색 방법", "전문 검색"→"전체 텍스트 검색"
  - "다양한 플러그인"→"커스텀 처리 단계" (결정 1 영향 가능성, 안전한 표현으로)
  - Read More 블로그 7건 삭제 확인 (#5)
  - 직역체 소소 교정: "이는 ~방식으로"→"검색 증강 생성(RAG)이라는 방식으로", "~에 기반한"→"~을 바탕으로"
  - dangling links 6건 (타겟 미생성, Knowledge 본격 작성 시 해소)
- **글로서리** 6건 추가: 검색 테스트, 레코드, 소스 텍스트, 전체 텍스트 검색(교정), 인덱스 방법, 검색 방법
- **sidebars.js**: test-retrieval 등록
- **빌드**: SUCCESS (dangling links는 경고, 타겟 미생성으로 예상)
- **파일럿 규칙 검증 결론**: #8은 개요에서 1건(외부 KB 삭제) 발동, i18n 검증은 Full-Text Search 교정 등 효과적. 규칙 안정화 확인 → 나머지 Knowledge 본격 작성 진입 가능
- **선행 보정 2건 완료**:
  - ① 내부 링크 형식 통일: conventions에 "dangling 시에도 상대경로 유지" + "앵커는 ko 한글 헤딩 기준" 방침 추가. Knowledge 개요 6건 절대→상대 소급 정정, Test Retrieval 영문 anchor 제거
  - ② 체크리스트 §1.3 개선: i18n "파일 식별"→"그룹 라벨 일괄 추출"로 변경. §2.2를 테이블 대조만으로 경량화

### 2026-06-09 (Monitor 3p + Publish 7p 작성 + 리뷰 반영 + Phase 4 커밋)

- **Monitor 3p + Publish 7p 작성** — 체크리스트 전 항목 통과
  - Monitor: Orchestrate 글로서리 교정("편성"→"오케스트레이트"), 점수 임계값 교정, A/B 테스트 항목 보충
  - Publish: "대화 시작"→"채팅 시작" 버튼 라벨 교정(i18n `chat.startChat`), API Access 글로서리 교정("API 접근"→"API 액세스")
  - deferred 3건: ① Logs env var 3건 → `references/deployment-config-extracts` 박제 ② Overview rate limits 항목 제거 보류 ③ 워크플로우 통계 UI 라벨 불일치(코드가 대화형 i18n 키 재활용 — 문서는 실제 의미로 서술, UI 정정 시 재대조)
- **Monitor 리뷰 🔴3건 코드 조사** (`references/spx-monitor-review`) 후 본문 반영 — analysis 2탭·logs 2탭+개인정보 재구성·annotation 경로 정정+대화형 전용 명시. 글로서리 로그·트리거 라벨 7건 추가
- **리뷰 추가 반영** — 어노테이션 용어 교정(적중→조회, 매칭→일치, i18n 근거), 개인정보 섹션 간결화, 토글 이름("주석 응답") 추가. **전역 규칙 #8(문장 적합성 필터) 신설** → scope-mapping·conventions·checklist 동기화
- **Phase 4 산출물 git 커밋** (5개 커밋, 워킹트리 clean)

### 2026-06-09 (Get Started 3p Phase 4 본격 작성 완료)

- **C1 deferred 7건 전량 처리** — Get Started 자체 작업 완료
  - Quick Start 2단계 워크플로우 빌드 본문 전체 번역 (원본 9노드 상세)
  - `gpt-5.2` 모델명 일반화 (직접 언급 제거 → "LLM"/"모델")
  - 로그인/SSO 표현 원칙 확립 — "SSO로 로그인하는 경우" 분기 제거, scope-mapping #6 박제, ko/ 4파일 6개소 일괄 수정
  - UI 라벨 i18n 전수 검증 — 불일치 10건 수정 (매개변수 추출기, List 연산자, Doc 추출기, 시작, 비전, 지시, 구조화된, 게시하기 등)
  - 이미지 누락 복원 — introduction 1 + key-concepts 7 + quick-start 1
  - MDX 포맷 수정 — key-concepts H3→H2 승격, Jinja2 코드블록 언어 태그, introduction description frontmatter
  - UI 텍스트 — "빈 상태로 시작"/"만들기"/"시스템 모델 설정" i18n 반영, 앱 생성 Info 뉘앙스(기본 비공개 + 권한 안내)
- **chapter-writing-checklist 기준 전체 검증 통과** — 전역 규칙 잔존 0건, MDX 포맷, 문체, 빌드 SUCCESS
- **남은 후속 (다른 챕터 의존)**: key-concepts 깨진 링크 3건 + introduction 튜토리얼 카드 → 대상 챕터 작성 후 자동 해소. 이미지 Phase 5

### 2026-06-05 (번역 품질 — 직역체 회피 체계 + 톤 일관성 + key-concepts 보강)

> 기존 ko 페이지 번역 품질 점검 중 직역체·톤 불일치·내용 누락 발견 → 규칙 체계화 + 전수 정비.

- **직역체(translationese) 회피 체계 구축**
  - [[conventions]] `### 직역체 회피` 신설 — 번역 절차(의미 재구성 4단계) + 직역 마커 M1~M11 체크리스트
  - **[[translationese-guide]] 사례집 신설** — 마커별 Before/After 예시 모음(누적) + grep 마커 + 검토 이력
  - [[chapter-writing-checklist]] §2.8 "직역체 회피(2-pass 번역)" 단계 추가
- **톤 일관성 + 표준 어미 확정**
  - conventions `### 톤 일관성` 신설 — 번역/신규 챕터 동일 톤, **신규(spx-native) 챕터는 "직역 검토"만 면제·톤·구조·용어는 동일 적용**, 작성 기준점(같은 그룹 기존 챕터) 규칙
  - **청유·명령 표준 어미 `~하시기 바랍니다` 확정** — `~하세요`/단독 `~하십시오` 지양. 존댓말 정책 박제 + checklist §2.3
  - 기존 페이지 어미 정리: quick-start 3건(`참고하세요`)·audit-log 3건(`요청/내보내/새로고침 하십시오`) → 표준 어미. grep 잔존 0건 확인
- **기존 ko 15p 전수 직역 검토 (EN 원문 1:1 대조, 3p씩 배치)**
  - 교정 18건: key-concepts 11 + knowledge/readme 2 + history-and-logs 3(M2 Shows) + variable-inspect 1(M6) + permissions 1(M3)
  - **결론**: key-concepts가 유일한 직역 다발 페이지, 나머지 번역 페이지는 대부분 양호. **spx 신규 챕터(권한·부서·대시보드·감사로그·지식권한)는 한국어 네이티브라 직역 대상 아님**(EN 원문 없음) — translationese-guide 검토 이력에 박제
- **key-concepts.mdx 내용 보강 (직역 외 별건)**
  - **⚠️ sys 변수 표 복원** — 원문(en)에 있던 `<Tabs>` 워크플로우/챗플로우 **시스템 입력 변수 표**(`sys.user_id`·`app_id`·`workflow_id`·`workflow_run_id`·`timestamp`·`conversation_id`·`dialogue_count`)가 ko에 **누락돼 있던 것 발견** → `### 시스템 입력 변수` 절 신설로 번역 복원. 의도적 축약 아닌 단순 누락으로 판단
  - `User Input` 노드 링크 라벨 `[시작]` → 글로서리 위반 → `[사용자 입력]` 정정
- **빌드** — `npm run build` SUCCESS. 신규 broken link 없음(기존 미포팅 페이지 링크만 잔존), 새 `<Tabs>`/표 MDX 정상 컴파일

### 2026-06-05 (신규 분석 — 부서 필터링 UI)

- **부서 필터링 UI 분석 완료** → [[references/spx-department-filter-analysis]]
  - 컴포넌트 식별: `web/app/components/header/account-dropdown/department-selector/index.tsx` — Dify 원본의 워크스페이스 전환 자리를 spx 단일 워크스페이스 정책에 맞춰 **부서 컨텍스트 전환으로 대체**
  - 상태 관리: `useSelectedDepartmentStore` (zustand `persist` → `localStorage["spx:selected-department"]`)
  - **영향 범위 — 앱·지식 2종만**: 도구 목록은 부서 필터링 미적용(`useSelectedDepartmentStore` 사용처 grep 0건)
  - **백엔드 동작**: `owner_department_id` 쿼리 파라미터로 일치 자원만 좁힘. **권한 평가는 별도 layer** — 권한 우회 안 함
  - **노출 분기 4가지**: OWNER/ADMIN(전체+활성 부서) / 비-admin 다중(본인 부서만, "전체" 옵션 없음) / 비-admin 단일(read-only 라벨) / 비-admin 0개(미렌더 + onboarding redirect)
  - **"(미지정)" 의미 확인**: 헤더 셀렉터에는 없음. 비슷한 라벨("할당 없음"/"미배정")은 멤버 페이지·대시보드의 별도 화면에 등장 — 의미 다름(부서 0개 사용자/자원)
  - **i18n 미등록 발견**: `common.departmentSelector.header`("부서")·`common.departmentSelector.all`("전체") `defaultValue` fallback 사용 → 정식 등록 후속
  - **A3 §4.5 sub-section 박제** + §10.1 챕터 본문 인용 매핑 행 추가
  - 후속: A3 챕터 본문 §"헤더 부서 선택" 절 추가(Phase 4), Quick Start 한 줄 anchor 추가(C1), i18n 정식 등록, "할당 없음"vs"미배정" 표기 통일

### 2026-06-04 (이사님 결정 반영 분석본 일괄 갱신, "남은 작업 1~4번")

> 2026-06-04 이사님 회의 결정 10건이 A1~A4 분석본·챕터 본문에 영향. 클러스터 A·B1 미완 작업 표(line 157~)의 시급도 순으로 분석본 갱신 진행. 각 작업 완료 후 자체 검토까지 묶음.

- **1번 — A3 [[references/spx-departments-management]] 갱신** (결정 5·6 반영)
  - §0 KC 추상화 표기 가이드 신설 (정책·사유·보존 영역·본 문서 내부 적용)
  - §1 본 문서 두 역할 명시 — 신규 챕터 분석 정전 + Dify 원본 team-members-management 변환 대체 정전 (결정 6)
  - §5 도입부 정책 박스 + §5.6 신설 — 챕터 본문 추상화 표기 예시 6개 절(도입·동작 정책·안전장치·매칭 우선순위·동기화 시점·진단 동선)
  - §8 재구성 — 8.1 결정 5·6 직접 반영 / 8.2 team-members 변환 처리 표 (원본 절별 변환 후 위치) / 8.3 잔여 컨펌
  - §10 인용 매핑 갱신 + §10.1 챕터 본문 인용 위치 명시 (외부 IdP 동기화는 §5.6 그대로 옮김 박제)
  - frontmatter status·status_history·related_decisions 신설
- **2번 — A4 [[references/spx-workspace-analysis]] 갱신** (결정 2·5·6 반영)
  - §1.1 결정 2·5·6 반영 위치 표 / §1.2 액션 분포 변화(5/29 → 6/4 초안 → 6/4 갱신) 시점별 표
  - §2 결정표 전면 갱신 — 컨펌 대기 → 6/4 초안·6/4 갱신 컬럼 분리, Team Members 변환·API Extension 3p 삭제 확정 반영
  - §3.2 Personal Settings — 결정 5 KC 추상화 적용 지침 ("외부 시스템(예, Keycloak)" 톤 박제, 보존 예외 명시)
  - §3.3 Team Members — 결정 6 변환 확정으로 재구성, A3 §8.2 변환 처리 표 인용, Phase 4 적용 방식 3단계 박제
  - §4 API Extension — 컨펌 대기 → 결정 2 outbound 일방향 재분류 후 삭제 확정 전면 갱신
  - §6 인용 매핑 — A3 §0·§5.6·§1·§8.2 인용 위치 추가
  - **자체 검토 5건 처리** — §7.2 KC 추상화 모순 해소 / §1.2 표 유지·번역 컬럼 보강 / §3.6 Plugins 사유 통일·6p로 확장 / §7 헤더 stale / §7.1 경로 혼용 분리. §9.2.1 신설로 박제
- **3번 — A2 [[references/spx-knowledge-permissions]] 갱신** (결정 2·4 반영)
  - §7 외부 연결 패턴 결정 적용 전면 갱신 — (a)(b) outbound 삭제 확정 / (c) inbound 유지·번역 확정
  - §7.2 본 챕터 영향 — (a)(b) 영향 없음, (c) Phase 4 추가 후보(API 키 권한 영향), FAQ "외부 데이터" 답변 가능 명시
  - §8.1 챕터 골격에 FAQ 답변 가능 상태 표시
  - **자체 검토 5건 + 챕터 본문 연쇄 1건 처리** — §1 "다섯" → "여섯", K5 "7종" → "유효 6종"(§5.0 신설로 UI 8/유효 6 구분), §6.1 상대경로 정정, §10.2 `[x]` 모순, §5.2 헤더 부연. 챕터 본문 `knowledge/permissions/readme.mdx`도 6종으로 연쇄 정정 + Note에 publish 추가. 빌드 통과. §10.2.1·§10.2.2 신설로 박제
- **4번 — A1 [[references/spx-app-permissions-analysis]] 정리** (결정 트랙 영향 정리)
  - frontmatter status_history·related_decisions 신설
  - §1.1 결정 1~10 영향 표 박제 — 대부분 직접 영향 없음, 결정 5는 챕터 본문에 적용
  - §13 6항목 표 형식 재구성 + §13.1 Marketplace/MCP grep 결과 박제 (Marketplace 0건·MCP §3.1 보존, 4번 작업 noop 명시)
  - §12.2 신설 — A2·A3·A4의 갱신 결과가 본 문서의 어느 절을 보강했는지 역방향 인용 매핑
  - **자체 검토 1건 처리** — §12.2 "§13.4·§13.3" 표기 → "§13 표 4번·3번 행" 정정 (§13.1 신설 절과 헷갈림 회피). §14.2.1 신설로 박제
- **5번 — A3 챕터 본문 + personal-settings KC 추상화** (결정 5 본문 적용)
  - `workspace-management/departments/readme.mdx` 9개 행 + `personal-settings/readme.mdx` 5개 행 — "Keycloak" 직접 노출을 A3 §5.6 톤("외부 시스템(예, Keycloak)" / "외부 시스템")으로 교체
  - 헤딩 "## Keycloak 그룹 동기화" → "## 외부 시스템 그룹 동기화", FAQ·SSO 옵션 라벨 일괄 추상화
  - 상호 링크 앵커 `#keycloak-그룹-동기화` → `#외부-시스템-그룹-동기화` 일관 갱신
  - **자체 검토 1건 처리** — 헤딩 "외부 IdP" 약어 → 결정 5 정책에 맞춰 "외부 시스템"으로 통일. 앵커도 동시 갱신
  - 빌드 검증 통과 (본 작업 외 broken link 없음)
- **6번 — A1 챕터 본문 명칭 갱신 + `app-permissions/` → `workspace/permissions/` 폴더 이동**
  - title "앱 권한 설정" → "권한 설정", 소개 "특정 앱" → "특정 리소스(앱·지식·도구)" / "모든 앱" → "모든 리소스" / "생성자(앱을 만든)" → "생성자(리소스를 만든)" 등 챕터명에 맞춰 톤 일반화
  - 새 위치 `ko/use-spx-agent/workspace/permissions/readme.mdx`로 작성, 이전 `app-permissions/` 폴더 삭제
  - A2·A3·A4 본문 내부 링크 4건 일괄 갱신 — `../../app-permissions` → `../../workspace/permissions` (knowledge/permissions·workspace-management/departments·personal-settings)
  - sidebars.js 카테고리 갱신 — label "앱 권한 설정" → "권한 설정", 경로 use-spx-agent/workspace/permissions/readme
  - **자체 검토 1건 처리** — 링크 텍스트 "[앱 권한 설정]" 4건 → "[권한 설정]"로 통일 (sidebar label과 일관). knowledge L8 "앱 권한 설정과 동일한 권한 모델" → "권한 설정 챕터와 동일한 권한 모델"
  - 빌드 변경분 확인 — A1 폴더 이동·sidebars·링크 모두 정상. **단 외부에서 추가된 `debug/error-type`의 `<Tabs>` MDX 문법 에러로 전체 빌드는 실패** (별도 이슈, 본 작업 범위 외)
- **7번 — Option α 적용: A3·personal-settings → workspace/ 통합 + sidebars 워크스페이스 그룹 통합**
  - **Option G → Option α 진화 반영**(CLAUDE.md §사이드바 구조). `monitor/app/`·`monitor/workspace/` sub-group 폐기, 워크스페이스 흡수 + 통계·감사 별도 top-level 그룹
  - 폴더 이동: `workspace-management/departments/` → `workspace/departments/`, `workspace-management/personal-settings/` → `workspace/personal-settings/`, 빈 `workspace-management/` 폴더 삭제
  - sidebars.js 워크스페이스 그룹 통합 — 기존 "권한 설정"·"워크스페이스 관리" 두 카테고리 → 단일 "워크스페이스" 카테고리. 순서: 사용자·부서 관리 → 권한 설정 → 개인 계정 (Option α 명시 순서). 기존 "대시보드" 카테고리 라벨 → "통계·감사" 갱신 (분석본 그룹 신설 토대)
  - 잔존 옛 경로 점검 — `workspace-management`·`app-permissions` 참조 0건
  - **자체 검토 1건 처리** — 상대 경로 단순화: `../../workspace/permissions` → `../permissions` (workspace 내부 상호 참조 2건. workspace/departments + workspace/personal-settings)
  - 빌드 SUCCESS — 본 작업 변경분으로 인한 broken link 없음 (debug/history-and-logs·knowledge/는 기존 외부 issue)
- **8번 — Option α 통계·감사 그룹 신설: dashboard/ → analytics-audit/dashboard/ 이동 + 라벨 스왑 정정**
  - 폴더 이동: `ko/use-spx-agent/dashboard/` → `ko/use-spx-agent/analytics-audit/dashboard/` (빈 dashboard/ 폴더 삭제)
  - sidebars.js "통계·감사" 카테고리 경로 갱신 — `use-spx-agent/analytics-audit/dashboard/readme`. 감사로그(audit-log) 슬롯은 주석으로 표시(B1 인터리브 작성 시 등록)
  - **라벨 스왑 오류 정정** — progress.md L273-274의 "References 후속 갱신" 두 항목의 A3/A4 라벨이 반대로 표기된 것을 정정. `spx-departments-management`=A3, `spx-workspace-analysis`=A4로 통일. 두 분석본 모두 1·2번 작업으로 갱신 완료 상태였으므로 체크박스 [ ]→[x] 처리 + stale 박스를 ✅ 정정 박스로 갱신
  - 잔존 `dashboard/` 참조 점검 — mdx·js 통틀어 0건 확인
  - 빌드 SUCCESS — 본 작업 변경분 정상
- **분석본 갱신·챕터 본문·폴더 이동·사이드바 재구성 1~8번 모두 완료** — Phase 4 본격 작성 진입 전 준비 완료. 다음은 B1 인터리브 1p 작성(`analytics-audit/audit-log/`) 또는 B2 분석

- **9번 — 정합 동기화 (progress·분석본 일괄)** — 1~8번 일련 작업 후 잔존 stale·미동기화 정정
  - **progress 챕터별 상태 표 동기화** (5건): L340 Workspace 행 "A4 갱신 필요" stale 정정, L348 권한 설정 행 경로(`workspace/permissions/`)·"구 '앱 권한 설정'" 변천사 박제, L349 사용자/부서 관리 행 경로(`workspace/departments/`)·5번 KC 추상화 명시, L350 감사로그 행 `analytics-audit/audit-log/` 슬롯 명시, L354 진행 순서 정합(A2 누락 정정, Phase 4 진행 방식과 일치)
  - **conventions 글로서리 후속 정합** (3건): progress L129·L232 [ ]→[x] 분리 처리, A1 §14.3·B1 §11 conventions 갱신 [x] 처리. progress L218 "권한 모델·로그인 옵션·감사 로그 3 sub-section 흡수" 완료 박제와 정합
  - **죽은 경로 정정** (4건): A3 §10.1 `workspace-management/...` → `workspace/departments/`, A3 §12.3 5번·7번 완료 처리·결과 박제, A4 §7.1·§9.3·§9.4 동기화 갱신
  - **Option G 라벨 통일** (4건 정정 + 3건 변천사 박제 보존): A1 §1.1·§14.2.1, A4 §3.3·§9.4 stale 정정. A3 §12.3·A4 §9.3·A1 §14.2.1 본문의 "G→α 진화" 표기는 변천사 박제로 의도된 보존
  - **A2·A4 분석본 메모 정합**: A2 §8.2 내부 링크 메모 갱신, A4 §9.3 별도 작업 추적 3~8번 모두 [x] 처리·결과 박제
  - **B1 후속 deferred 정합**: "데이터셋 vs 지식 표기 통일"·"conventions 글로서리 일괄 갱신" [x] 처리, KC 추상화·인터리브 1p 항목 별도 명시
  - 빌드 SUCCESS — 본 동기화 작업 변경분 정상
  - 진척표·분석본 5개 모두 1~8번과 정합. 다음은 B1 인터리브 또는 B2 분석

### 2026-06-04
- **C1 완료 — Get Started 3p (간소화 분석 + 인터리브 3p)** → [[references/spx-get-started-analysis]]
  - 간소화 분석본 작성 — 신규 코드 deep dive 불필요(A1·A3·대시보드·B1 인용). 3p별 처리 지침(전역 규칙·결정 적용 지점 / 티저 4종 / 로그인·앱 생성 한 줄 / 부서·RBAC 정의)·인용 매핑·챕터 골격 박제
  - **인터리브 챕터 3p 작성** — `ko/use-spx-agent/getting-started/{introduction,quick-start,key-concepts}.mdx`, `sidebars.js` "시작하기" 카테고리 신설(최상단), 빌드 통과
  - **⚠️ 이슈 발견·수정 1건 — `<Card href>` 상대경로 취약성**: `Card.jsx`가 raw `<a>` 태그를 렌더 → ① Docusaurus 깨진 링크 검사 **우회**(introduction이 broken 목록에 안 뜬 진짜 이유 = 검사 자체를 안 함, "링크 정상"이 아니었음), ② `trailingSlash` 미설정 환경에서 상대경로(`./`·`../`)가 서빙 URL에 따라 깨질 수 있음. 원본 Dify와 동일하게 **절대경로**(`/use-spx-agent/...`)로 교정 후 재빌드 통과. (※ 마크다운 링크 `[..](../..)`는 Docusaurus가 검증하므로 quick-start·key-concepts는 무관)
  - **한계 (의도적, deferred 박제)**:
    - Quick Start 2단계 워크플로우 빌드 본문은 노드 요약만 — 전체 번역은 유지·번역(P2)으로 분리(C1은 차이 검증에 집중)
    - key-concepts 깨진 링크(`publish/publish-mcp`·`nodes/user-input`·`nodes/trigger/overview`) = 아직 포팅 안 한 페이지 미리 링크. 기존 누적 패턴과 동일, warn 레벨
    - introduction 튜토리얼 카드 = `tutorials/` 미포팅이라 dangling(raw anchor라 빌드 경고도 없음) → Phase 4 튜토리얼 작성 후 유효화
    - 원본 제목 "30-Minute Quick Start"에서 "30분" 제거(Cloud 튜토리얼 분량 결부 표현) — 명시 결정은 아님, 본격 작성 시 재확인 여지
  - 정확성 검증: RBAC 역할 표 ↔ A1 §7.3, 로그인 부서 자동 배정 ↔ A3 §5.4/§5.6 부합. KC 추상화(전역 규칙 #6) 일관 적용. **클러스터 C 종료 — 사전 분석 클러스터(A·B·C) 전부 완료**
- **B1 마무리 완료 — KC 추상화 + 인터리브 챕터 1p** (분석 완료 후 후속 일괄 처리)
  - **분석본 KC 추상화** (전역 규칙 #6) — §1.1 표기 가이드 신설. 분석본은 정전이라 코드/메타 식별자(DB·JWT·컨테이너·함수명)는 §0.3 보존 영역 따라 유지, 사용자 노출 묘사 문장(§4.1 "사용자" 열·검색 절)만 추상화 + 챕터 인용 환기
  - **인터리브 챕터 1p 작성** — `ko/use-spx-agent/analytics-audit/audit-log/readme.mdx`. 소개·접근 권한(owner/admin)·2탭·이벤트 조회/필터·상세·기록되는 활동(카테고리별)·내보내기·시스템 로그·참고(지연·보존·범위) 구성
  - **챕터 본문 KC 완전 추상화** — "Keycloak" 직접 노출 0건. "외부 시스템에서 보강" / "사용자 이름·이메일로 검색" / 보안 이벤트 "서버 연동 구성에 따라 수집" 톤
  - **deferred 함께 해소** — 시스템 로그 탭은 짧은 1개 절(운영 점검용)로 비중 축소, "데이터셋=지식" 안내 `<Note>` 박음
  - **sidebars.js 등록** — "통계·감사" 카테고리 슬롯 주석 → 실제 항목 교체. `npm run build` 통과(신규 broken link 없음 — 기존 미작성 Phase 4 페이지 링크만 잔존)
- **B1 검토 — 코드 재대조 발견 1건** (2026-06-04)
  - ⚠️ **Export 상한 불일치** — 다이얼로그는 "최대 50,000건"이라 표시하나, `ExportButton.buildUrl`이 `limit` 파라미터를 전달 안 해 export 라우트 기본값 **10,000건**에서 잘림(잘림 경고 없음). 50,000은 URL에 `?limit=50000` 수동 입력 시만. 분석본 §9에 ⚠️ 박제, 챕터 본문은 "최대 50,000" 단정 제거 → "상한 있으니 기간 나눠 내보내기" 안내로 정정. 개발팀 전달 후보(버튼 limit 전달 또는 문구 정정)
  - 나머지 검토 항목(진입점·탭·열·필터·카테고리·격리·보존·수집 주기)은 코드와 일치 확인
  - **클러스터 B 남은 작업: B2 폐기 (2026-06-04 재고)** — Option α 그룹 분리(모니터링 / 통계·감사)로 사용자 혼동 가능성 ↓, 별도 분석본·3축 비교표 ROI 낮음. Monitor Analysis는 단순 부분 수정(제목 "Analysis" → "모니터링" 통일)만 Phase 4 본문 작성 시 처리. **클러스터 B 종료, 다음 클러스터 C(Get Started)로 이동 가능**
- **B1 분석 완료 — 감사 로그 (dify-audit deep dive)** → [[references/spx-audit-log-analysis]]
  - **별도 앱 구조 확인** — 감사 로그는 Dify 본체가 아니라 `dify-audit`(독립 Next.js 16 앱). Dify 설정 화면에 **iframe 임베드**(설정→감사 로그 탭, owner/admin만), 같은 origin으로 access_token 쿠키 전달
  - **수집 5경로 박제** — ① DB 폴링 13수집기(dify_db, 5분 cron, cursor 기반) ② PostgreSQL 트리거 6종(pg_trigger, DELETE·역할변경 — 폴링 사각 보완) ③ nginx 와처 3종(보안 401/403/429·api_call) ④ self-audit 6종(감사 열람·내보내기 자체 기록) ⑤ 시스템 로그 일배치(별도 `spx_system_logs`, 새벽 1시 docker logs)
  - **통합 테이블** `spx_audit_events` 단일 — category(admin/user/security)·action·actor·target·details(JSON)·tenantId·source. 대시보드(B2) 집계용 생성컬럼·mat view 3종도 확인
  - **이벤트 카탈로그 전수** — action별 한국어 라벨은 `audit-meta.ts`가 사전. §10에 카테고리·행위자유형·액션 라벨 전수 박제(conventions 글로서리 반영 대상)
  - **UI 박제** — 2탭(이벤트/시스템 로그), 목록 7열+비고 요약, FilterBar(카테고리→액션 동적·기간·검색은 Keycloak 사용자까지), 상세 4섹션+원본 JSON, Export(CSV BOM/JSON·현재 필터·최대 5만건)
  - **권한·격리** — owner/admin only(이중 체크), 테넌트 격리(adminTenantIds OR NULL 글로벌), 직접 URL 차단(iframe navigation만 허용)
  - **§8 3축 영역 구분** 박제 — 대시보드(워크스페이스 KPI) / 모니터링(앱 단위) / 감사 로그(이벤트) — B2가 3축 비교표로 인용
  - 후속 deferred: 시스템 로그 탭 비중 / "데이터셋"vs"지식" 표기 통일(A2) / nginx 배포 의존 톤 / **KC 추상화(전역 규칙 #6)** 본 분석본에도 적용 필요 / 인터리브 1p
  - **클러스터 B 진행 상태** — 6/4 재고 결과 B2 폐기, B1만으로 클러스터 B 종료. Monitor Analysis는 Phase 4 본문 작성 시 단순 부분 수정으로 처리
- **A4 분석 완료 — Workspace 11p 전수 재검토** → [[references/spx-workspace-analysis]]
  - 5/29 가정 오류 2건 정정: Personal Settings(KC SSO 기반 → 실제 4종 옵션), Team Members(삭제 → A3로 위임)
  - 5/29 "유지·번역" 2건 → 부분 수정 격상(App Management·Model Providers — SaaS 분기 잔재)
  - 11p 액션 분포: 부분 수정 5p / 삭제 3p / 컨펌 대기 3p(API Extension)
  - A1~A3 인용 매핑 박제 — Workspace 페이지의 권한·부서 관련 콘텐츠는 모두 A1~A3로 위임 가능
  - **인터리브 챕터 1p (대표)**: `ko/use-spx-agent/workspace-management/personal-settings/readme.mdx` 작성·등록·빌드 통과 — 5/29 가정 정정 실효성 검증
  - **클러스터 A 완료** — 다음 클러스터 B(B1 감사 로그)로 이동
- **A3 분석 완료 — 사용자/부서 관리 (운영 흐름)** → [[references/spx-departments-management]]
  - 부서 페이지(`departments-page/`)·멤버 페이지(`members-page/`)·멤버 모달·부서 셀 UI 코드 전수 검증
  - 백엔드 `department_service.py` 5개 메서드, `account_service._sync_department_from_keycloak_groups` 정책 검증
  - **A1 §C4 정정 박제** — HANDOVER에 "Keycloak 동기화 가산형(additive)"이라 적혔으나 코드는 **mirror(wholesale replace)** 정책. 그룹에서 빠지면 다음 로그인 시 부서 멤버십 자동 제거. A1 산출물 갱신 완료
  - A1·A2 모호 동선(DENY 진단·지식 검색 권한 재동기화)의 정답 위치 매핑 — A3가 클러스터 A의 운영 동선 허브 역할
  - 사용성 함정 박제 — 멤버 모달 "제거"는 모든 부서 해제(부서별 부분 제거는 부서 셀 set으로만), 부서 편집·삭제 버튼 hidden 처리 정책 안내
  - **인터리브 챕터 1p**: `ko/use-spx-agent/workspace-management/departments/readme.mdx` 작성·`sidebars.js` "워크스페이스 관리" 카테고리 등록·빌드 통과
  - 다음 단계: A4(Workspace 11p 전수 재검토) — Personal Account 정정, API Extension 정책, 전역 규칙 5개 적용
- **A2 분석 완료 — Knowledge 권한 (도메인 특이사항)** → [[references/spx-knowledge-permissions]]
  - A1 코어를 정전(canonical)으로 두고 차이점 6건(K1~K6)만 박제: 네이티브 동기화 / creator 항상 포함 / view+ALLOW 투영 / drift 검증 / 액션 7종 / 원본 통합 위치
  - `dataset_acl_sync_service.py` 전수 검증, `_project()` 매핑·INV-8/9 확인
  - `dataset-permissions/` UI는 App과 거의 동형 — 권한 부여 모달은 App `grant-permission-modal.tsx`를 그대로 import (액션 8종 노출, duplicate는 효과 없음)
  - 원본 Knowledge 3p 권한 언급 grep — `manage-knowledge/introduction.mdx`만 명시 행 1개, 나머지 2p는 신규 추가 — 3p별 통합 위치·1문단 안내 문안 작성
  - **인터리브 챕터 1p**: `ko/use-spx-agent/knowledge/permissions/readme.mdx` 작성·등록·빌드 통과
  - 후속 deferred 3건 (파이프라인 가시성 UI / "운영자" 호칭 통일 / 외부 KB 컨펌 반영) — Phase 4 본격 작성 시
  - 다음 단계: A3(사용자/부서 관리) — A1·A2 권한 모델 링크 + 부서 CRUD UI + KC 그룹 sync

### 2026-06-02
- **A1 분석 완료 — 권한 설정 (권한 모델 코어)** → [[references/spx-app-permissions-analysis]]
  - HDD·HANDOVER 정독 + 코드 9개 영역 검증
  - HDD↔코드 차이 9건(C1~C9) 정리. 신규 식별 2건: C5(워크스페이스 RBAC 3탭 분리), C7(부서 편집/삭제 버튼 hidden)
  - 사용자 매뉴얼 어휘 재정리 (도메인 모델 / 가시성 4단계 / ACL / 소유권 / 권한 판정 4단계 / 역할별 capability 매트릭스 / UI 진입점)
  - A2~A4 인용 매핑 박제 (Knowledge·부서 관리·Workspace가 본 문서를 정전으로 참조)
  - **인터리브 챕터 1p**: `ko/use-spx-agent/app-permissions/readme.mdx` 작성·`sidebars.js` 등록·빌드 통과
  - 작성 중 누락 보강: §5.4(사용성 제약), §9.2(권한 메뉴 vs 워크스페이스 권한 탭 가시성 메커니즘 구분), §11(한국어 매핑 박제 — 가시성·액션·주체·효과)
  - 다음 단계 후보: A2(지식 권한) ∥ A3(부서 관리) 병렬
- **호스팅 방향 결정: Docusaurus 전환 확정** — Mintlify 옵션(Cloud/Enterprise) 폐기
  - 사유: 사내망 환경에서 외부 GitHub 의존 회피, 자체 호스팅 안정성, 라이선스 비용 0
  - 결정 박제 → [[decisions]] "2026-06-02 — Docusaurus 전환 확정" 섹션
  - Mintlify Cloud 배포 에러(navigation 호환성) 해소 작업 중단·폐기
- **Phase 3.5 신설** — Mintlify → Docusaurus 전환 실행 단계를 Phase 4(본격 포팅) 앞에 배치
  - 작업 항목 15건 + 권장 순서(시나리오 A → B) 진척표에 박음 (8건 → 검토 후 15건 확장)
  - 작업량 산정: ~4.5~6일 (감사 보고서 4~5.5일 + 추가 항목 ~0.5~0.75일)
  - 추가 항목: static 에셋 배치, versions/ 처리, GitHub 연동 정리, ja/zh/en 방침, scripts/tools 정리, 문서 갱신, Mintlify 잔재 정리
  - ~~후속 갱신 필요~~ → Phase 3.5 체크리스트 "문서·설정 갱신" 항목으로 흡수
- **Phase 3.5 전환 실행** (문서·설정 갱신 전까지):
  - Docusaurus 프로젝트 루트 구성 완료 (`package.json`, `docusaurus.config.js`, `sidebars.js`)
  - `"type": "module"` 시도 → webpack `require.resolveWeak` 충돌 → CJS 유지, 마이그레이션 스크립트만 `.mjs`
  - MDXComponents 래퍼 6종 + CSS 작성, ko/ 2파일 빌드 성공 확인
  - 변환 스크립트 3종 작성 (`tools/migrate/`), ko/ 파일에 프론트매터·링크 변환 적용
  - 검색 플러그인 `@easyops-cn/docusaurus-search-local` 동작 확인 (인덱스 생성)
  - versions/ 보존, scripts/tools 원본 보존
  - origin(`mjlee-spelix/spx-agent-demo`) 리모트 제거, Mintlify Cloud 수동 해제 필요
  - 최종 빌드: `npm run build` 성공 (broken links는 Phase 4 페이지 추가 시 해소)
- **Phase 1.3 매트릭스 재검토 통과** (9개 섹션 102p 전수 재검토)
  - 전역 규칙 5개 신설 ([[scope-mapping]] §전역 적용 규칙): SaaS 플랜 제거 / Cloud 분기 통합 / Sandbox 용어 명확화 / 자체 호스팅 분기·env var 추출 / 외부 Dify 리소스 (GitHub·channel 제거 / Marketplace 보류 / 기능 확장 추출)
  - 확정 삭제 5건 → 7건 (web-app-access Enterprise tag, rate-limit Cloud quota 추가)
  - 부분 수정 격상: Knowledge 3개 (Create Intro, Create Pipeline, Manage KB Settings — 권한 설정 섹션 추가)
  - 다운그레이드: nodes/llm, workspace/model-providers vLLM 언급 제거 (사용자 백엔드 일반화)
  - 비고 정밀화: Get Started 3개 (KC SSO 가정 → 사용자명/비번 폼), Monitor Analysis (제목 → "모니터링" UI label 통일), Publish Overview (Marketplace 처리 보류로 톤다운)
  - 신규 챕터 5 → 4 (설정 사이드바 제거, KAN-29는 부수 기능이라 기존 페이지 부분 수정에서 한 줄 언급)
  - 검토 묶음 4 → 6 세분화 (외부 SaaS trace 송출 / 외부 import 패턴 a / 외부 KB read-only 패턴 b / 외부 노출 API 패턴 c)
  - 용어 글로서리 확장 ([[conventions]]): Monitoring 행에 "Analysis 페이지 = 모니터링" 명시, Dashboard 행 신규 추가 (앱별 모니터링과 워크스페이스 KPI 영역 구분)
  - Workspace 섹션 전체 ⚠️ 재검토 필요 (전역 규칙 적용 + 소스 분석, deferred)
  - 데일리 노트 deferred 5건 추가: deployment-config-extracts / feature-extension-extracts / dify-edition-comparison / Marketplace 컨펌 / Workspace 재검토 / 지식 권한 소스 분석
- **Phase 4 사전 분석 클러스터링** — 7개 분석 항목을 코드 영역·도메인 기준 3개 클러스터로 재구성, 인터리브 방식 채택
  - **클러스터 A (권한·RBAC, 4건)**: A1 앱 권한 코어 → A2 지식 권한 ∥ A3 부서 관리 → A4 Workspace 재검토. A1에서 권한 모델(가시성 4단계·ACL·소유권) 정립 후 A2~A4는 차이점만 추가, 산출물 간 [[링크]]로 중복 제거
  - **클러스터 B (관찰성)**: B1 감사로그만 진행. B2는 6/4 재고로 폐기 (Option α 그룹 분리로 3축 비교 필요성 ↓). Monitor Analysis는 Phase 4 본문 작성 시 단순 부분 수정
  - **클러스터 C (진입점, 1건)**: C1 Get Started — A·B 완료 후 키워드·정의 안정 시점에 마무리
  - **인터리브**: 분석 완료 → 챕터 1p 임시 작성 → 누락 발견 시 분석 보강 → 다음 분석 (분석 누락 발견 빈도 ↑)
  - 컨펌 트랙(이사님 답변 후 처리 8건)은 별도 트랙, 영향 받는 클러스터(A2·A4·B2)에 반영

### 2026-06-02 (Phase 3.5 문서·설정 갱신)
- **Phase 3.5 문서·설정 갱신 4건 완료**:
  - `conventions.md` — MDX 포맷 규칙 Docusaurus 반영 (MDXComponents 래퍼 6종, `sidebar_label`, 상대경로 링크, `sidebars.js` 네비게이션), 빌드·개발 환경 섹션 추가
  - `.claude/CLAUDE.md` — 보존 대상 표 갱신 (`docs.json` read-only 이동, 신규 추가 영역 표 6항목 신설), 빌드 도구 섹션 확장 (개발 명령어·설정 파일 표·전환 원칙), 외부 리소스 origin 제거 반영, 덮어쓰는 규칙 `sidebars.js` 반영
  - 루트 `README.md` — SPX Agent Docs 전면 재작성 (Docusaurus 기술 스택, Phase 7단계, 리포 구조, 커밋 컨벤션, 라이선스)
  - `NOTICE.md` — CC BY 4.0 변경 사항 추가: Docusaurus 마이그레이션, `ko/` 신설, 브랜딩 교체, MDXComponents 래퍼, 검색 플러그인, `README.md` 재작성

### 2026-06-04
- **이사님 회의 결정 10건 일괄 박제** → [[decisions#2026-06-04 — 이사님 회의 결정 일괄 박제]]
  - Marketplace + 플러그인 전체 제거 (모델 제공자는 유지)
  - 외부 연결(outbound) 제거: Monitor 7건, 지식 외부 import 3건, 외부 KB 2건, Twitter, API Extension 3건 — 총 14건 추가 삭제
  - MCP 유지·번역 확정
  - inbound 3건(Embedding / Developing APIs / Maintain via API) 유지·번역 확정
  - 전역 규칙 #6 신설 — KC 추상화 ("외부 시스템(예, Keycloak)" / "관리 시스템" 표기, 모든 페이지 본문 적용)
  - workspace/team-members-management 변환 (사용자·부서 관리 신규 챕터로 대체)
  - CI/CD 표시 가이드 (개발 환경 한정 기능 별도 마커)
  - 문서는 전체 사용자 대상, CI/CD만 환경 구분
  - **"앱 권한 설정" → "권한 설정" rename** (A1 분석 결과 앱·지식·도구 공통 권한 모델로 확정 → "앱" 한정 표기 제거)
  - **사이드바 구조 Option α 채택** (당일 G에서 진화) — Workspace 흡수(권한 설정·부서 관리) + **통계·감사 별도 top-level 그룹 신설**(대시보드·감사로그). 모니터링 그룹은 앱 단위만 평면. G 시점 "워크스페이스 단위" sub-group 작명이 모호하다는 우려로 별도 도메인 분리로 정정
- **매트릭스 갱신 일괄 반영**:
  - 확정 삭제 7건 → 23건 (Marketplace/플러그인/외부 연결 결과)
  - 변환 1건 신설 (team-members-management)
  - 검토 27건 → **0건** (모든 액션 확정, Phase 4 본격 포팅 진입 가능)
  - 챕터명 일괄 rename: "앱 권한 설정" → "권한 설정" (scope-mapping / progress / decisions / conventions / CLAUDE.md / references 분석본)
- **컨펌 자료 [[references/external-connection-framing-proposal]] 가결과 반영 완료** — 더 이상 활용 안 함, 참고용 보존
- **후속 작업**:
  - 폴더 구조 박기 (workspace/permissions/, workspace/departments/, analytics-audit/dashboard/, analytics-audit/audit-log/) — Monitor는 평면 유지
  - 기존 페이지 이동 (dashboard/, app-permissions/)
  - `sidebars.js` Option α 구조 반영 (Workspace 흡수 + 통계·감사 top-level 신규)
  - References 분석본 일부 갱신 (A3 spx-departments-management의 KC 추상화 박제, A4 spx-workspace-analysis의 범위 축소 반영)

### 2026-06-08 (Docusaurus 테마 — Mintlify pixel-perfect 복제)
- **목표**: Dify docs (Mintlify maple 테마) 렌더링을 Docusaurus에서 동일하게 재현
- **분석 방법**: `docs.dify.ai` quick-start / error-type 페이지 HTML을 curl로 저장 → Tailwind 클래스에서 px 값 전수 추출 → 빌드 결과물 HTML과 1:1 비교
- **`src/css/custom.css` 전면 재작성**:
  - Gray 팔레트: Tailwind 기본 → Mintlify 커스텀 (`--mint-gray-*`, 예: gray-50=#F2F5FA)
  - 레이아웃 grid 구조: Docusaurus 기본 75%/25% → **본문 flex-grow + TOC 고정 304px** (Mintlify `xl:w-[calc(100%-28rem)]` + `w-[19rem]` 복제)
  - 콘텐츠 max-width: 컨테이너 672px 캡 제거 → article만 672px (본문 텍스트 폭 제한)
  - 사이드바→본문 간격: 48px → 64px (pl-16)
  - Steps CSS: 12px 파란 dot → **28px 회색 원 + 번호 + 1px 수직선 + 마지막 gradient fade**
  - Callout CSS: neutral 단일 → **타입별 5색** (info=neutral, note=blue, tip/check=green, warning=yellow)
  - Tabs CSS: gap 24px, 배경 hover 제거 → border-bottom만, radius 제거
  - Accordion CSS: padding 16/20px, hover 배경, title font-weight 500, 콘텐츠 mx-24px
  - Frame CSS: grid 도트 배경(::before) + overlay border(::after) + 이미지 radius 12px 추가
  - 코드 블록: 라이트 모드 밝은 배경 (Prism github 테마), margin 조정
  - Pagination: border 제거 → hover ring, h-64px, rounded-12px
  - TOC: Mintlify 동일 스타일 (border-left, padding, active 색상)
  - 사이드바: padding 28/24px, 그룹 간격 2rem, active text-shadow faux-bold
  - Typography: eyebrow→H1 gap 10px, H1 30px !important, 링크 underline 기본
  - 코드 폰트: JetBrains Mono 추가
- **컴포넌트 JSX 수정/신규**:
  - `Steps.jsx` 재작성 — 자동 번호 매기기, 28px 원형 (`mintlify-step-number__circle`), title `<h4>`→`<p>` (TOC 미잡힘)
  - `Admonitions.jsx` 재작성 — 타입별 CSS 클래스 (`mintlify-callout--info/note/tip/check/warning`), 원본 SVG 아이콘 (크기 타입별 차이 반영)
  - `Accordion.jsx` — caret 16→12px, **위치 오른쪽→왼쪽** (Mintlify 동일)
  - `DocItem/Content/index.jsx` **신규** — DocItem/Content swizzle로 H1과 MDX body 사이에 description + copy 버튼 삽입 (SSR 호환, display:none hack 제거)
  - `DocItem/Layout.jsx` — eyebrow만 남기고 description/copy 로직 Content로 이관
  - `TOC/index.jsx` — pass-through (잘못 추가한 "이 페이지에서" 헤더 삭제)
- **`docusaurus.config.js`**: Prism 테마 light=github, dark=dracula 설정
- **분석 산출물 저장**:
  - `.claude/docs/references/mintlify-quick-start-page.html` — curl 저장
  - `.claude/docs/references/mintlify-error-type-page.html` — 기존
- **남은 차이** (의도적 미적용 또는 Docusaurus 구조 한계):
  - 사이드바 그룹 아이콘 (FontAwesome mask-image) — Docusaurus 기본 미지원, 별도 swizzle 필요
  - Copy 버튼 Mintlify는 H1 옆 inline (split button), 우리는 H1 아래 (단독 버튼)
  - 이미지 zoom (react-medium-image-zoom) — Mintlify 전용

### 2026-06-08 (스타일 정밀 정합 + 헤더 구조 버그 수정)

> 브라우저 자동화(Claude in Chrome)로 원본 Mintlify(3001)와 Docusaurus(3000)를 나란히 띄우고 **computed-style 실측 diff** 방식으로 정합. 절차·기준값 박제: [[references/mintlify-to-docusaurus]] §14 / [[references/mintlify-style-baseline]]

**1) computed-style 실측 정합 — `custom.css` 15개 항목** (전부 양쪽 실측 비교 후 재측정 검증)
- 라이트: 본문 max-width 672→**576px**(2xl만 672) / 제목 line-height(h1 36·h2 32·h3 28)·margin / 콜아웃 16·28 / inline code(weight 500·border 제거·반투명 bg·pl 8) / 콜아웃 글자색 #3E4146 / 사이드바 메뉴색 #6F7277 / 코드블록 2겹 배경(외곽 #F2F5FA + 내부 #fff·radius14)
- 다크: 본문 #DEE1E6 / 제목 #FFFFFF / 콜아웃 #9EA1A6 / inline code bg 0.05 / 코드 내부 #0B0C0F
- **교훈**: curl HTML·Tailwind 클래스 px 추출·추정은 구조적으로 틀림(JS 미실행·토큰 의존·상속/pseudo 누락). **살아있는 DOM의 getComputedStyle 실측 diff**가 정답
- 보류(사유 baseline 기록): 사이드바 폭 4px·제목색 미미차·링크/TOC 표본 부족·다크 배경색(원본은 `background-color` 미사용으로 측정 불가)

**2) 헤더 구조 버그 2건 — 사용자 화면 비교로 발견** (template 노드 페이지에서 eyebrow·페이지 복사 버튼 누락)
- **eyebrow 누락**: `DocItem/Layout.jsx`의 `CATEGORY_LABELS`에 `nodes`/`build`/`publish`/`tutorials` 키 누락 → 해당 카테고리 페이지에서 eyebrow `null`. 4개 키 추가로 해결
- **copy 버튼+description 누락**: `DocItem/Content` swizzle(코드는 정상 — `doc-header-inject` 항상 렌더)이 **dev 서버에 미반영**. 새 swizzle 파일은 HMR 안 되고 **재시작 필요**. `npm start` 재시작으로 해결
- 검증: template 페이지에서 eyebrow "노드" / "페이지 복사" 버튼 / description 모두 복구 확인 ✅
- **교훈**: computed-style 측정은 본문 요소(h1·p·콜아웃·코드) 셀렉터만 봐서 **페이지 헤더 구조(eyebrow·copy)를 놓침** — 스타일 정합과 별개로 구조 렌더 점검 필요. theme swizzle 변경(특히 신규 파일)은 **dev 서버 재시작 필수**
  - 반응형 max-width (576px→672px) — 고정 672px로 단순화
  - 폰트 CDN 의존 (사내망 배포 시 self-host 전환 필요)

### 2026-06-10 (Knowledge 2차 배치 — 나머지 11p 작성)

Knowledge 그룹 전체 완료. 파일럿 2p + 1차 배치 4p + **2차 배치 11p** = 17p.

**작성 내역:**
- **Pipeline 6p**: readme(개요), create-knowledge-pipeline(부분 수정), knowledge-pipeline-orchestration, publish-knowledge-pipeline, upload-files, manage-knowledge-base
- **Manage 3p**: maintain-knowledge-documents, introduction(설정 관리, 부분 수정), maintain-dataset-via-api
- **Standalone 2p**: metadata, integrate-knowledge-within-application

**전역 규칙 적용:**
- #1 SaaS 플랜: Publish Pipeline의 Sandbox plan 제한 Warning 삭제, Maintain Documents의 Dify Cloud 자동 비활성화 Note 삭제, Maintain Documents의 유료 기능 Info 삭제
- #2 Cloud 분기: 없음
- #4 Self-hosted 분기/env var: Orchestration의 Summary Auto-Gen "self-hosted only" 분기 제거, Orchestration/Maintain Documents의 env var(`ATTACHMENT_IMAGE_FILE_SIZE_LIMIT`, `SINGLE_CHUNK_ATTACHMENT_LIMIT`) 본문에서 제거
- #5 Dify 브랜딩: 전 페이지에서 "Dify" 브랜드명 제거 (Dify Extractor→내장 추출기, Dify Marketplace 링크/안내 삭제, Dify DSL→DSL, docs.dify.ai API 참조 링크 삭제)
- #8 문장 적합성: Orchestration의 plugin 개발 Tip 삭제, Marketplace 도구 탐색 Tip 삭제, authorize-data-source 링크 삭제

**부분 수정(§A) 적용:**
- Create Pipeline: 하단에 권한 설정 Tip 추가 (`../../workspace/permissions/`)
- Orchestration Step 5: 하단에 권한 설정 Tip 추가
- Manage KB Settings: 하단에 권한 설정 Tip 추가

**sidebars.js 갱신:**
- 지식 파이프라인(6p), 지식 관리(3p), metadata, integrate 등록
- 빌드 SUCCESS (신규 페이지 broken 0건, 기존 broken 2건은 nodes 페이지의 앵커 문제)

**글로서리 추가 용어**: i18n 검증 기반 — 지식 파이프라인, 빈 지식 파이프라인, 오케스트레이션, 데이터 소스, 입력 필드, 전역 입력, 고유한 입력, 아카이브, 이름 바꾸기, 요약 생성 등 pipeline.json·dataset-pipeline.json·dataset-documents.json에서 추출

**체크리스트 보정:** Card href 5건(Pipeline readme) 상대→절대경로 변환 (§3.1 Card.jsx broken link 검사 우회 방지)

**상태:** 본문 작성 완료 → **사용자 최종 검수 대기**

### 2026-06-11 (Workspace 그룹 3p 작성)

원본 포팅 마지막 그룹. 총 6p: 신규 3p + 기존 인터리브 3p (departments·permissions·personal-settings).

**신규 작성 내역:**
- **Overview** (readme.mdx, 부분 수정): 워크스페이스 구조도(Billing 삭제), 단일 워크스페이스 정책, 6종 역할 테이블(빌더 추가), 부서·권한 소개, 설정 메뉴 안내 표
- **Model Providers** (model-providers.mdx, 부분 수정): System Providers 섹션 전면 삭제(SaaS), Custom Providers 중심 재구성, 모델 자격 증명 관리(사전 정의/커스텀 탭), 로드 밸런싱(SaaS callout 삭제), Access/Billing 섹션 삭제, Troubleshooting 유지
- **App Management** (app-management.mdx, 부분 수정): 앱 정보 편집, 복제, DSL 내보내기/가져오기(SaaS/Community 분기 삭제), 앱 삭제, 권한 섹션 1문단 추가

**전역 규칙 적용:**
- #1 SaaS: Model Providers — Load Balancing 유료 기능 callout 삭제, Access/Billing 섹션 삭제, System Providers 삭제
- #2 Cloud/CE 분기: App Management — DSL 가져오기의 SaaS users/Community users 분기 삭제
- #5 Dify 브랜딩: 전 페이지에서 "Dify" → 일반화 또는 "spx-agent"
- #8 문장 적합성: Model Providers — "Cost Optimization" 시나리오에서 free/low-cost quotas 언급 삭제

**기존 인터리브 확인:**
- Personal Settings: A4 §7에서 작성 완료. 전역 규칙·직역체 점검 → 변경 불필요
- 사용자/부서 관리: A3에서 작성 완료
- 권한 설정: A1에서 작성 완료

**직역체 검수:** grep M1("~을 위한") 1건 발견 → "프록시를 위한"→"프록시용" 수정
**전역 규칙 잔존 grep:** 신규 3p — 0건 (Dify/SaaS/Keycloak/KC SSO 잔존 없음)

**i18n 발견:**
- **빌더(Builder)** 역할 신규 발견 (`members.builder`). 편집자와 일반 사이. 글로서리에 추가
- 설정 메뉴 라벨 7건 추가 (모델 제공자·사용자 관리·부서 관리·언어·내 계정·작업 공간·대시보드)
- Normal "일반 멤버"→"일반" i18n 교정
- **deferred 3건**: view="보기"/transfer="양도"/grant="권한 추가" vs 글로서리 차이 — 권한 챕터 검수 시 확인 필요

**sidebars.js:** 개요·모델 제공자·앱 관리 3건 등록. 빌드 SUCCESS (신규 broken 0건)

**체크리스트 전수 검토 (사용자 검수 전):**
- 사용자 요청으로 §1~§3 + §A 체크리스트 항목 전수 대조
- **누락 3건 발견·보완**:
  1. **§2.4 3패스 원문 대조** — Model Providers 원본 Access/Billing 섹션의 API 키 보안 Warning이 섹션 삭제 시 함께 누락. `<Warning>` 추가: "API 키는 워크스페이스 전체에서 모델 접근에 사용…관리자에게만 부여"
  2. **§2.5 i18n 사후 검증** — Model Providers 본문에 영어 UI 라벨 6건이 한국어 미교체: `Manage Credentials`→자격 증명 관리, `Add credential`→자격 증명 추가, `Add Model`→모델 추가, `Save`→저장, `Load balancing`→로드 밸런싱, `Specify model credential`→모델 자격 증명 지정
  3. **§A 보강 — scope-mapping "관리자 대시보드 언급"** — Overview 네비게이션 표에 대시보드 행 미포함. 메인 내비에 대시보드 행 추가 + 설정 메뉴 표에 대시보드·감사로그 2행 추가 + Note에 관리자 전용 명시
- 보완 후 빌드 재검증 SUCCESS (신규 broken 0건)

**상태:** Workspace 그룹 완료 → **원본 포팅 9그룹 전체 본문 작성 완료** → 사용자 검수 대기
