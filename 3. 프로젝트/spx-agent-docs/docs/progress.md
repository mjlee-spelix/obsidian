# 진행 상황

## 📍 현재 상태 한눈에 (2026-06-15)

> **이 블록만 읽으면 현황 파악 OK.** 세부는 아래 섹션, 과거 로그는 [[progress-archive]] 참조.
> ⚠️ 세션 종료 시 이 블록을 가장 먼저 갱신할 것 (날짜·상태·다음 할 일).

- **프로젝트**: Dify 문서 → Docusaurus 한국어 포팅 (spx-agent-docs)
- **현재 Phase**: **Phase 4 본격 포팅 진행 중** (Phase 0~3.5 완료 / Phase 5 스크린샷·6 배포 대기)
- **원본 포팅 9그룹 전체 본문 작성 완료 — Workspace 3p 신규 + Knowledge 17p 검수 대기**

| 그룹                                                                  | 상태                                        |
| ------------------------------------------------------------------- | ----------------------------------------- |
| Get Started · Nodes · Build · Debug · Tutorials · Monitor · Publish | ✅ 본문 완료                                   |
| Knowledge                                                           | 📝 본문 작성 완료 (17p) — **사용자 검수 대기**         |
| Workspace                                                           | ✅ 본문 완료 (6p: 개요·모델 제공자·앱 관리 + 기존 인터리브 3p) |
| (신규) 권한설정 · 사용자/부서                                                    | ✅ 완료 (인터리브 작성, Workspace 그룹에 포함)          |
| (신규) 대시보드 · 감사로그                                                       | ✅ 본문 확장 완료 (2026-06-12, §C 3단 해설)           |

**▶ 다음 할 일 (우선순위순)**
1. **사용자 검수 대기** — Knowledge 2차 배치 11p + Workspace 3p + 통계·감사 2p(대시보드·감사로그, 2026-06-12 확장) (체크리스트 자체 검토 완료, 사용자 피드백 대기)
2. **Phase 3.5 잔여** — Mintlify 전용 파일 정리 · Cloud 수동 해제
3. **워크스페이스 검수 반영 + 소급 QA 패스 — 커밋** (2026-06-15 워킹트리 정리 상태, 사용자 직접 커밋)

**▶ Deferred (장기 추적)**
- **검색 기능 재작업 완료** (2026-06-15~16) — 검색이 **작동 자체가 안 되던 것**(navbar 숨김 부작용) 발견·복구 + 드롭다운/모달 Mintlify 재설계. 상세·교훈 [[design/porting-checklist]] §P20. **미커밋**(무관 MDX ~50개와 분리해 스코프 커밋 필요) · 모바일 실기기 검증만 잔여(스샷 차단)
- ~~**완료 그룹 소급 QA 패스** (G3/G1/#8·결정11·G6)~~ → ✅ **완료 (2026-06-15)** — 8그룹 81파일 fan-out 점검, grep 17건 + 정성 2건 적용, 적대적 검토 통과. 방법론 [[references/qa-pass-method]]. (#1~#5·i18n은 [[references/translation-verification-report]] 별도 세션 소관)
- **[[references/dify-edition-comparison]] 작성** — CE vs SaaS 기능 분류 (규칙 #8 판단 근거)
- ~~Knowledge 개요 dangling links 4건 잔여~~ → ✅ **전량 해소** (2026-06-10 2차 배치)
- ~~Test Retrieval dangling link 1건~~ → ✅ 해소
- ~~readme.mdx 링크 오탐 가능성~~ → ✅ 해소
- **Knowledge 개요 "플러그인" 표현** — pipeline 페이지에서 "custom steps and various plugins" 중 plugins가 spx-agent에서 사용 가능한지 확인 필요 (현재 "커스텀 처리 단계"로 수정)
- **기존 노드 페이지 broken 2건** — knowledge-retrieval `#create-knowledge` 앵커, variable-assigner `#variables` 앵커 (이번 작업과 무관한 기존 broken)
- ~~**Model Providers 로드 밸런싱** — CE에서 실제 동작하는지 확인 필요~~ → ✅ **해소 (2026-06-15)** — 코드 검증 결과 CE 기본 비활성(`MODEL_LB_ENABLED` 기본 `False`, billing 분기는 SaaS 전용). model-providers "## 로드 밸런싱" 절 삭제. 근거 [[references/spx-workspace-analysis]] §3.5
- **i18n 라벨 불일치 3건** — `appPermissions.action.view`="보기" vs 글로서리 "조회", `action.transfer`="양도" vs "소유권 이전", `acl.grant`="권한 추가" vs "권한 부여". 권한 챕터 검수 시 확인 필요

> ✅ 2026-06-11: **Workspace 그룹 3p 작성 완료**
>   - Overview(부분수정): 단일 워크스페이스 정책, 6종 역할(빌더 추가), 부서·권한 소개, 설정 메뉴 안내
>   - Model Providers(부분수정): System Providers(SaaS) 삭제, Custom 제공자 설정·자격 증명 관리·로드 밸런싱 유지, 유료 callout·Billing 섹션 삭제
>   - App Management(부분수정): SaaS/Community 분기 제거, DSL 내보내기·가져오기, 권한 섹션 1문단 추가
>   - 전역 규칙: #1 SaaS 제거(Model Providers Billing·Load Balancing callout), #2 Cloud/CE 분기(Personal Settings·Model Providers), #5 Dify 브랜딩 전면 교체
>   - 직역체 1건 수정("프록시를 위한"→"프록시용")
>   - 글로서리 갱신: 빌더(Builder) 역할 추가, 설정 메뉴 라벨 7건 추가, Normal "일반 멤버"→"일반" i18n 교정
>   - sidebars.js 등록 (개요·모델 제공자·앱 관리 3건), 빌드 SUCCESS (신규 broken 0건)
>   - Personal Settings 인터리브 기작성 확인 → 신규 작성 불필요
>   - **체크리스트 전수 검토 (사용자 검수 전)**: 누락 3건 발견·보완 — ① §2.4 원문 대조: API 키 보안 Warning 누락 → Model Providers에 `<Warning>` 추가, ② §2.5 i18n 사후 검증: 영어 UI 라벨 6건 한국어 교체, ③ §A 보강: Overview에 대시보드·감사로그 네비게이션 추가. 보완 후 빌드 재검증 SUCCESS

> 🎨 **2026-06-15: UI 리디자인 트랙 시작 (콘텐츠 포팅과 별개)**
>   - 배경: 기존 Mintlify "maple" 복제 테마 → spx-agent 로고(인디고 노드그래프) 기반 자체 디자인으로 전환 결정. Mintlify 픽셀 복제는 Docusaurus와 안 맞아 포기.
>   - 산출물(`design/`): `ui-redesign-proposal.md`(설계안) · `ui-redesign-prototype.html`(확정 시각 명세, 살아있는 스타일 가이드) · **`porting-checklist.md`(프로토타입↔실사이트 전영역 감사 + P1~P5 작업목록)**
>   - 적용 완료(A+B1): 폰트(Pretendard/IBM Plex Mono) · 컬러토큰(인디고/슬레이트) · 콜아웃 5종 · 기본요소 · 복사버튼/Studio 제거 · eyebrow · 접기(hideable) · active/hover 짧은바 · 컴팩트 페이지네이션 · 사이드바 간격 · 다크토글 — 전부 `src/css/custom.css` + `src/theme/*` (MDX·docs.json 무수정), 빌드 SUCCESS
>   - **다음 할 일**: `design/porting-checklist.md` P1(타이포 h1 26·h2 19·h3 16, 본문폭 720, 배경 #FBFBFD, radius 14)부터 → P2 컴포넌트(KPI 부재 등) → P3 사이드바/TOC → P4 모바일/검색 → P5 정리
>   - 결정 필요: 카테고리 펼침 기본값 · chevron 좌/우 · TOC "이 페이지" 라벨

> 🆕 **2026-06-18: 워크플로 배포(Promote/CI/CD) 신규 챕터 작성**
>   - 1차: spx-agent Promote 모듈 소스 검증(feature flag·UI·모달·권한·백엔드·i18n) → [[references/spx-promote-cicd-analysis]] 박제. 사용자 노출(§2·4·6) vs 백엔드(§5) 분리, "개발 환경 전용" 콜아웃 근거(§3) 박제
>   - 2차: 사용자 컨펌(워크스페이스 그룹 배치) 후 `ko/use-spx-agent/workspace/deploy/readme.mdx` 작성 — **백엔드 과정 제거**, 사용자 노출만(스냅샷/운영 반영 흐름·카드 상태·권한·스튜디오 뱃지), 상단 `<Info>` **개발 환경 전용** 콜아웃. `sidebars.js` 워크스페이스 그룹(권한 설정↔개인 계정 사이) 등록, 빌드 SUCCESS(신규 broken 0)
>   - 발견: promote/deploy i18n 라벨 대부분 컴포넌트 하드코딩(전용 i18n json 없음) → 본문은 코드 문자열 정전 사용. 사이드바 위치 결정 [[decisions]] 박제 후보
>   - 후속: 스크린샷(Phase 5), 글로서리 갱신(분석본 §9), `t('settings.deploy')` common 네임스페이스 등록 여부 확인

> 📝 **최근 작업 로그는 [[progress-archive]] 참조** (2026-06-10 이관 — 한눈에 블록은 라이브 현황만 유지).

---

## Monitor 그룹 리뷰 피드백 (2026-06-09)

> Monitor 3p 초안 작성 후 사용자 리뷰. **해결에 필요한 "정답 소스" 기준으로 유형 분류.**
> 🔴 제품 확인 (spx-agent/Dify 실제 화면·동작 — 확정 전 해당 문단 초안 유지) · 🟡 영문 원본 대조 · 🟢 문서 즉시 수정(문장 다듬기)
> 출처: [[1. Daily/2026-06-09]] 메모. 권장 순서: 🟢 → 🟡(제품 없이 가능) → 🔴 3건 일괄(제품 1회 띄워 확인).

**🔴→ A. 제품 확인 — 코드 조사 완료 (2026-06-09). 사실 확정, 본문 반영만 남음** → 근거·상세 [[references/spx-monitor-review]]
- [x] **모니터링** — ✅ analysis.mdx `<Tabs>` 워크플로우/대화 2탭 재구성 완료. 대화형 8지표 + 완성형 차이 `<Info>` / 워크플로우 4지표 (일별 실행·종료·토큰·상호작용)
- [x] **로그 / 개인정보 보호 섹션** — ✅ logs.mdx: 로그 콘솔 워크플로우/대화 2탭 추가 + "개인정보 보호" → "로그 보관과 접근 제어" + `<Note>` "운영·법적 고려사항" 분리
- [x] **어노테이션 답변 / 설정 활성화 경로** — ✅ annotation-reply.mdx: "로그 및 어노테이션 탭" 주 경로로 정정 + 대화형 전용 `<Info>` + 채팅앱 한정 대체 경로 `<Tip>`

**🟡 B. 영문 원본 대조 필요**
- [x] **어노테이션 답변 / `## 동작 원리`** — 원본 5단계 중 ⑤("어떤 어노테이션이 얼마나 자주 쓰이는지 추적") 누락 + Steps 번호·내용이 분리되어 렌더 깨짐 → ✅ 5단계 번호 리스트로 복원, Steps 컴포넌트 → 순서 리스트로 전환

**🟢 C. 문서 즉시 수정 (문장 다듬기)**
- [x] **어노테이션 답변 / 첫 정의 문장** — ✅ "~등록하는 기능입니다" → "~등록해 둘 수 있습니다"
- [x] **어노테이션 답변 / 어색 문장 4건** — ✅ 적중 추적(가치→적중 빈도 기준 설명) / 지속적 개선(커버리지→포착 관점) / 적중률 분석(제거→정리) / 질문 패턴(식별→로그 연계 구체화)

---

## Phase 진척

- [x] **Phase 0**: 환경 셋업
  - [x] 리포 클론 (`Projects/spx-agent-docs/`)
  - [x] git config 작성자 정보 회사 이메일 확인 (mjlee@spelix.com)
  - [x] `npm install -g mintlify` + `mintlify dev` 동작 확인
  - [x] 베이스라인 커밋 `5c1c3a4c` (Dify 1.13.3 sync) 확정 → [[decisions]]
  - [x] `NOTICE.md` (CC BY 4.0 출처 표기 + baseline SHA 명시) 작성
  - [x] `.gitignore`에 `.claude/` 추가 (옵시디언 노트 git 분리)
  - [x] orphan 브랜치로 단일 root 커밋 시작 — `3dde5c1d chore: initial fork from dify-docs@5c1c3a4c`
  - [x] 기존 main(=upstream 추적) 삭제, orphan을 main으로 rename
- [x] **Phase 1**: 소스 분석 + 범위 결정 (2026-05-29 완료, 2026-06-02 재검토 통과)
  - [x] `docs.json` 네비게이션 트리 분석 → [[references/dify-docs-structure]]
  - [x] `en/` 디렉토리 트리 파악 (Use Dify: ~102 pages, 9 groups)
  - [x] Use Dify 섹션 식별 (4개 dropdown 중 Use Dify만 대상)
  - [x] 3-way 매핑 표 1차 작성 → [[scope-mapping]]
  - [x] 신규 챕터 후보 확정 (대시보드·감사로그·RBAC·KC SSO·설정 사이드바, ~10p)
  - [x] **Phase 1.3 매트릭스 재검토** (2026-06-02): 9개 섹션(102p) 전수 재검토, 전역 규칙 5개 신설, 확정 삭제 +2건(web-app-access, rate-limit), 설정 사이드바 신규 챕터 제거 → 8~12p, 용어 글로서리 확장 (Dashboard·Monitoring 매핑 명확화), 패턴 (a)/(b)/(c) 분류 도입(지식 외부 연결)
- [ ] **Phase 2**: 파일럿 2챕터 (1차 번역 + 2차 신규)
  - **사전 셋업** (파일럿 진입 전) — 2026-05-29 완료:
    - [x] `conventions.md` 갱신 — writing-guides 하이브리드 반영
      - formatting-guide 참조 위임 + 한국어 톤 placeholder + 용어집 확장 (노드·UI 레이블·spx-agent 전용)
    - [x] `ko/use-spx-agent/` 디렉토리 골격 생성 (`knowledge/`, `dashboard/` + placeholder MDX)
    - [x] `docs.json`에 ko 언어 섹션 신설 + 한국어를 디폴트 언어로 설정
    - [x] `docs.json`에서 ja, zh nav 비활성 (파일은 보존)
    - [x] `mintlify dev`로 ko 렌더링 + ja/zh 미노출 검증
  - **1차 파일럿 — Knowledge Overview** (원본 번역 패턴) — 완료:
    - [x] `ko/use-spx-agent/knowledge/readme.mdx` 작성 (원본 번역, Dify 브랜드 제거, 내부 링크 ko 변경)
    - [x] `docs.json` ko nav에 등록
    - [x] 텍스트 중심 작성 (스크린샷 없음, Phase 5 교체)
    - [x] mintlify dev 미리보기 확인
    - [x] 번역 톤 검증 → `conventions.md` 합쇼체 확정
  - **UI/브랜딩 정리** — 완료:
    - [x] 로고 spx-agent 교체, name "SPX Agent Docs"
    - [x] dropdown/version/language 셀렉터 제거 (플랫 navigation)
    - [x] Changelog·Footer 소셜·GA4 제거, Studio → 192.168.10.194
    - [x] Powered by mintlify CSS 숨김 (배포 시 재판단)
    - [x] redirects ~900줄 제거
  - **2차 파일럿 — 대시보드** (신규 작성 패턴) — 완료:
    - [x] spx-agent 프론트엔드 코드 분석 (admin 컴포넌트 전수 조사)
    - [x] `ko/use-spx-agent/dashboard/readme.mdx` 작성 (KPI 4종, 종합 뷰, 드릴-스루 4종)
    - [x] `docs.json` nav에 등록
    - [x] mintlify dev 미리보기 확인 (`대시보드 - SPX Agent Docs`)
  - **파일럿 마감**:
    - [ ] 풀 미러링 필요성 재판단 → Phase 4 시작 시 결정으로 보류
    - [x] `conventions.md` 톤·MDX 규칙·용어집 1차 확정 (합쇼체, description 원본 따름, icon은 docs.json 그룹)
    - [ ] Phase 3 컨펌 자료 준비 (매핑 표 + 파일럿 2건 + 호스팅 옵션 비교)
  - **추가 작업** — 2026-06-01:
    - [x] 파일 경로 변경: `ko/use-dify/` → `ko/use-spx-agent/` (내부 링크 포함)
    - [x] GitHub 리포 연결: origin → `mjlee-spelix/spx-agent-demo`, upstream → `langgenius/dify-docs` (참조)
    - [x] 첫 push 완료 (커밋 5개)
    - [x] Mintlify Cloud 연결 (spx-agent-demo, spelix.mintlify.app)
    - [ ] Mintlify Cloud 배포 에러 해소 중 (navigation 구조 호환성 문제)
    - [x] 호스팅 옵션 비교표 작성 → 데일리 메모 (2026-06-01)
    - [x] 대시보드 UI 분석 결과 → [[references/spx-dashboard-analysis]]
- [ ] **Phase 3**: 이사님 컨펌 / 호스팅 방향 확정
  - **사전 조사 — Mintlify → Docusaurus 전환 감사** (사내 git=Gitolite로 Mintlify Cloud 연동 불가 확정 → 자체 호스팅 후보 검증) — **2026-06-01 완료**
    - [x] Mintlify 전용 MDX 컴포넌트 인벤토리 (`en/use-dify/` 범위 한정)
      - 14종 컴포넌트, 총 ~769회 사용. 최다: `<Frame>` 191회, `<Info>` 127회
      - `<ParamField>`, `<ResponseField>`, `<Mermaid>` 등 미사용 확인
    - [x] 프론트매터 차이 조사 (Mintlify 키 vs Docusaurus 키)
      - 6종 키: title(106), icon(52), description(42), sidebarTitle(27), tag(3), mode(1)
      - Mintlify 전용: icon, sidebarTitle→sidebar_label, tag→tags, mode
    - [x] 네비게이션 설정 변환 매핑 (`docs.json` → `sidebars.js` + `docusaurus.config.js`)
    - [x] 링크 형식 전수 조사 (절대경로 vs doc ID, 변환 스크립트 작성 가능성)
      - 내부 62건 (절대경로, 확장자 없음 → 호환), 외부 286건
    - [x] MDX 빌드 호환성 검증 (Mintlify MDX v2 ↔ Docusaurus MDX v3 파싱 차이)
      - `{`/`<` 엄격 파싱, 컴포넌트 자동 임포트 제거 → MDXComponents 래퍼로 해결
    - [x] 이미지/에셋 경로 패턴 — 384건 (로컬 47%, CDN 47%, 외부 6%)
    - [x] 검색 인프라 후보 → `@easyops-cn/docusaurus-search-local` 권장 (한국어, 오프라인)
    - [x] API 레퍼런스 자동생성 → 현재 scope 외, 필요 시 `docusaurus-plugin-openapi-docs`
    - [x] 컴포넌트별 자동 변환(MDXComponents 래퍼) vs 수동 재작성 분류
      - 자동(래퍼): 767건 / 수동: CodeGroup 2건
    - [x] 빈도 상위 컴포넌트 before/after 샘플 — `<Info>`, `<Frame>` 예시
    - [x] 작업량 산정: 시나리오 A(ko/ 만) 1~1.5일, B(ko/+en/use-dify/) 4~5.5일
    - [x] 산출물: [[references/mintlify-to-docusaurus]] 작성 완료
    - 금지: 실제 파일 수정 X, 조사·문서화만
  - [ ] 매핑 표 + 파일럿 + 호스팅 옵션 비교 자료 준비
  - [ ] 회의 컨펌 → [[decisions]] 기록
  - [ ] 차감/추가 범위 확정
  - [x] 호스팅 방향 확정 — **Docusaurus 전환** (2026-06-02, 사내망/외부 GitHub 의존 회피·안정성·비용 → [[decisions]])
- [ ] **Phase 3.5**: Mintlify → Docusaurus 전환 실행 (~4.5~6일, 감사 보고서 기준 → [[references/mintlify-to-docusaurus]])
  - [x] Docusaurus 프로젝트 초기 셋업 — 리포 루트에 직접 구성 (`package.json`, `docusaurus.config.js`, `sidebars.js`, `src/`, `static/`), docs 경로 `ko/`만 빌드 대상, CJS 모드, 검색 `@easyops-cn/docusaurus-search-local`
  - [x] static 에셋 배치 — `images/`·`logo/`·`favicon.svg`·`assets/` → `static/` 복사 완료. 기존 절대경로 유지
  - [x] MDXComponents 글로벌 래퍼 6종 — `src/theme/MDXComponents/` (Admonitions, Frame, Steps, TabsWrapper, Card, Accordion + index.js)
  - [x] CSS 스타일링 — `src/css/custom.css` (테마 컬러 + 래퍼 컴포넌트 스타일)
  - [x] 변환 스크립트 3종 — `tools/migrate/convert-nav.mjs`, `convert-frontmatter.mjs`, `convert-links.mjs`
  - [x] MDX v3 호환성 수정 — ko/ 프론트매터 변환(`sidebarTitle`→`sidebar_label`), 내부 링크 prefix `/ko/` 제거, `{`/`<` escape 불필요 확인
  - [x] `<CodeGroup>` — 2건 모두 en/ 파일, ko/ 빌드에 무관 → Phase 4로 연기
  - [x] `versions/` 폴더 — 보존 (824MB, Mintlify docs 버전 히스토리. Docusaurus 빌드 무관, 삭제 불필요)
  - [x] `scripts/`·`tools/` — 원본 보존 (원본 보존 원칙 적용). `tools/migrate/` 3개 신규 추가
  - [x] GitHub 연동 정리 — origin(`mjlee-spelix/spx-agent-demo`) 제거 완료, Mintlify Cloud 수동 해제 필요, `.github/workflows/` 원본 보존
  - [x] 빌드 검증 — `npm run build` 성공, 검색 인덱스 생성 확인, broken links는 Phase 4 페이지 추가 시 해소 예정
  - [x] 문서·설정 갱신 — 0.25일 (2026-06-02 완료)
    - [x] `conventions.md` — Mintlify 컴포넌트 → MDXComponents 래퍼 문법 반영, 내부 링크 형식(`/ko/` prefix 제거 → 상대경로), frontmatter(`sidebarTitle` → `sidebar_label`), 네비게이션(`docs.json` → `sidebars.js`), 빌드·개발 환경 섹션 추가
    - [x] `.claude/CLAUDE.md` — 보존 대상 표 갱신(`docs.json` read-only, 신규 추가 영역 표 신설), 부득이 수정 영역→부득이 수정한 영역(완료형), 덮어쓰는 규칙 갱신, 빌드 도구 섹션 확장(개발 명령어·설정 파일 표), 외부 리소스(origin 제거 반영)
    - [x] 루트 `README.md` — SPX Agent Docs용 전면 재작성 (Docusaurus 기술 스택, Phase 7단계, 리포 구조, 커밋 컨벤션)
    - [x] `NOTICE.md` — CC BY 4.0 의무: Docusaurus 전환·ko/ 신설·브랜딩 교체·빌드 도구 마이그레이션 변경 사실 명시
  - [ ] Mintlify 전용 파일 정리 — `.mintignore` 삭제, `style.css` → `src/css/custom.css` 흡수, 루트 MDX(`development.mdx`·`introduction.mdx`) 전환·삭제, 루트 이미지(`dify-logo.png` 등) 이동·정리
  - [ ] 버퍼 (예측 외 이슈 대응) — 0.5~1일
  - **권장 순서**: 시나리오 A(ko/ 파일럿만, 1~1.5일) → 래퍼·스크립트 검증 → Phase 4 본격 포팅 진입 시 시나리오 B로 확장
  - **원본 보존 처리**: `docs.json` 원본은 NOTICE 차원에서 보존(읽기 전용), `sidebars.js`는 변환 산출물로 별도 관리 — Phase 3.5 진입 시 박제 필요
- [ ] **Phase 4**: 본격 포팅
  - **사전 분석 (각 챕터 작성 시작 전 spx-agent 코드/UI 분석)** — **모든 분석은 `references/spx-<주제>.md`로 박제 의무** (CLAUDE.md 응답 규칙)

    > **진행 방식 (2026-06-02 결정)**: 클러스터 단위 **인터리브** — 각 분석 산출물 완료 시 해당 챕터 1p 임시 작성으로 검증 → 누락 발견 시 분석 보강 → 다음 분석. 산출물 간 [[링크]]로 권한 모델/도메인 정의 중복 제거.
    >
    > **순서**: A1 → (A2 ∥ A3) → A4 → B1 → B2 → C1

    ### 클러스터 A — 권한·RBAC (4건)

    > 공통 코드 영역: `web/app/components/{app,dataset,tool}-permissions/`, `services/rbac/`, 가시성 4단계 enum, ACL/소유권 모델, `spx_*` 사이드카 테이블. **A1에서 권한 모델 코어를 정립하면 A2~A4는 차이점만 추가**.
    >
    > **🔴 선행 참고 자료 (A1 분석 시작 전 필독)**:
    > - `C:\Users\Administrator\Projects\spx-agent\dify_rbac_hdd_design.md` — RBAC HDD 설계 문서 (1082줄)
    > - `C:\Users\Administrator\Projects\spx-agent\RBAC_HANDOVER.md` — RBAC 핸드오버 문서 (605줄)
    >
    > 위 두 문서가 권한 모델 설계 의도·5개 사이드카 테이블 스키마·가시성 4단계 정의를 이미 박제하고 있을 가능성 높음. 코드 deep dive 전 먼저 읽어서 **모르는 부분만 코드로 확인**하는 게 효율적. A1 산출물(`spx-app-permissions-analysis.md`)은 이 두 문서의 요약 + 코드 검증 결과 + 사용자 매뉴얼 관점 재정리 구조로 작성.

    - [x] **A1. 권한 설정 신규 챕터 (P1) — 권한 모델 코어** (2026-06-02 완료) — `web/app/components/app-permissions/`, `dataset-permissions/`, `tool-permissions/` 풀스택 분석 (가시성 4단계 동작, ACL CRUD, 소유권 이전 UI 흐름). **다른 분석이 여기 링크** → [[references/spx-app-permissions-analysis]]
      - [x] HDD·HANDOVER 정독, 코드 9개 영역 검증 (authorization_service / constants / app-permissions / dataset-permissions / tool-permissions / rbac_enforcement / tool_providers_permission / workspace/rbac / account-setting)
      - [x] HDD↔코드 차이 9건 정리(C1~C9) — 신규 식별: C5 워크스페이스 RBAC 3탭 분리, C7 부서 편집/삭제 hidden
      - [x] 사용자 매뉴얼용 어휘로 재정리 (도메인 모델·가시성 4단계·ACL·소유권·권한 판정·역할 매트릭스·UI 위치)
      - [x] A2~A4 인용 매핑 표 박제 — 후속 산출물이 본 문서를 정전으로 참조
      - [x] **인터리브 챕터 1p**: `ko/use-spx-agent/app-permissions/readme.mdx` 작성 + `sidebars.js` 등록 + 빌드 검증 통과
      - [x] 작성 중 누락 보강: §5.4(사용성 제약) 신설, §9.2(권한 메뉴 vs 워크스페이스 권한 탭 가시성 메커니즘 구분) 갱신, §11(한국어 매핑 박제)
      - [x] conventions 글로서리 일괄 갱신 — line 218에 권한 모델·로그인 옵션·감사 로그 3개 sub-section 흡수 완료(2026-06-04)
      - [ ] 후속(deferred to Phase 4 본격 작성): §11.4 3건(사이드바 i18n / DENY 진단 동선 / transfer UI 동선) 추가 검토
    - [x] **A2. Knowledge 권한 설정 3p (P1)** (2026-06-04 완료) — A1 권한 모델 [[references/spx-app-permissions-analysis]] 링크 + 지식 도메인 특이사항만 → [[references/spx-knowledge-permissions]]
      - [x] `dataset_acl_sync_service.py` 전수 검증, 가시성→네이티브 매핑·INV-8·INV-9·drift API 박제
      - [x] `dataset-permissions/` UI 코드 검증 — 모달은 App과 공용, 액션 8종 노출(duplicate 효과 없음)
      - [x] 원본 Knowledge 3p의 권한 언급 grep — `manage-knowledge/introduction.mdx`만 명시 행 보유. 나머지 2p는 신규 추가, 3p별 통합 위치·1문단 안내 정리
      - [x] K1~K6 차이점 박제 (네이티브 sync / creator 포함 / view+ALLOW 투영 / drift / 액션 7종 / 원본 통합 위치)
      - [x] **인터리브 챕터 1p**: `ko/use-spx-agent/knowledge/permissions/readme.mdx` 작성·`sidebars.js` 등록·빌드 통과
      - [ ] 후속(deferred to Phase 4 본격 작성): 파이프라인 생성 화면 가시성 UI 추가 검증, "운영자" 호칭 A3·A4와 통일
    - [x] **A3. 사용자/부서 관리 신규 챕터 (P2)** (2026-06-04 완료) — 부서 CRUD UI(워크스페이스 설정 내 사이드바 2탭) + KC 그룹 동기화 운영 흐름 + A1·A2 모호 동선 흡수 → [[references/spx-departments-management]]
      - [x] `departments-page/`·`members-page/`·`members-modal.tsx`·`department-cell.tsx` UI 코드 검증
      - [x] `department_service.py` 5개 메서드·`account_service._sync_department_from_keycloak_groups` 정책 검증
      - [x] **A1 §C4 정정 박제** — Keycloak 동기화가 HANDOVER 기술(가산형)과 다르게 코드는 **mirror(wholesale replace)** 정책. A1 산출물 갱신 완료
      - [x] §3.5 사용성 함정 박제 — 멤버 모달의 `unassign` 액션은 모든 부서 해제 (부서별 부분 제거는 부서 셀 set으로만)
      - [x] A1·A2 모호 동선 흡수 — DENY 진단·지식 검색 권한 재동기화 절차의 정답 위치 매핑
      - [x] **인터리브 챕터 1p**: `ko/use-spx-agent/workspace-management/departments/readme.mdx` 작성·`sidebars.js` "워크스페이스 관리" 카테고리 등록·빌드 통과
      - [ ] 후속(deferred to Phase 4 본격 작성): Dify 원본 멤버 페이지 역할 변경 흐름과 통합, 부서 편집·삭제 UI 노출 정책 결정
    - [x] **A4. Workspace 11p 전수 재검토 (P1)** (2026-06-04 완료) — A1~A3 정립 후 5/29 비고 정정 + 2026-06-02 전역 규칙 #1~#5 적용 → [[references/spx-workspace-analysis]]
      - [x] 원본 11p grep + 핵심 키워드 식별 (SaaS·Cloud version·Marketplace·System Providers)
      - [x] 5/29 매트릭스 가정 오류 2건 정정:
        - Personal Settings "KC SSO 기반 계정" → 실제 4종 옵션(이메일+비번/이메일+코드/소셜/SSO) systemFeatures 플래그
        - Team Members "삭제 — UI 멤버 관리 없음" → spx-agent는 멤버·부서 UI 모두 보유, A3 챕터로 콘텐츠 위임
      - [x] 5/29 "유지·번역" 2건 → 부분 수정 격상: App Management(DSL Version SaaS/Community 분기), Model Providers(System Providers 절 SaaS 개념)
      - [x] 액션 분포 확정: 부분 수정 5p·삭제 3p·컨펌 대기 3p
      - [x] API Extension 3p 컨펌 트랙 분리 (사내 외부 노출 정책 미결)
      - [x] 11p별 처리 지침 + 전역 규칙 #1~#5 적용 위치 + A1~A3 인용 매핑 박제
      - [x] **인터리브 챕터 1p (대표)**: `ko/use-spx-agent/workspace-management/personal-settings/readme.mdx` 작성·`sidebars.js` 등록·빌드 통과 — 5/29 가정 정정 실효성 검증
      - [ ] 후속(deferred to Phase 4 본격 작성): Personal Settings UI 정확한 진입점 확인, Team Members 페이지 vs A3 챕터 통합 vs 리다이렉트 결정, scope-mapping 8 Workspace 표 본 결정으로 갱신

    ### 신규 분석 — 2026-06-05 부서 필터링 UI 발견

    > 스튜디오 상단 부서 드롭다운(앱 목록 부서별 필터링) 가이드라인 누락 발견. Dify 원본 없음 — spx-agent 추가 기능. A3(사용자/부서 관리) 챕터에 sub-section 신설 예정. 분석 후 references 박제.

    - [x] **부서 필터링 UI 분석** → [[references/spx-department-filter-analysis]] 작성 완료 (2026-06-05)
      - [x] 영향 범위 식별 — **앱 목록·지식 목록 2종만** (도구 목록은 미적용 — `useSelectedDepartmentStore` 사용처 grep 결과 0건)
      - [x] "(미지정)" 의미 확인 — **헤더 셀렉터에는 없음**. "전체"(필터 해제)만 노출. "할당 없음"·"미배정"은 멤버 페이지·대시보드의 별도 라벨로 의미 다름(부서 0개 사용자/자원). 매뉴얼에서 혼동 방지 박제
      - [x] 권한과의 상호작용 — 필터는 권한 평가 별도 layer. `WHERE owner_department_id = :selected` 추가만 — **권한 우회 안 함**(접근 권한 자원 안에서 추가 좁힘)
      - [x] 노출 부서 결정 로직 — 4분기 박제: OWNER/ADMIN(전체+활성 부서) / 비-admin 다중(본인 부서들만) / 비-admin 단일(read-only) / 비-admin 0개(미렌더)
      - [x] 코드 컴포넌트 — `web/app/components/header/account-dropdown/department-selector/index.tsx` + `useSelectedDepartmentStore` (zustand + localStorage)
      - [x] i18n 라벨 — `common.departmentSelector.header`("부서")·`common.departmentSelector.all`("전체") **i18n 미등록 발견** → `defaultValue` fallback 사용. 정식 등록 후속 항목
    - [x] **A3 분석본 §4.5 "헤더 부서 셀렉터" sub-section 박제 + §10.1 인용 매핑 행 추가** (2026-06-05 완료)
    - [ ] **A3 챕터 본문 (`workspace/departments/readme.mdx`) §"헤더 부서 선택" 절 추가** — Phase 4 본문 작성 시 ([[references/spx-department-filter-analysis#6-챕터-본문-톤-예시]] 그대로 옮김)
    - [ ] **Quick Start에 한 줄 anchor 추가** — "상단 부서 드롭다운으로 부서별 자원 보기 가능" (C1 분석 시 함께)
    - [x] **[[3. 프로젝트/spx-agent-docs/docs/scope-mapping]] 사용자/부서 관리 챕터 비고 정밀화** (2026-06-05 완료) — 신규 챕터 표 비고에 "헤더 부서 선택 sub-section 포함, 도구 미적용, 권한 평가는 별개" 명시 + 코드 매핑 표 프론트엔드 소스에 헤더 셀렉터 컴포넌트 박제 (`department-selector/`, `useActorDepartments`, `useSelectedDepartmentStore`)
    - [ ] i18n 정식 등록 + "할당 없음" vs "미배정" 표기 통일 — Phase 4 본격 작성 시

    ### 클러스터 A·B1 — 6/4 결정 반영 + 미완 작업 갱신 (Cluster B2 시작 전 처리 권장)

    > 2026-06-04 이사님 회의 결정 10건이 A1~A4·B1 분석본·챕터 본문에 영향 + B1 인터리브 1p 미작성. 시급도·작업량 순.

    - [x] **A3 [[references/spx-departments-management]] 갱신 (가장 시급)** (2026-06-04 완료, "남은 작업 1번")
      - [x] **KC 추상화 본문 적용** (전역 규칙 #6 박제) — §0 표기 가이드 신설(정책·사유·보존 영역·본 문서 내부 적용), §5 도입부 정책 박스
      - [x] **team-members-management 변환 정보 박제** (결정 6) — frontmatter `purpose`·§1 본 문서 두 역할 명시·§8.2 변환 처리 표 신설(원본 절별 변환 후 위치)·§10 A4 인용 매핑에 반영
      - [x] §5.6 신설 — 챕터 본문 추상화 표기 예시 6개 절(도입·동작 정책·안전장치·매칭 우선순위·동기화 시점·진단 동선) — 챕터 작성 시 그대로 옮길 톤
      - [x] §10.1 신설 — 챕터 본문이 본 문서 어디를 인용할지 위치 명시 (외부 IdP 동기화는 §5.6 그대로 옮김 박제)
    - [x] **A3 챕터 본문 + personal-settings KC 추상화 적용** (5번 작업, 2026-06-04 완료)
      - [x] `workspace-management/departments/readme.mdx` — 9개 직접 노출 행을 A3 §5.6 톤("외부 시스템(예, Keycloak)" / "외부 시스템")으로 교체. 헤딩 "## Keycloak 그룹 동기화" → "## 외부 시스템 그룹 동기화", FAQ 헤딩 "### Keycloak에서 그룹을 뺐는데..." → "### 외부 시스템에서 그룹을 뺐는데..."
      - [x] 그룹 동기화 동작 설명(mirror 정책 등) 사실 그대로 유지
      - [x] `workspace-management/personal-settings/readme.mdx` — 5개 직접 노출 행 추상화. SSO 표 행 라벨 "SSO (Keycloak)" → "SSO" + 설명에 "외부 시스템(예, Keycloak)" 풀어쓰기. FAQ "SSO (Keycloak)" 라벨도 "SSO"로 통일
      - [x] 상호 링크 anchor 정정 — `#keycloak-그룹-동기화` → `#외부-시스템-그룹-동기화` (양 본문 일관)
      - [x] **자체 검토 1건 처리** — 헤딩에 "외부 **IdP**" 약어 사용 → 결정 5 정책("외부 시스템")과 일관성 위해 "외부 시스템 그룹 동기화"로 통일. 앵커도 동시 갱신
      - [x] 빌드 검증 통과 (본 작업 외 broken link 없음)
    - [x] **A4 [[references/spx-workspace-analysis]] 갱신** (2026-06-04 완료)
      - [x] API Extension 3p 컨펌 대기 → **삭제 확정**으로 상태 변경 (결정 2 적용, outbound 일방향 재분류) — §2 행 8/9/10·§4 전면 갱신, §4.2 outbound 일방향 근거 박제
      - [x] team-members-management 변환 정보 박제 (결정 6) — A3 챕터로 대체된다는 인용 — §2 행 3·§3.3·§6
      - [x] Personal Account "KC SSO 기반" 가정 정정 박제 (KC 추상화도 함께 적용) — §3.2 (4종 옵션 정정 + 추상화 지침)
      - **검토 발견 — 분석본 자체 수정 필요 (본문 단계 전 처리, 2026-06-04 검토)** — **5건 모두 처리 완료 (2026-06-04)**:
        - [x] (실질) §7.2 인터리브 골격 KC 추상화 모순 해소 — "Keycloak SSO"·"KC SSO" → "SSO + 외부 시스템(예, Keycloak)" 톤으로 교체, personal-settings 챕터 KC 추상화 환기 박스 추가 (A3 §5.6 인용)
        - [x] (실질) §1.2 액션 분포 표 — "유지·번역" 컬럼 추가하여 합계 11 맞춤. 5/29 유지·번역 2건이 6/4 초안에서 부분 수정으로 격상된 경위 각주 추가
        - [x] (minor) Plugins 삭제 사유 통일 — §3.6 "전역 규칙 #5" → "결정 1"로 §2와 일관. §3.6 자체도 5/29부터 3p → 결정 2 적용 6p로 확장 갱신
        - [x] (minor) §7 헤더 "(작성 예정)" → "(작성 완료)" + §9.1 빌드 통과 항목 인용
        - [x] (minor) §7.1 경로 혼용 분리 — 현재 위치(`workspace-management/personal-settings/`)와 목표 위치(`workspace/personal-settings/`, Option α 이동 후) 명시 분리
        - [x] §9.2.1 신설로 검토 결과 5건 처리 박제
    - [x] **A2 [[references/spx-knowledge-permissions]] 갱신 (가벼움)** (2026-06-04 완료)
      - [x] §외부 연결 패턴 (a)/(b)/(c) 상태 표기 갱신: — §7.1 전면 갱신
        - 패턴 (a) 외부 import → **삭제 확정**
        - 패턴 (b) 외부 KB read-only → **삭제 확정**
        - 패턴 (c) 외부 노출 API → **유지·번역 확정** (inbound 영역)
      - [x] §"차감 시 본 챕터 영향 없음" 표기는 유지 (실제로 영향 없음) — §7.2 행 1
      - **검토 발견 — 분석본 자체 수정 필요 (본문 단계 전 처리, 2026-06-04 검토)** — **5건 + 챕터 본문 연쇄 1건 처리 완료 (2026-06-04)**:
        - [x] (실질) §1 "다섯 가지" → "여섯 가지" 정정 (K1~K6 = 6개와 일관)
        - [x] (실질) 액션 종수 명확화 — K5 "7종" → "유효 6종 (publish·duplicate 미정의)"로 정정. **§5.0 신설**로 UI 8종 vs 유효 6종 구분. **execute(검색 테스트) 적정성 검증** — A1 §3.2 + `rbac_enforcement.py` 매핑으로 정확함 박제. §5.2 헤더, §6.3·§8.1 챕터 골격·§10.1 후속도 일괄 정정
        - [x] (실질) §6.1 상대경로 `./permissions` → `../permissions` 정정 (3p 모두 `knowledge/<subdir>/<file>.mdx` 위치라 한 단계 위)
        - [x] (minor) §10.2 마지막 `[x]` 모순 정정 — "Phase 4 본격 작성 시 챕터 본문 FAQ 답변 교체"는 분석본 근거 박제까지로 한정 표기. §10.3에 본문 교체 항목 별도 추가
        - [x] (minor) §5.2 헤더 "8종 노출" → "UI 8종 / 유효 6종" 부연 추가 (§1 K5와 일관)
        - [x] **챕터 본문 연쇄 정정** — `knowledge/permissions/readme.mdx`의 L13 "7종 + 복제만" → "6종 + 발행·복제", L47 "7종" → "6종", L58 Note에 publish 추가 (분석본 K5와 일관성). 빌드 통과
        - [x] §10.2.1·§10.2.2 신설로 검토 결과 5건 + 챕터 본문 연쇄 정정 박제
    - [x] **A1 [[references/spx-app-permissions-analysis]] 정리 (작음)** (2026-06-04 완료)
      - [x] 챕터명 rename 반영 ("앱 권한 설정" → "권한 설정")
      - [x] §13 후속 항목 중 Marketplace/MCP 관련 행 정리 — grep 결과 Marketplace 0건·MCP 1건(§3.1 표현 유지). 4번 작업 noop임을 §13.1에 박제
      - [x] frontmatter status_history·related_decisions 신설, §1.1 결정 1~10 영향 표, §13 6항목 표 재구성, §12.2 역방향 인용 매핑 신설 (A2 §5.0·A3 §0/§5/§5.6/§8.2·A4 §3.3/§3.4 위치 박제)
      - [x] 자체 검토 1건 처리 — §12.2 "§13.4·§13.3" 표기 → "§13 표 4번·3번 행"으로 정정 (§13.1 신설 절과 헷갈림 회피)
      - [x] 챕터 본문 헤더·소개 "권한 설정" 명칭 반영 + `app-permissions/` → `workspace/permissions/` 폴더 이동 (6번 작업, 2026-06-04 완료)
    - [x] **분석본 내부 동기화 — 완료 상태·Option α 반영 누락 (2026-06-04 검토 발견)** — 9번 정합 동기화 작업에서 일괄 처리 완료
      > 작업 추적(progress)·챕터 본문·폴더·sidebars는 Option α로 완료됐으나, 분석본 reference 파일 내부가 옛 상태(미완 체크·죽은 경로·Option G 라벨) 잔존. 빌드 영향 없음(내부 .md 메모), 분석본만 읽는 사람이 오해.
      - [x] (실질) 분석본 체크리스트가 **완료된 작업을 `[ ]` 미완으로 표기** → `[x]` 정정 (9번 동기화):
        - A3 §12.3 5번·7번 → [x] 처리·결과 박제
        - A4 §9.3 5번·6번·7번·sidebars 8번 → [x] 처리, 별도 작업 추적 표 재구성
        - A1 §11.1 작성 파일 경로 "이동 예정" → 6번에서 이미 이동 완료로 갱신
      - [x] (실질) 죽은 경로 `workspace-management/` 잔존 정정 — A3 §10.1 L250 → `workspace/departments/`로 갱신, A3 §12.3 L470·L471 완료 처리, A4 §9.3 L394·§7.1 L307 정합 갱신 (2026-06-04 동기화 작업)
      - [x] (라벨) "Option G" 잔존 → "Option α" 통일 — A1 L379(이동 완료로 갱신), A3 §12.3(완료 처리), A4 §7.1·§9.3(완료·동기화 갱신), §9.2.1 본문 표기 (2026-06-04 동기화 작업). A1 L55·L512, A4 L173·L308·§9.2.1 본문 일부, **decisions 결정 9 본문**의 "Option G" 잔존은 변천사 박제로 보존(역사 추적용)
    - [x] **B1 [[references/spx-audit-log-analysis]] 갱신 + 인터리브 1p 작성** (2026-06-04 완료)
      - [x] **KC 추상화 분석본 적용** (전역 규칙 #6) — §1.1 표기 가이드 신설(보존 영역 명시: DB·JWT·컨테이너 등 코드/메타 식별자는 분석본 정전이라 §0.3 따라 보존), §4.1 "사용자" 열·검색 절은 추상화 표기 + 챕터 인용 환기. deferred 항목 [x] 처리
      - [x] **인터리브 챕터 1p 작성** — `ko/use-spx-agent/analytics-audit/audit-log/readme.mdx` (Option α 폴더). sidebars.js "통계·감사" 카테고리 등록(슬롯 주석 → 실제 항목 교체) + **빌드 통과**(신규 broken link 없음, 기존 미작성 페이지 링크만 잔존)
      - [x] 챕터 본문 KC 추상화 적용 — "Keycloak" 직접 노출 0건. "외부 시스템에서 보강"·"사용자 이름·이메일로 검색"·"서버 연동 구성에 따라" 톤
      - [x] B1 deferred 함께 처리 — 시스템 로그 탭은 챕터에서 짧은 1개 절(운영 점검용)로 비중 축소, nginx/보안 이벤트는 "서버 연동 구성에 따라 수집" 톤 적용. "데이터셋=지식" 안내 `<Note>` 박음
      - [x] **"데이터셋" vs "지식" 표기 통일** — 옵션 (가) 채택 (2026-06-04). 글로서리는 "지식" 유지, B1 챕터 본문에 "감사 로그에 '데이터셋'으로 표시되는 항목은 지식 베이스를 의미합니다" 한 줄 안내 박기. audit-meta.ts 코드 수정(옵션 나)은 보류 → [[1. Daily/2026-06-04]] 메모 박제
      - [x] **conventions 글로서리 일괄 갱신** (`audit-meta.ts` 카테고리·액션 라벨) — 2026-06-04 완료. 권한 모델·로그인 옵션·감사 로그 3개 sub-section 흡수 (~50건 매핑)

    ### 클러스터 B — 관찰성·로깅 (2건)

    > "워크스페이스 KPI(대시보드) vs 앱 단위(Analysis) vs 이벤트(감사로그)" 영역 구분이 핵심.

    - [x] **B1. 감사로그 신규 챕터 (P1)** (2026-06-04 완료) — `dify-audit/` deep dive (별도 Next.js 앱, iframe 임베드) → [[references/spx-audit-log-analysis]]
      - [x] 아키텍처 박제 — 별도 앱 + iframe(설정→감사 로그 탭, owner/admin), 3 DB(audit/dify/keycloak) 읽기, 통합 테이블 `spx_audit_events`
      - [x] 수집 경로 5종 박제 — DB 폴링 13수집기(dify_db) / PostgreSQL 트리거 6종(pg_trigger, DELETE·역할변경) / nginx 와처 3종(보안·api_call) / self-audit 6종 / 시스템 로그 일배치(별도 `spx_system_logs`)
      - [x] 이벤트 카탈로그 전수 — action·category·source·원천테이블·한국어 라벨(`audit-meta.ts` 기준)
      - [x] UI 박제 — 2탭(이벤트/시스템 로그), EventTable 7열, FilterBar(카테고리→액션 동적·기간·검색은 Keycloak까지), 상세 4섹션, Export(CSV BOM/JSON·필터 적용·최대 5만건)
      - [x] 권한·격리 박제 — owner/admin only, 테넌트 격리(adminTenantIds OR NULL), 직접 URL 차단(iframe만)
      - [x] 수집 주기·보존 — 폴링 5분(최대 지연), 보존 기본 90일, 시스템 로그 1시 배치
      - [x] §8 영역 구분(대시보드/모니터링/감사 로그 3축) — B2 인용용, §10 한국어 매핑 박제
      - [x] **"데이터셋" vs "지식" 표기 통일** — line 201 옵션 (가) 채택 완료. B1 챕터 본문에 안내 한 줄 박기로 결정
      - [x] **conventions 글로서리 일괄 갱신** — line 218 완료(권한 모델·로그인 옵션·감사 로그 3개 sub-section)
      - [x] **후속 일괄 완료 (2026-06-04)** — KC 추상화(분석본 §1.1 + 챕터 본문) / 인터리브 챕터 1p(`analytics-audit/audit-log/readme.mdx` 작성·sidebars 등록·빌드 통과) / 시스템 로그 탭 비중 축소 / nginx "서버 연동 구성에 따라" 톤. 남은 deferred: 스크린샷(Phase 5), B2 3축 비교표(B2에서)
    - [x] **B2. Monitor Analysis 페이지 — 폐기 (2026-06-04)** — 별도 분석본·3축 비교표 모두 deferred. Monitor Analysis는 단순 부분 수정 (제목 "Analysis" → "모니터링" 통일)만 Phase 4 본문 작성 시 처리. 별도 references 산출물 생성 안 함.
      > **폐기 사유**: 원래 B2가 묶인 이유 = 대시보드/감사로그가 모니터링 그룹 안에 있던 시절(원래 grouping)에는 3축 비교 inline이 필요했음. **Option α(2026-06-04 확정)로 모니터링 / 통계·감사 분리되면서 사용자 혼동 가능성 ↓** → 별도 비교표 만들 ROI 낮음.
      > **3축 비교표 deferred**: B1 §8에 이미 정리되어 있음. 사용자 피드백 시 본문에 추가 (현재는 미작성).
      > **남은 작업**: Monitor Analysis 챕터 본문 작성 시 제목·UI 라벨 "Analysis" → "모니터링" 치환만 (전역 규칙 적용 차원). 별도 분석·인용 없음.

    ### 클러스터 C — 진입점 정리 (1건, 후행)

    > A·B 완료 후 키워드·정의 안정 시점에 마무리.

    - [x] **C1. Get Started 3p (간소화)** (2026-06-04 완료) — Introduction 신규 챕터 키워드 티저 위치 / Quick Start 부서·권한 노출 위치 한 줄 / Key Concepts 부서·RBAC 정의. A1·A3 결과 인용 → [[references/spx-get-started-analysis]]
      - [x] 원본 3p grep + 전역 규칙(#1·#3·#5·#6)·결정(1·3·5) 적용 지점 식별 — 신규 코드 분석 불필요(A1·A3·대시보드·B1 인용)
      - [x] Introduction — Dify 소개 재작성, CardGroup 6장 → 유지 3 + 신규 티저 4(권한 설정·사용자/부서 관리·대시보드·감사로그) 재구성, 외부 채널(Self Host·Forum·Changelog) 제거
      - [x] Quick Start — 로그인=사용자명/비밀번호 폼(+SSO 시 부서 자동 배정 한 줄, A3 §5.4/§5.6 인용), 모델 제공자 "플러그인 설치" 표현 조정(결정 1), 앱 생성 단계 소유자·소유 부서 + 권한 설정 링크 한 줄(A1 §3·§4·§6)
      - [x] Key Concepts — 공통 개념 유지·번역, 부서·RBAC 정의만 추가(A1 §7.3 역할 5종·A3 §1·§5), 가시성·ACL·소유권은 권한 설정 챕터 위임 경계 박제
      - [x] 인용 매핑 + 챕터 본문 골격 3p + 후속 체크리스트 박제. 역방향 A3 §10 인용(§5.4·§5.6) 일치 확인
      - [x] **인터리브 챕터 3p 작성** (2026-06-04) — `ko/use-spx-agent/getting-started/{introduction,quick-start,key-concepts}.mdx` 작성·`sidebars.js` "시작하기" 카테고리 신설(최상단)·빌드 통과. Introduction 티저 4종(관리 기능 CardGroup)·Quick Start 로그인+앱생성 한 줄·Key Concepts 부서/RBAC 정의 검증 완료. Quick Start 워크플로우 빌드 본문(2단계)은 유지·번역 요약으로 처리(P2 본격 번역 대상). KC 추상화 적용(외부 시스템 표기)
      - [x] **Phase 4 본격 작성 완료** (2026-06-09) — 아래 deferred 7건 처리:
        - [x] Quick Start 2단계 워크플로우 빌드 본문 전체 번역 — 원본 9개 노드 구성(시작→매개변수 추출기→IF/ELSE→List 연산자→Doc 추출기→LLM→반복→템플릿→출력) 상세 번역. 3단계(테스트) 입력 예시·팁 보강
        - [x] `gpt-5.2` 모델명 일반화 — 직접 언급 제거, "LLM"/"모델"로 대체 (기존 "시작하기 전에" 톤과 통일)
        - [x] 로그인/SSO 표현 원칙 확립 — "SSO로 로그인하는 경우" 분기 자체 제거. "연동된 계정 관리 시스템에 따라 자동으로" 패턴. scope-mapping 전역 규칙 #6에 박제. 전체 ko/ 4개 파일 6개소 일괄 수정
        - [x] UI 라벨 i18n 전수 검증 — 3p 전체 볼드 라벨 추출 → `web/i18n/ko-KR/*.json` 대조. 불일치 10건 수정 (파라미터 추출기→매개변수 추출기, 리스트 연산→List 연산자, 문서 추출기→Doc 추출기, 사용자 입력→시작, VISION→비전, 지시문→지시, 구조화→구조화된, 게시→게시하기 등). 변수 참조도 노드 기본명 반영 (시작/, Doc 추출기/, 템플릿/)
        - [x] 이미지 누락 복원 — introduction 대표 이미지 1개, key-concepts 개념 설명 이미지 7개, quick-start 워크플로우 개요 다이어그램 1개 추가
        - [x] MDX 포맷 수정 — key-concepts H3→H2 승격(헤딩 순서 규칙), quick-start Jinja2 코드블록 언어 태그 추가. introduction description frontmatter 추가
        - [x] UI 텍스트 수정 — "빈 앱으로 시작"→"빈 상태로 시작", "생성"→"만들기", "기본 모델 설정"→"시스템 모델 설정" (i18n 검증), 앱 생성 Info 뉘앙스 변경(기본 비공개 + 권한 설정 안내)
        - [x] chapter-writing-checklist 기준 전체 검증 통과 (전역 규칙 #1~#6 잔존 0건, MDX 포맷, 문체, 빌드 SUCCESS)
      - [ ] 후속(다른 챕터 의존): 신규 챕터(nodes·publish/mcp·tutorials) 작성 후 key-concepts 깨진 링크 3건 + introduction 튜토리얼 카드 자동 해소. 이미지는 Phase 5 spx-agent 스크린샷 교체

    ### 완료된 분석 (참조용)

    - [x] **대시보드 신규 챕터** — 2차 파일럿에서 deep dive 완료 → [[references/spx-dashboard-analysis]] (본격 작성 시 확장, B2에서 영역 비교 시 참조)

    ### 컨펌 결과 적용 (2026-06-04 이사님 회의로 일괄 해소)

    > 모두 [[decisions#2026-06-04 — 이사님 회의 결정 일괄 박제]]로 박힌 후 매트릭스에 반영 완료.

    - [x] Marketplace 처리 — **전체 제거** (결정 1)
    - [x] Plugin Trigger → **삭제** (Marketplace 제거 연쇄, 결정 1)
    - [x] MCP → **유지·번역** (결정 3)
    - [x] 외부 SaaS observability 송출 (Monitor integrations 7p) → **삭제** (결정 2)
    - [x] 외부 데이터 import (지식 a) → **삭제** (결정 2)
    - [x] 외부 KB read-only 연결 (지식 b) → **삭제** (결정 2)
    - [x] 외부 노출 inbound 3건 (publish/webapp/embedding-in-websites / developing-with-apis / maintain-dataset-via-api) → **유지·번역** (결정 4)
    - [x] workspace/api-extension/* 3건 — 양방향 표기 정정 후 outbound 일방향으로 재분류 → **삭제** (결정 2 적용)
    - [x] workspace/team-members-management → **변환** (사용자·부서 관리 신규 챕터로 대체, 결정 6)
    - [x] KC 추상화 — 전역 규칙 #6 신설, 모든 페이지 본문에 적용 (결정 5)

    ### 사이드바 구조 (2026-06-04 결정 10 — Option α 채택, Option G에서 진화)

    > Workspace 흡수(권한 설정·부서 관리) + **통계·감사 별도 top-level 그룹 신설**(대시보드·감사로그). 모니터링 그룹은 앱 단위 조회만 평면 구조. [[decisions#결정 10]] 참조.
    >
    > **G → α 변경 사유**: Monitor 안 "워크스페이스 단위" sub-group 작명이 직관적이지 않다는 우려. 의미상으로도 워크스페이스 단위 조회(대시보드·감사로그)는 별도 도메인(통계·감사)로 분리하는 게 자연.

    **폴더 구조 (Phase 4 본격 진입 전 박기)**:
    - [x] `ko/use-spx-agent/workspace/permissions/` 생성 + 기존 `ko/use-spx-agent/app-permissions/` 삭제 (6번 작업, 2026-06-04 완료)
    - [x] `ko/use-spx-agent/workspace/departments/` + `workspace/personal-settings/` 이동 + 기존 `workspace-management/` 폴더 삭제 (7번 작업, 2026-06-04 완료)
    - [x] `sidebars.js` 워크스페이스 그룹 통합 — 기존 "권한 설정" + "워크스페이스 관리" 두 카테고리 → 단일 "워크스페이스" 카테고리. 순서 사용자·부서 관리 → 권한 설정 → 개인 계정. 기존 "대시보드" 카테고리 라벨 → "통계·감사" 갱신(분석본 그룹 신설 토대) (7번 작업)
    - [x] 내부 링크 일괄 갱신 — `workspace-management/` 참조 0건 확인. `workspace/permissions` 상호 참조 단순화(`../../workspace/permissions` → `../permissions`)
    - [ ] `ko/use-spx-agent/monitor/` 유지 (Analysis / Logs / Annotation Reply 평면 구조, sub-group 없음) — Phase 4 본격 작성 시
    - [x] `ko/use-spx-agent/analytics-audit/dashboard/` 생성 + 기존 `ko/use-spx-agent/dashboard/` 이동 (8번 작업, 2026-06-04 완료). sidebars.js "통계·감사" 카테고리 경로 갱신
    - [x] `ko/use-spx-agent/analytics-audit/audit-log/` 신규 — B1 인터리브 1p `readme.mdx` 작성, sidebars.js "통계·감사" 카테고리에 등록(슬롯 주석 → 실제 항목 교체), 빌드 통과 (2026-06-04 완료)
    - [ ] `sidebars.js` 시작하기·노드·빌드·게시·튜토리얼 그룹 — Phase 4 본격 작성 시

    **챕터명 일괄 갱신 (2026-06-04 rename 반영)**:
    - [x] "앱 권한 설정" → "권한 설정" 매트릭스·decisions·conventions·CLAUDE.md·progress 반영 완료
    - [x] 챕터 본문 헤더·소개 문장 "권한 설정" 명칭 반영 — 6번 작업(2026-06-04)에서 폴더 이동(`app-permissions/` → `workspace/permissions/`)과 함께 완료

    **References 후속 갱신 (Cluster B 진행 전 / 분석본 보강)** — 2026-06-04 라벨 스왑 정정 + 완료 처리 (8번 작업):
    - [x] [[references/spx-departments-management]] (**A3**) — 결정 5·6 반영 완료. 1번 작업에서 §0 KC 추상화 가이드·§1 변환 정전·§5.6 추상화 표기 예시·§8 재구성 박제
    - [x] [[references/spx-workspace-analysis]] (**A4**) — 결정 2·5·6 반영 완료. 2번 작업에서 §1.1 결정 반영 표·§2 결정표 재구성·§3.3 변환 확정·§4 API Extension 삭제 확정·자체 검토 5건 박제

    > ✅ **라벨 스왑 오류 정정** (8번 작업, 2026-06-04) — 원본 표기는 라벨이 반대(A4↔A3)로 적혀 있었음. A3=사용자/부서 관리=`spx-departments-management`, A4=Workspace 재검토=`spx-workspace-analysis`로 정정. 두 분석본 모두 1·2번 작업으로 갱신 완료된 상태였으므로 완료 처리.

    ### CI/CD 표시 가이드 (2026-06-04 결정 7)

    > spx-agent에 개발/운영 두 환경 띄울 때 앱 이관용 CI/CD가 있음. 현재 다른 분 작업 중, 매트릭스 미반영.

    - [ ] CI/CD 기능 추가 후 해당 페이지에 "이 기능은 개발 환경 전용입니다" 마커 표기
    - [ ] [[conventions]]에 환경 분리 마커 작성 규칙 추가 (CI/CD 추가 시점)
    - [ ] Twitter Chatflow / Embedding in Websites — 외부 호출 정책 따라
  - [ ] 챕터별 작업 (아래 표 참고) — 클러스터 인터리브로 분산 진행, 클러스터별 분석 → 챕터 작성 → 다음 클러스터

    ### Phase 4 본격 포팅 — 진행 로그

    #### 2026-06-08: Nodes/answer (테스트 1p)
    - [x] `ko/use-spx-agent/nodes/answer.mdx` 작성 (유지·번역)
    - [x] `sidebars.js` "노드" 카테고리 신설 + answer 등록
    - [x] `npm run build` 통과 (answer 발 broken link 0건)
    - [x] 체크리스트 전 항목 통과 (§1.5 decisions / §2.8 3패스 / §3.2 i18n 사후 검증 / §3.3 전역 규칙 grep / §3.4 메타 갱신)
    - **i18n 사후 검증 발견**: "챗플로우" → **"채팅 플로우"** 교정 (`app.json` `types.advanced` 권위). 글로서리 갱신 완료
    - 후속(deferred):
      - [ ] `answer.mdx` 내 출력(Output) 노드 텍스트 참조 → `output.mdx` 작성 시 마크다운 링크로 교체
      - [x] `key-concepts.mdx`의 "챗플로우" 9건 → "채팅 플로우" 소급 교정 (2026-06-09 완료)

    #### 2026-06-08: Nodes/ifelse (테스트 2p)
    - [x] `ko/use-spx-agent/nodes/ifelse.mdx` 작성 (유지·번역)
    - [x] 체크리스트 전 항목 TODO로 펼치고 순서대로 통과
    - [x] i18n 비교 연산자 라벨 검증 (포함/시작/이다/비어 있음 등 `workflow.json` 일치)
    - [x] 직역체 M1~M11 대조 — grep 0건 + 소리 내 읽기(M2·M10) 클린
    - [x] `npm run build` 통과, 전역 규칙 grep 클린, progress 갱신
    - deferred 없음

    #### 2026-06-08: Nodes 배치 1 (5p)
    - [x] user-input, trigger/overview, trigger/schedule-trigger, trigger/webhook-trigger, llm 작성
    - [x] 전역 규칙 적용: trigger/overview SaaS 콜아웃 2건·Plugin Trigger 섹션 삭제, llm "Dify" 2곳 제거, webhook-trigger self-hosted 프레이밍·plugin 링크 제거
    - [x] 교정: user-input M5 "로부터"→제거, user-input·llm "(Chatflow)" 병기 추가
    - [x] `npm run build` 통과, 전역 규칙·직역체 grep 클린
    - deferred: webhook-trigger → variable-aggregator dangling link (배치 3에서 해소)

    #### Nodes 배치 2 (6p), 배치 3 (5p)
    - [x] 배치 2: knowledge-retrieval, output, agent, question-classifier, human-input, iteration 작성
    - [x] 배치 3: loop, code, template, variable-aggregator, variable-assigner 작성
    - [x] deferred 해소: webhook-trigger → variable-aggregator dangling link (배치 3에서 해소)

    #### Nodes 배치 4 (5p)
    - [x] doc-extractor, parameter-extractor, http-request, list-operator, tools 작성
    - [x] 전역 규칙 적용: doc-extractor `UNSTRUCTURED_API_URL`/`UNSTRUCTURED_API_KEY` env var명 제거(규칙 #4), tools "Dify" 4곳·Plugin Dev 링크 제거(규칙 #5)
    - [x] 직역체 교정 6건: doc-extractor M2 "제공합니다"→"변환됩니다", parameter-extractor M1 "를 위한"→"용", http-request M1 "를 위한"→"~의", tools M2 "제공합니다" 3건→교정
    - [x] §2.4 3패스(원문 대조) 5p 전수 — 누락 0건
    - [x] §2.4 2패스(M2/M3/M10 수동 검토) 5p 전수 — 추가 교정 0건
    - [x] §2.5 i18n 사후 검증 — 교정 2건: parameter-extractor "오류 설명"→**"오류 원인"**(i18n `errorReason`), http-request "응답 본문"→**"응답 내용"**(i18n `outputVars.body`)
    - [x] `npm run build` 최종 통과
    - deferred: list-operator i18n `listFilter.desc` = "설명" — descending이 description으로 오역 가능성 (spx-agent i18n 측 확인 필요)

    #### Nodes 그룹 마감 ✅
    - [x] 전 23p 작성 완료 (Plugin Trigger 삭제 1건 제외)
    - [x] 빌드 최종 통과 (배치 4 발 broken link 0건)
    - [x] deferred 전부 해소 (webhook→variable-aggregator, answer→output 포함)

    #### 2026-06-08: Build 7p 일괄 작성
    - [x] `ko/use-spx-agent/build/` 디렉토리 신설, 7p 작성: shortcut-key, goto-anything, orchestrate-node, predefined-error-handling-logic, mcp, version-control, additional-features
    - [x] 전역 규칙 적용: goto-anything `@plugin` 섹션 삭제(결정 1), mcp "Dify" 브랜드 제거(규칙 #5), additional-features motionshot.app iframe 제거·marketplace 링크 제거(결정 1)
    - [x] 직역체 M1("를 위한") 2건 교정: additional-features L82, predefined-error-handling-logic L44·L78
    - [x] i18n 사후 검증: 오류 처리 전략(없음/기본값/실패 분기), 버전 관리(현재 초안/게시하기/업데이트 게시/버전 기록), 앱 기능(특징/대화 시작/팔로우업/텍스트에서 음성으로/인용 및 소유권/콘텐츠 모더레이션) — conventions 글로서리에 17항목 추가
    - [x] `sidebars.js` "빌드" 카테고리 신설 (노드와 지식 사이)
    - [x] `npm run build` 통과 — 신규 broken link 1건: mcp→../publish/publish-mcp (Publish 그룹 미작성, deferred)
    - decisions 영향: **결정 1**(goto-anything @plugin 삭제, additional-features marketplace 삭제) + **결정 3**(MCP 유지). 나머지 N/A
    - §2.4 3패스 원문 대조: 7p 전수 — 누락 0건 (의도적 삭제: iframe 2, @plugin 섹션 1, marketplace 링크 1)
    - deferred: mcp.mdx → `../publish/publish-mcp` dangling link (Publish 그룹 작성 시 해소)

    #### Build 그룹 마감 ✅
    - [x] 전 7p 작성 완료 (삭제 0건)
    - [x] 빌드 최종 통과 (Build 발 새 broken link: mcp→publish-mcp 1건, Publish 미작성으로 deferred)
    - [x] conventions 글로서리 갱신 — "빌드·버전·앱 기능" 섹션 17항목 추가
    - [x] deferred 1건 박제: mcp→publish-mcp (Publish 그룹 작성 시 해소)

    #### 2026-06-08: Tutorials 14p 일괄 작성
    - [x] `ko/use-spx-agent/tutorials/workflow-101/` 디렉토리 신설, Workflow 101 Lesson 01~10 (10p) 작성
    - [x] `ko/use-spx-agent/tutorials/` 독립형 4p 작성: simple-chatbot, customer-service-bot, build-ai-image-generation-app, article-reader
    - [x] 전역 규칙 #5 적용 (전 14p): "Dify" 브랜드명·로고·외부 링크(dify.ai, marketplace.dify.ai) 제거/교체, "Dify 101"→"워크플로우 101", Marketplace 설치 경로→일반화
    - [x] 전역 규칙 #2 적용: L04 Notion/Website sync 언급 제거 (외부 연결 삭제, 결정 2), customer-service-bot 외부 KB/website sync 섹션 제거
    - [x] 전역 규칙 #1 적용: AI Image Generation "Free version" 언급 제거 (SaaS 플랜)
    - [x] 직역체 검수 (2패스): 이중 피동("되어집니다") 1건, "를 통해" 4건, "을 위한" 1건 교정
    - [x] `<CodeGroup>` 미등록 컴포넌트 → 태그 제거 (L03, L04)
    - [x] `sidebars.js` "튜토리얼" 카테고리 신설 (통계·감사 아래), Workflow 101 sub-category + 독립형 4p 등록
    - [x] `npm run build` 통과 — Tutorials 발 broken link 0건
    - decisions 영향: **결정 2**(Twitter Chatflow 삭제 — 미작성으로 반영). 나머지 N/A
    - i18n 활용: 노드명(시작/출력/지식 검색/변수 집계자/매개변수 추출기/반복/에이전트/템플릿 등), UI 라벨(스튜디오/탐색/게시하기/앱 실행/앱 일괄 실행/테스트 실행/체크리스트/모니터링/미리보기) — `workflow.json`·`common.json` 기반

    #### Tutorials 그룹 마감 ✅
    - [x] 전 14p 작성 완료 (Twitter Chatflow 삭제 1건 제외)
    - [x] 빌드 최종 통과 (Tutorials 발 broken link 0건)
    - [x] progress 상태 표 갱신 — Tutorials 행 ✅ 완료
    - deferred:
      - [ ] L04 원본에 있던 Google Drive 다운로드 링크 제거 → 연습용 파일 준비 안내로 대체. Phase 5에서 샘플 파일 제공 검토
      - [ ] L07·L08 Marketplace/Plugin 설치 흐름 일반화 처리됨 — spx-agent 실제 도구 설치 UI와 비교 검증 필요 (Phase 5 화면 교체 시)
      - [ ] customer-service-bot 외부 이미지(assets-docs.dify.ai) → Phase 5 이미지 교체 시 내부화
      - [ ] L10 "Publish as a Tool" 항목 제거됨 — 플러그인 시스템 복원 시 재추가 검토
      - [ ] introduction.mdx 튜토리얼 카드 dangling anchor → sidebars 등록으로 경로 유효화됨, anchor 매칭은 별도 확인

- [ ] **Phase 5**: 화면/스크린샷 교체
  - [ ] 원본 이미지 인벤토리 작성
  - [ ] spx-agent 화면 캡처 가이드라인 작성
  - [ ] 챕터별 이미지 교체 (Phase 4와 병행 가능)
- [ ] **Phase 6**: 빌드·호스팅·배포
  - [ ] 호스팅 결정 사항 실행
  - [ ] 사내 git 신규 리포 생성 (옵션 따라)
  - [ ] CI/배포 자동화 (필요 시)
  - [ ] 동료/이사님 접근 URL 공유

## 챕터별 상태

> 상태 코드: ⏳ 대기 / 🔄 진행 중 / ✅ 완료 / 🚫 삭제 / ❓ 보류(검토)

### 원본 포팅 (그룹별)

| 그룹          | 페이지 수    | 유지·번역   | 부분 수정  | 삭제     | 변환    | 검토    | 상태                                                    |
| ----------- | -------- | ------- | ------ | ------ | ----- | ----- | ----------------------------------------------------- |
| Get Started | 3        | 0       | 3      | 0      | 0     | 0     | ✅ 완료 — 잔여: 신규 챕터 anchor 재검증(대상 챕터 의존) + Phase 5 스크린샷  |
| Nodes       | 24       | 23      | 0      | 1      | 0     | 0     | ✅ 완료 (23/23 번역, Plugin Trigger 삭제, 2026-06-08)        |
| Build       | 7        | 7       | 0      | 0      | 0     | 0     | ✅ 완료 (7/7 번역, MCP 유지, 빌드 통과, 2026-06-08)              |
| Debug       | 4        | 4       | 0      | 0      | 0     | 0     | ✅ 완료 (4p 번역, i18n 검증, 빌드 통과, 2026-06-05)              |
| Publish     | 9        | 6       | 1      | 2      | 0     | 0     | ✅ 완료 (7p 번역, 전역규칙 적용, 빌드 통과, 2026-06-09)             |
| Monitor     | 10       | 2       | 1      | 7      | 0     | 0     | ✅ 완료 (3p 작성+리뷰 반영, integrations 7 삭제, 2026-06-09)    |
| Knowledge   | ~23      | ~14     | 3      | 6      | 0     | 0     | 📝 본문 완료 (17p) — 사용자 검수 대기 (2026-06-10)               |
| Workspace   | 11       | 2       | 2      | 6      | 1     | 0     | ✅ 완료 (6p: 신규 3p + 인터리브 3p, 2026-06-11)               |
| Tutorials   | 15       | 14      | 0      | 1      | 0     | 0     | ✅ 완료 (14p 번역, 전역규칙 적용, 빌드 통과, 2026-06-08)             |
| **합계**      | **~103** | **~72** | **10** | **23** | **1** | **0** |                                                       |

> 2026-06-11 갱신. **원본 포팅 9그룹 전체 본문 작성 완료.** 6/4 이사님 결정 반영 (Marketplace/플러그인 제거 / 외부 연결 제거 / inbound 유지 / MCP 유지 / team-members 변환). **검토 22 → 0건**, 삭제 7 → 23건, 변환 신설 1건.
>
> ⚠️ Knowledge 페이지 수 ~23은 매트릭스 row 수 기준 (원본 표기 20과 불일치 — Phase 1.1 시점 추정치라 정정 필요 시 별도 확인). 총합 ~103도 같은 사유로 추정치.
>
> 2026-06-09 정정: **Get Started 실제 본문 작성 완료** 확인(디스크 `ko/use-spx-agent/getting-started/` 실측 — quick-start 564줄 등). 기존 "임시 1p" 기록은 미갱신 stale이었음. getting-started·nodes·tutorials 등 Phase 4 산출물은 디스크 존재하나 git 미커밋 상태였음 → **2026-06-09 커밋 완료** (5개 커밋, 워킹트리 clean).

#### 그룹별 사전 분석 요구사항 (Phase 4 시작 전)

> 분석 산출물은 **반드시 `references/spx-<주제>.md`로 박제** (CLAUDE.md 응답 규칙). 클러스터 컬럼은 위 Phase 4 사전 분석 진행 순서(A1→A2/A3→A4→B1→B2→C1) 참조.

| 그룹            | spx-agent 분석 필요 사항                                                                                            | 산출 references 문서                           | 클러스터      | 시급도                                                                                   |
| ------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------ | --------- | ------------------------------------------------------------------------------------- |
| Get Started   | 거의 완료 — Introduction 신규 챕터 키워드 티저, Quick Start 부서·권한 한 줄 위치, Key Concepts 부서·RBAC 정의                          | `spx-get-started-analysis.md` (간소화 가능)     | **C1**    | 낮음 (A·B 후행)                                                                           |
| Nodes         | ✅ Plugin Trigger 삭제 확정 (6/4)                                                                                  | (불필요)                                      | (해소)      | -                                                                                     |
| Build         | ✅ MCP 유지 확정 (6/4)                                                                                             | (불필요)                                      | (해소)      | -                                                                                     |
| Debug         | 분석 불필요                                                                                                        | -                                          | -         | -                                                                                     |
| Publish       | ✅ Marketplace 제거 / Embedding·Developing APIs 유지 확정 (6/4) + Publish Overview 권한 제약 한 줄                         | (Phase 4 본문 작성 시 한 줄 추가)                   | (해소)      | -                                                                                     |
| Monitor       | Analysis 페이지 UI 매핑 ("모니터링"), 신규 챕터 "대시보드"와 영역 구분                                                              | `spx-monitoring-analysis.md`               | **B2**    | 중간                                                                                    |
| **Knowledge** | **권한 설정 3p UI 노출 위치·필드명·가시성 4단계 매핑·ACL 부여 방식**, 외부 연결 결정 적용(외부 import·외부 KB 삭제 / Maintain via API inbound 유지) | ✅ [[references/spx-knowledge-permissions]] | **A2 완료** | 권한 부분 분석 완료, 외부 연결 6/4 결정으로 해소                                                        |
| **Workspace** | **11p 전수 재검토 (5/29 비고 vs 2026-06-02 전역 규칙)**, Personal Account 가정 정정, Workspace 구조, API Extension 삭제 (6/4)    | ✅ [[references/spx-workspace-analysis]]    | **A4 완료** | 부분 수정 4p·삭제 6p·변환 1p (team-members) — 2번 작업(2026-06-04)에서 결정 2·5·6 반영 완료, 자체 검토 5건 처리 |
| Tutorials     | ✅ Twitter Chatflow 삭제 확정 (6/4)                                                                                | (불필요)                                      | (해소)      | -                                                                                     |

### 신규 챕터 (spx-agent 전용, v2 — 2026-05-29 재구성)

| 챕터                  | 페이지(추정) | 상태       | 우선순위 | 클러스터      | 비고                                                      | Phase 4 시작 전 spx-agent 분석 필요 사항                                                                                                                                                                                                                                                                               |
| ------------------- | ------- | -------- | ---- | --------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 대시보드                | 2~3     | 🔄 임시 1p | P1   | (완료)      | KPI 카드, 부서별 리소스, 모델 토큰, 드릴다운                            | ✅ 분석 완료 → [[references/spx-dashboard-analysis]] (2차 파일럿 deep dive). 임시 1p `ko/use-spx-agent/analytics-audit/dashboard/readme.mdx` (8번 작업 이동). 본격 작성 시 확장만                                                                                                                                                     |
| 권한 설정 (구 "앱 권한 설정") | 2~3     | ✅ 완료     | P1   | **A1 완료** | 가시성 4단계, ACL 부여/취소, 소유권 (앱/지식/도구 공통, 2026-06-04 rename) | ✅ 분석 완료 → [[references/spx-app-permissions-analysis]]. `ko/use-spx-agent/workspace/permissions/readme.mdx` 작성 완료 (A1 인터리브 + Workspace 그룹 마감). 빌드 통과                                                                                                                                                           |
| 사용자/부서 관리           | 2~3     | ✅ 완료     | P2   | **A3 완료** | 부서 목록, 멤버 배정, KC 그룹 연동                                  | ✅ 분석 완료 → [[references/spx-departments-management]]. `ko/use-spx-agent/workspace/departments/readme.mdx` 작성 완료 (A3 인터리브 + KC 추상화 적용). 빌드 통과                                                                                                                                                                   |
| 감사로그                | 2~3     | 🔄 임시 1p | P1   | **B1 완료** | KAN-28, 이벤트 조회·필터·내보내기                                  | ✅ 분석 완료 → [[references/spx-audit-log-analysis]]. 별도 Next.js 앱(dify-audit) iframe 임베드(설정→감사 로그, owner/admin). 수집 5경로·이벤트 카탈로그 전수·2탭 UI·Export·테넌트 격리 박제. **인터리브 1p `ko/use-spx-agent/analytics-audit/audit-log/readme.mdx` 작성·sidebars 등록·빌드 통과(2026-06-04)** + KC 추상화(전역 규칙 #6) 완전 적용. 본격 작성은 Phase 4 본문 진입 시 |

**신규 합계: 8~12 페이지**

> **진행 순서** (2026-06-04 갱신, 본 진척표 line 110 Phase 4 진행 방식과 정합): A1 권한 설정(코어) → A2 지식 권한 ∥ A3 사용자/부서 관리 → A4 Workspace 재검토 → B1 감사로그 → B2 모니터링 → C1 Get Started. **A1~A4·B1 분석 완료 (2026-06-04)**. 다음은 B1 인터리브 1p 또는 B2 분석.

> 변경: ~~KC SSO~~ 삭제 (Quick Start에 흡수), ~~RBAC~~ → 권한 설정 + 사용자/부서 관리 분리, ~~설정 사이드바 (KAN-29)~~ 삭제 (2026-06-02, 원본 다이얼로그에 토글만 추가된 수준 → 별도 챕터 불필요)

## 작업 로그 (시간순)

> 📦 시간순 작업 로그는 파일 크기 관리를 위해 분리했습니다 (2026-06-09).
> 과거 작업 내역 전체 → [[progress-archive]]
