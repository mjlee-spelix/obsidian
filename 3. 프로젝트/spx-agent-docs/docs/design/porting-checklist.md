# UI 리디자인 — 프로토타입 ↔ 실사이트 감사/포팅 체크리스트

> 기준(target): `design/ui-redesign-prototype.html` (확정 시각 명세)
> 대상(impl): `src/css/custom.css` + `src/theme/*` + `docusaurus.config.js` + `sidebars.js`
> 작성: 2026-06-15 · **전 영역 1:1 감사 결과 기반.** 새 세션 인수인계용.
> 원칙: MDX·`docs.json`·`writing-guides` 무수정. CSS + 스위즐만. 변경 후 `npm run build` → 사용자 스샷 확인(렌더 추측 금지).
> rem 환산: 26=1.625 / 19=1.1875 / 16=1 / 15=0.9375 / 14=0.875 / 13=0.8125

## 감사 커버리지 (빠짐없이 대조한 영역)

전역토큰 · 레이아웃 · 타이포 · 사이드바 · TOC · 콘텐츠헤더 · 콜아웃 · 코드 · 표 · 탭 · 카드 · 스텝 · 아코디언 · 프레임 · KPI · 페이지네이션 · 기본마크다운 · 다크모드 · 반응형/모바일 · 인터랙션(JS). → 아래는 **MATCH 제외, 조치 필요(DIFF/MISSING)만** 정리.

---

## ✅ 이미 완료
폰트(Pretendard/IBM Plex Mono) · 컬러토큰(인디고/슬레이트) · 콜아웃 5종 재색(틀) · 기본요소(blockquote/hr/img/h4~6/중첩목록) · 복사버튼 제거 · Studio 제거 · 검색텍스트 · eyebrow(상위 카테고리) · 접기(hideable) · active/hover 짧은바(틀) · 컴팩트 페이지네이션 · 사이드바 그룹간격 · 다크토글 아이콘화 · 페이지네이션/blockquote/hr/img/표/탭 색.

---

## P1 — "달라 보임"의 최대 원인 (먼저)

### 타이포 스케일 (`custom.css`)
- [x] `.markdown h1:first-child` 30px/600 → **26px/700** (line-height 1.25, `!important` 유지)
- [x] `.markdown h2` 24px/600, mt 48 mb 16 → **19px/650**, mt 44 mb 14, ls -0.012em
- [x] `.markdown h3` 20px, mt 48 → **16px**, mt 26 mb 10
- [x] `.markdown h4` 16px, mt 28 → **15px**, mt 22 mb 8
- [x] `.page-description`(lead) 18px, gray-500 → **16px**, line-height 1.7, **slate-400(`--mint-gray-400`)**
- [x] `.page-eyebrow` 14px → **13px** (mb 12)
- [x] 본문 `p` margin 20px → 18px (선택)
- [x] 첫 h2 상단 간격 축소 — `.markdown > h2:first-child { margin-top: 10px }` 적용
- [x] 다크 제목색 `#ffffff` → **`#F8FAFC`(off-white)** 통일 (h1~h4)

### 본문 폭 / 컨테이너 (`custom.css`)
- [x] `article` max-width 576/672 → **720px** (2xl 분기 제거, margin-inline auto)
- [x] `--ifm-container-width: 10000px` → **1480px** 중앙 정렬
- [x] 반응형 본문 패딩 — ≤1100: 40, ≤720: 20 (docItemCol)

### 배경/표면 (`custom.css`)
- [x] `--ifm-background-color` → **paper `#FBFBFD`**, `--surface #FFFFFF` 토큰 신설 (콜아웃/코드 틴트 베이스를 surface로 분리)
- [x] border → **slate-200 `#E2E8F0`(불투명, `--mint-gray-200`)**, 다크 `#2A2F3C`

### Radius 드리프트 (`custom.css`)
- [x] 콜아웃·카드·코드·아코디언·KPI **14px**로 통일 (`--spx-radius` 토큰 신설, `--ifm-global-radius`는 16 유지)

---

## P2 — 컴포넌트 디테일

### 콜아웃 (`custom.css .mintlify-callout*`)
- [x] 패딩 16/20 → **14/18**, 폰트 16 → **15px**, radius → 14
- [x] info 블루 `#3b82f6` → **`#2563eb`** (border/bg/icon)
- [x] warning border `#d97706` → **`#b45309`**
- [x] 틴트 베이스 `--ifm-background-color` → **`--surface`** (라이트/다크 자동 대응)

### 코드블록 (`custom.css`)
- [x] 2px 외곽 프레임 제거 → **단일 surface 카드**(border 1px, radius 14)
- [x] 제목바 **인디고 색점** 추가(`codeBlockTitle::before`) + mono 12px slate-400
- [ ] ⏸ `wrap` 메타 remark 매핑 (기능, deferred — CSS 무관)
- [ ] ⏸ 공백 제목 메타 `title=""` 변환 remark (기능, deferred)

### 카드 (`custom.css .mintlify-card*`)
- [x] CardGroup **2열 grid-template** 추가(`1fr 1fr`)
- [x] 링크 hover 그림자 → **인디고**, `translateY(-1px)` 리프트, radius 14
- [x] 카드 아이콘 **둥근 indigo-50 타일(30×30, radius 9)** 복구 (`--indigo-50` 토큰 신설)
- [x] desc 색 slate-400, 제목 600/ink

### 스텝 (`custom.css .mintlify-step*`)
- [x] 번호 원 **28→24px**, 숫자 **mono** 적용
- [x] 연결선 → **1.5px, 원 중심(11px)→다음 원 통과** geometry
- [x] 원 bg → slate-100 (다크 #1A1E27)

### 아코디언 (`custom.css .mintlify-accordion*`)
- [x] radius →14, summary 패딩 16/20 → 13/18, 제목 → **600/15px**
- [x] 본문 좌측 들여쓰기 0 → **40px**, chevron 색 → slate-400

### 프레임 (`custom.css .mintlify-frame*`)
- [x] 도트 그리드 13px(slate-200), 내부 img radius 12→10

### 탭 (`custom.css .tabs__item`)
- [x] 패딩 12/0 → 9/0, **비활성 색 → slate-400**, hover 밑줄 프리뷰 제거(색만)

### 표
- [x] th 색 ink 명시(다크 대응), 셀 패딩 10/14

### KPI ⚠️ (콘텐츠에 실제 사용 — 대시보드)
- [x] `.kpi / .kpi__label / .kpi__value / .kpi__delta(up/down)` **CSS 신설** (프로토타입 값 이식). 단, 현재 MDX는 프로즈·스크린샷으로 설명하고 HTML 위젯 미사용 — 향후 사용 대비 추가

---

## P3 — 사이드바 / TOC 디테일

### 사이드바 (`custom.css` + `DocSidebar/Desktop.jsx` + `sidebars.js`)
- [x] 레일 → **slate-200 1.5px** (`--ifm-border-color`)
- [x] active/hover 바 높이 16 → **18px**
- [x] active 텍스트 → **600** (text-shadow 제거)
- [x] 그룹 헤더 → **600**, hover **bg(slate-100)+radius7**
- [x] 항목 색 → **slate-400**
- [x] ✅ **chevron 좌측 배치** — 사용자 결정: **우측 유지 (Docusaurus 기본)**. 변경 없음
- [x] 그룹 간격 6px → 10px
- [x] 검색박스: → **1px border/radius9**, kbd **칩 스타일**(border, pad 3/7)
- [x] 로고: 현 이미지 유지 (동일 노드 그래프, 변경 불필요)
- [x] ✅ **카테고리 펼침 기본값** — 사용자 결정: **활성 그룹만 펼침 (현행)**. 변경 없음
- [x] ✅ 접기 컨트롤 — 사용자 결정: **현행 유지 (하단버튼+좌측띠)**. 변경 없음

### TOC (`custom.css` + `TOC/index.jsx`)
- [x] **"이 페이지" 라벨 추가** (스위즐 TOC 헤더)
- [x] 컨테이너 좌측 divider(border-left) 추가 (`.theme-doc-toc-desktop`)
- [x] 링크 색 → slate-400, 바 높이 → 18, active weight → 600

---

## P4 — 모바일 / 검색 (별도 라운드)

- [x] **모바일 사이드바 접근** ⚠️: `DocRoot/Layout` 스위즐로 햄버거+scrim+드로어 상태 주입. ≤996px에서 데스크탑 사이드바 컨테이너(`.theme-doc-sidebar-container`)를 fixed 오버레이 드로어로 재활용(Docusaurus `display:none` 덮어씀). 라우트 변경/ESC/scrim 클릭 시 닫힘, body 스크롤 잠금, 본문 상단 햄버거 클리어 여백
- [x] 반응형 브레이크포인트: 드로어는 Docusaurus 기준 **996px** 채택(사이드바 컨테이너 display 전환점과 일치). 본문 패딩은 1100/720 유지
- [x] 검색 전반 재작업 완료 → **P20 참조**. ⚠️ 이 항목의 전제(`.DocSearch-*`)는 **오류**였음 — easyops는 DocSearch/Algolia 미사용, **인라인 자동완성**(`.navbar__search` + autocomplete.js 드롭다운). 실제로는 검색이 **navbar 숨김으로 작동 자체가 안 되던 것**을 발견·수정하고, 모달·드롭다운 디자인까지 완료
- [~] **모바일 드로어 기능 결함은 해소**(P8 #7: 빈 화면 → 항상 Desktop 렌더). 실기기 픽셀 검증만 미완(모바일 스샷 미수신)

---

## P5 — 정리 / 콘텐츠 (낮음)

- [x] 죽은 CSS 제거: `.copy-page-btn`, `.sidebar-header__toggle-track/thumb/icon-*` 제거 완료 (JSX 미참조 확인)
- [x] broken anchor: `key-concepts` "변수" 헤딩에 `{#variables}` 부여
- [x] broken anchor: `knowledge/readme` "지식 생성" 헤딩에 `{#create-knowledge}` 부여
- [x] broken link: `knowledge-retrieval` 링크 `../knowledge/readme` → `../knowledge/readme.mdx`(파일 기반, 폴더 인덱스 라우팅 대응). **빌드 broken link/anchor 0건 확인**

---

## P6 — 잔여 1:1 재대조 (2026-06-15, P1~P3·P5 적용 후 프로토타입 전체 재대조)

> P1~P5 적용분을 프로토타입과 한 줄씩 다시 대조해 발견한 차이. **수정 권장(정합성)** / **선택(낮음)** / **무시 가능(미세)** 으로 분류.

### 정합성 — 수정 완료 ✅
- [x] **본문 기본 텍스트 색**: `li` 등 비-`p` 텍스트가 Docusaurus 기본색(near-black)이라 `p`(ink-700 `#3A4053`)와 불일치 → `--ifm-font-color-base: ink-700`(다크 gray-200) 지정
- [x] **첫 h2 간격 셀렉터 버그**: `.markdown > h2:first-child`는 앞에 `header(h1)`·`doc-header-inject`가 있어 **절대 매치 안 됨** → `.doc-header-inject + h2` / `header + h2`로 보정 (프로토타입 `.lead + h2` = mt 10px 재현)
- [x] **Frame 캡션**: 프로토타입은 border-top 없음 + slate-400 + padding 8/0/2 → 현재 border-top·gray-600 제거하고 정합 (P2 frame 잔여분)

### 정합성 — 수정 완료 ✅
- [x] **다크모드 muted 색(slate-400)**: `[data-theme='dark'] { --mint-gray-400: #7C8AA0 }`로 프로토타입 다크값 통일 → 사이드바·TOC·캡션·KPI 라벨·카드 desc·탭 비활성 일괄 적용

### 선택 — 적용 완료 ✅
- [x] **인라인 코드**: `:not(pre) > code`로 인라인 전용 스코프, `solid slate-100 / radius 5 / pad 1·6 / 12.5px / ink-700`(다크 #1A1E27) 적용
- [x] **헤딩 scroll-margin-top**: h2~h4에 `scroll-margin-top: 32px`
- [x] **사이드바 헤더 패딩**: 18/20/12 + gap 14px
- [x] **테마 토글 hover 색**: `ink`로 변경(다크 gray-50)
- [x] **사이드바 nav 얇은 스크롤바**: 6px slate-300 thumb (webkit + firefox)
- [x] **Pagination 화살표 아이콘**: `PaginatorNavLink` 스위즐 — prev `‹` / next `›` chevron 추가(순서 JSX 제어로 row-reverse 충돌 회피)

### 무시 가능 (미세 수치, 인상 영향 없음)
- tabs gap 24 vs 프로토타입 22 · callout icon mt 2 vs 3 · code pre 14 vs 13.5px · 표/스텝 margin 미세차

---

## P7 — 실렌더 자체 점검 (2026-06-15, localhost:3000 스샷 대조)

> 실제 렌더를 프로토타입과 대조하다 발견한 **P6에 없던 결함**. (스샷: knowledge 페이지에서 H1 "지식" 아래 회색 "소개"가 중복으로 노출되던 문제에서 출발)

### 수정 완료 ✅
- [x] **description 자동추출 누수 버그** (가장 큼): frontmatter에 `description`이 없는 페이지(88p 중 43p)에서 Docusaurus가 본문 첫 헤딩/문장을 `metadata.description`으로 자동 추출 → 스위즐 `DocItem/Content`가 이를 lead로 렌더해 **첫 헤딩이 회색으로 중복 노출**. → `metadata.description` 대신 **`frontMatter.description`(작성자 명시값)만** 렌더하도록 변경 ([Content/index.jsx](/src/theme/DocItem/Content/index.jsx))
- [x] **lead 없는 페이지 첫 h2 간격**: description 누수 수정으로 43p가 H1→H2 직결 → `header + h2`는 20px(과하지 않게), `doc-header-inject + h2`(lead 있음)는 10px로 분리
- [x] **TOC "이 페이지" 라벨 위치 결함**: 스위즐이 라벨을 sticky 컨테이너 **밖** 형제로 둬서 ① 스크롤 시 라벨이 사라지고 ② divider·rail 정렬 어긋남 → `.theme-doc-toc-desktop::before`로 sticky 컨테이너 **안에** 주입(스크롤 고정 + rail 정렬). TOC 스위즐은 pass-through로 환원

### 확인 → 수정 완료 ✅
- [x] 사이드바 **하단 글리프** 정체 = Docusaurus 기본 `IconArrow`의 **이중 화살표(`»`, fill #7a7a7a)**. 접기/펼치기 버튼이 공유 → `@theme/Icon/Arrow` 스위즐로 **단일 chevron(stroke currentColor)** 교체. 기준 방향 `›`(오른쪽)으로 두면 회전 로직상 접기 `‹`·펼치기 `›`로 프로토타입 가장자리 핸들과 일치. 버튼 flex 중앙정렬 + 기본 margin-top 오프셋 제거

---

## P8 — 사용자 실사용 피드백 라운드 (2026-06-15, 스샷 9건)

> localhost 실사용 중 발견한 동작/디테일 이슈. 일부는 스위즐 동작 버그.

- [x] **#7 모바일 드로어 빈 화면** (버그): `DocSidebar`가 창 크기로 Desktop/Mobile 분기 → 모바일은 navbar 의존 Mobile 사이드바(빈 상태) 렌더. `DocSidebar/index` 스위즐로 **항상 Desktop 렌더** → 드로어에 메뉴 정상 표시
- [x] **#3 eyebrow = 실제 폴더 경로** (버그+개선): CATEGORY_LABELS 단일라벨 추측 → `useSidebarBreadcrumbs`로 실제 경로(`지식 / 지식 생성`). eyebrow를 `DocItem/Content`의 markdown 안으로 이동 → **H1과 좌측 정렬** 일치. 구분선(`/`) 스타일 추가
- [x] **#1 접기 버튼 → edge-tab**: Docusaurus 하단 버튼 숨김, `DocSidebar/Desktop`에 우측 가장자리 중앙 핸들 렌더(`props.onCollapse`). 컨테이너 `clip-path:none`로 바깥 돌출 허용
- [x] **#8 완전 접힘 + 펼치기 핸들**: `docSidebarContainerHidden width:0`로 잔여폭 제거, 펼치기 버튼을 화면 좌측 가장자리 `position:fixed` edge-tab으로
- [x] **#2b 회색 바 = active 바 크기**: 풀높이 보더 레일 제거 → `::before` 18px 바로 통일(기본 회색→active 인디고). 사이드바·TOC 공통
- [x] **#2a L2 레벨 정렬**: 하위 카테고리(지식 생성)가 형제 리프(개요)보다 덜 들여써져 다른 레벨처럼 보임 → 좌측 정렬 일치
- [x] **#4 H1↔첫 콘텐츠 간격**: lead 없는 페이지에서 H1 바로 뒤 첫 요소(콜아웃 등) `margin-top: 24px` (margin collapse 회피)
- [x] **#6 TOC 항목 간격 축소**: line-height 1.7→1.45, 링크 패딩 5→3px
- [x] **#5 탭 활성바·구분선 겹침**: 구분선 2px + 탭 보더 2px + `margin-bottom:-2px` → 동일 위치·두께로 정확히 포개짐
- [~] **데스크탑은 P9~P15 사용자 스샷으로 검증·반복 조정 완료**(edge-tab 접기/펼치기 포함). 모바일 실기기 스샷만 미검증

---

## P9 — 사용자 피드백 2차 (2026-06-15, 스샷 7건)

- [x] **#1 메뉴 좌측 정렬**: 그룹 헤더가 검색 박스보다 왼쪽 → nav에 `padding 4px 20px 24px`, `.menu padding:0` (검색 박스 20px와 정렬)
- [x] **#2 하단 잔재**: 죽은 CSS `.sidebar-bottom*`·`.sidebar-header__title` 제거 (실물 잔재 잔존 시 추가 스샷 필요)
- [x] **#3 edge-tab 배경**: `var(--surface)` → `var(--paper)` (화면 배경과 동일)
- [x] **#4 접힘 시 헤더 노출 차단 + 펼치기 중앙**: `docSidebarContainerHidden .sidebar-mintlify { opacity:0; pointer-events:none }`로 로고·다크토글·검색·메뉴 숨김. 펼치기 핸들 `position:fixed; top:50%` **!important**로 중앙 고정(상단 올라오던 문제)
- [x] **#5 좌측 슬라이드 모션**: `.sidebar-mintlify`에 `transform translateX(-16px)`↔0 + transition
- [x] **#6 접힘 시 본문 확장**: `docMainContainerEnhanced article { max-width: 820px }`
- [x] **#7a 수직 정렬**: `.theme-doc-markdown margin-top 0`(eyebrow/H1 상향) + `.theme-doc-toc-desktop padding-top 40px`(이 페이지 라벨을 검색 높이로)
- [x] **#7b TOC 라벨 간격**: `::before margin-bottom 12→6px`
- [x] **위치 항목** — #1 정렬·#7a 3컬럼 상단 높이(검색=H1=이페이지)는 P9~P14에서 사용자 스샷으로 반복 조정·수용 완료

---

## P10 — 사용자 피드백 3차 (2026-06-15, 스샷 9건)

> 다수가 **specificity/CommonMark 동작 버그**. 원인 규명 후 수정.

- [x] **#2 하단 잔재 = Docusaurus 접기 버튼**: `@media(≥997) .collapseSidebarButton{display:block!important}`가 내 `display:none!important`와 동률 → 소스순서로 Docusaurus 승. `.theme-doc-sidebar-container` 접두로 specificity 높여 확실히 숨김
- [x] **#3 접힘 시 hover 어두워짐 = 펼치기 버튼이 전체 덮음**: Docusaurus `.expandButton{width:100%;height:100%}`가 내 24px 오버라이드를 이김 → specificity↑ + `width/height !important`로 24×46 핸들 강제
- [x] **#8 마크다운 굵게 미적용 = CJK CommonMark 이슈**: `**...(HTTP)**를`처럼 닫는 `**` 앞 괄호+뒤 한글이면 강조 안 닫힘 → `remark-cjk-friendly` 설치, config async화해 docs remarkPlugins 주입. **`<strong>` 정상 렌더 확인**
- [x] **#7 이전/다음 «/» 제거**: infima `.pagination-nav__label::before/after{content:'«'/'»'}` → `content:none` (우리 chevron과 중복)
- [x] **#6 콜아웃 아이콘**: 제각각 크기 fill 아이콘 → 프로토타입 **일관 18×18 outline**(stroke) 5종 교체, margin-top 3px
- [x] **#5 표 배경**: Docusaurus 줄무늬(`tr:nth-child(2n)`) 제거 → 프로토타입처럼 행 배경 투명(th만 slate-100)
- [x] **#9 TOC 없을 때 페이지네이션 폭**: `.pagination-nav max-width 720`(접힘 820) + margin auto로 본문과 정렬
- [x] **#1 chevron 삐져나옴**: nav 좌우 패딩이 폭 고정 `.sidebar`를 넘치게 함 → 패딩을 `.menu`로 이동(검색 박스 20px 정렬, caret 우측 정렬)
- [x] **#4 텍스트 수직 정렬**: article·TOC `padding-top 44px` 동일 → eyebrow·이 페이지·검색 높이 맞춤
- [x] **#3 본문 좌우 여백·#1·#4 픽셀 정렬** — P12(#3 중앙정렬)·P13(#5 여백 균형)·P14에서 해소

---

## P11 — 사용자 피드백 4차 (2026-06-15, 스샷 7건)

- [x] **#1 랜딩 hero 제거**: `ko/index.mdx`(slug:/)가 hero를 렌더 → `<Redirect to=getting-started/introduction>`로 교체(hero 없이 즉시 이동, `/` 유효 유지해 로고 링크 안 깨짐). redirect 플러그인 제거
- [x] **#6 헤딩 # 앵커 제거**: `.hash-link { display: none }`
- [x] **#5 사이드바·TOC 폭 축소**: `--doc-sidebar-width 304→280px`, TOC `col--3 19→16rem`
- [x] **#4 펼치기 핸들 = 접기 핸들 디자인 통일**: 접기 아이콘 16→18px(동일), 펼치기 버튼 bg/border/radius를 high-specificity로 강제(모듈 기본 덮음)
- [x] **#2 chevron 위치** — P12(caret background-position right)~P14에서 정렬 반복 조정 완료
- [x] **#7 탭 활성바·구분선** — **P15 박스형 탭 전환으로 밑줄/구분선 자체 제거 → 갭 이슈 종결**
- [x] **#3 접힘 시 본문 좌우 여백** — P12(#3 남은 영역 중앙정렬)·P13(#5 여백 균형)에서 해소

---

## P12 — 사용자 피드백 5차 (2026-06-15, 스샷 6건)

- [x] **#4 Card 링크 빈 박스**: `<Card href>`=`<a>`(inline)에 블록 자식 → 테두리 쪼개짐. `.mintlify-card { display: block }`
- [x] **#5 본문↔TOC 경계 divider 제거**: `.theme-doc-toc-desktop` border-left 삭제
- [x] **#2 탭 활성바·구분선 겹침(재시도)**: 음수 margin이 flex align-stretch와 충돌해 갭 발생 → 구분선을 `.tabs::after`(bottom 2px)로 깔고 활성 보더를 `z-index:1`로 같은 위치에 겹침 (확실한 방식)
- [x] **#1 사이드바 폭 축소 + caret 정렬**: `--doc-sidebar-width 280→264px`, 메뉴 패딩 16px, caret `background-position right`(인셋 축소)
- [x] **#3 접힘 시 (전체−TOC) 중앙 정렬**: `docItemWrapperEnhanced max-width:none` → 본문 컬럼이 남은 영역 차지, article margin auto로 중앙
- [x] **#6 트리형 레벨 표시(1차 시안)**: 중첩 ul에 세로 가이드선 + 각 항목 ㄴ 가로 연결 tick. active=인디고 tick
- [x] #1 좌우 대칭(P14)·#2 탭(P15 박스형)·#6 트리(P13 제거) 모두 해소

---

## P13 — 사용자 피드백 6차 (2026-06-15, 스샷 3건)

- [x] **#2 트리 제거**: P12 트리 가이드 CSS 전부 롤백 (기존 rail/바 복귀)
- [x] **#1 사이드바 폭 추가 축소**: 264→240px
- [x] **#3 탭 활성바·구분선**: 활성 표시도 `::after`(bottom:0/height:2px)로 → 구분선 `.tabs::after`와 동일 위치·두께로 z-index 겹침 (border-box 차이 제거)
- [x] **#4 링크 카드 밑줄 제거**: `.mintlify-card--link text-decoration:none !important` (`.markdown a` 밑줄 무력화)
- [x] **#5 본문 좌우 여백 균형**: TOC 측 과잉 패딩(col 24+toc 22=46px) 제거 → 본문↔사이드바, 본문↔TOC 여백 균형. 본문 패딩 60→48px
- [x] #1 정렬(P14)·#3 탭 겹침(P15 박스형) 해소

---

## P14 — 사용자 피드백 7차 (2026-06-15, 스샷 2건)

- [x] **#1 검색 박스↔목록 정렬**: 헤더 패딩 20→16px(메뉴와 동일) + 사이드바 nav **스크롤바 숨김**(우측 스크롤바 폭이 빠져 비대칭이던 문제 제거)
- [x] **#3 링크 카드 hover**: 그림자·리프트 제거 → 페이지네이션처럼 **테두리 색만** 변경
- [x] **#4 TOC 우측 여백**: col--3 16→13rem 축소
- [x] **#2 탭 활성바·구분선 갭** — **P15 박스형 탭 전환으로 종결**(밑줄/구분선 자체 제거, 갭 원천 소멸)

---

## P15 — 박스형 탭 (2026-06-15)

> 밑줄·구분선 정렬 갭과 씨름하던 탭을 **박스형**으로 전환(사용자 결정). 갭 문제 원천 제거.

- [x] 프로토타입 HTML 탭을 박스형으로 교체 (테두리 박스 + 헤더 바 + 패널, 활성 = 인디고 pill)
- [x] 헤더 바 배경을 **표 헤더(th)와 동일**하게: `mint-gray-100`(다크 `mint-gray-800`)
- [x] Docusaurus 포팅: `.tabs-container`=박스, `.tabs`(ul)=헤더 바, `.tabs__item`=pill(활성 indigo-50+인디고), `.tabs-container > .margin-top--md`=패널 패딩. 기존 ::after 구분선/활성바 전부 제거 → 갭 이슈 종결

---

## P16 — 박스형 탭 1차 조정 (2026-06-15)

- [x] **헤더↔본문 간격 축소**: 패널 상단 패딩 16→12px + 첫 요소 `margin-top:0`
- [x] **활성 탭 배경 제거**: indigo-50 pill 제거 → **인디고 텍스트만**으로 활성 표시 (hover도 배경 제거)
- [x] **탭 구분 "/"**: `.tabs__item + .tabs__item::before { content:"/" }` (슬레이트, `일반 / 부모-자식`)
- [x] 프로토타입 HTML 동일 반영

---

## P17 — 박스형 탭 2차 + 설명문 + 제목 크기 (2026-06-15)

- [x] **탭 헤더↔본문 간격 더 축소**: 패널 상단 패딩 12→8px
- [x] **탭 "/" 중앙 배치**: 탭 gap 0, 항목 패딩 `4px 0`, `/`에 좌우 9px 동일 여백
- [x] **설명문(lead) 색 버그**: `.markdown p`(specificity 더 높음)가 `.page-description` 색을 덮어 어둡게 나옴 → `.markdown .page-description`로 접두 높여 **프로토타입 슬레이트(slate-400) 고정**
- [x] **본문 제목 크기 1차 확대**: h1 26→28 / h2 19→21 / h3 16→17 / h4 15→16px (프로토타입 동일)

---

## P18 — 제목 크기 추가 확대 (2026-06-15)

- [x] **h2~h4 추가 확대(본문 16px와 구분 강화)**: h2 21→**24** / h3 17→**19** / h4 16→**17px** (프로토타입 동일). 단계 5px씩 벌어져 위계 또렷

---

## P19 — 라이선스 출처 표기 보강 (2026-06-15)

> 조사 결론: footer의 "Based on Dify Docs (CC BY 4.0)"는 **CC BY 4.0 출처 표기 의무**(License §3.a) 이행을 위해 필요. 삭제 금지.

- [x] **footer 출처 표기에 링크 추가** ([docusaurus.config.js](/docusaurus.config.js)): `Based on [Dify Docs](upstream repo) (modified), licensed under [CC BY 4.0](license)` — 원자료 URI + 수정 사실 + 라이선스 URI 4요건 충족
- [x] **NOTICE.md Attribution 항목 갱신**: 배포 사이트 footer가 게시 매체에 대한 출처 표기를 함께 제공함을 명시

---

## P20 — 검색 기능 복구 + 드롭다운/모달 재설계 (2026-06-15~16)

> 발단: 검색창 클릭 시 아무 반응 없음. 조사 결과 **검색이 작동 자체가 안 되던 상태**였고(navbar 숨김 부작용), 복구 후 Mintlify 톤으로 드롭다운·모달까지 재설계.
> 핵심 제약: easyops 검색은 **전용 Web Worker**(`new Worker('./worker.js')`)라 외부 컴포넌트에서 재사용 불가 + **프로덕션 빌드에서만 작동**(`npm run build`→`serve`, dev 모드 X). 그래서 모달은 easyops `<SearchBar>`를 **재사용**(A안, 사용자 확정).

### 기능 복구 (가장 큰 건)
- [x] **검색 미작동 원인**: navbar를 `display:none`(custom.css:96) + `navbar.items:[]`로 숨겨, 검색 모달/Ctrl+K 리스너를 가진 `SearchBar`가 **마운트 자체가 안 됨**. 사이드바 커스텀 버튼은 허공에 Ctrl+K만 쏘고 있었음 → **사이드바 헤더에 실제 `<SearchBar/>` 마운트** ([DocSidebar/Desktop.jsx](/src/theme/DocSidebar/Desktop.jsx))
- [x] **Ctrl+K 먹통 = 중복 input**: Docusaurus가 navbar에 검색바를 **자동 주입** → 숨은 두 번째 SearchBar가 Ctrl+K 포커스를 가로챔. navbar 자동 검색바를 null 스위즐로 제거 ([Navbar/Search](/src/theme/Navbar/Search/index.jsx))
- [x] **박스 깨짐**: autocomplete.js가 input을 `<span>`(class=`searchBar` 해시, ≠`algolia-autocomplete`)으로 감싸 inline-block이 되며 width 붕괴 → `.navbar__search > span`을 block+full-width 강제

### 설정·i18n ([docusaurus.config.js](/docusaurus.config.js) / [i18n/ko/code.json](/i18n/ko/code.json))
- [x] `searchBarPosition: 'left'` — 좌측 사이드바라 드롭다운 좌측 정렬(기본 우측은 화면 밖으로 넘침)
- [x] `searchResultLimits: 12` — 모달 풍성하게 (기본 8)
- [x] `explicitSearchResultPath: true` — 전체 브레드크럼 경로 표시 (Mintlify식)
- [x] i18n 한국어화: placeholder "검색", /search 페이지 문구("… 검색 결과"·"문서 검색"·"문서 N개"·"검색 중…")
- [~] **색인 유형 제한**(제목+헤딩만)은 **보류**: `scanDocuments.js`(node_modules) 수정 필요 → 영속화에 patch-package 필요한데 사용자가 의존성 추가 거부 → **원복**(전체 색인 유지)

### "모든 결과 보기" → 모달 (A안: easyops SearchBar 재사용)
- [x] [Root 스위즐](/src/theme/Root/index.jsx): `.hitFooter a` 클릭을 **capture 단계**에서 가로채 /search 이동 차단 → 모달 오픈(쿼리 추출)
- [x] [SearchModal](/src/components/SearchModal/index.jsx): 중앙 오버레이 패널에 `<SearchBar>` 재사용. flex 체인(입력 고정 + 결과 스크롤) · 쿼리 프리필(autocomplete 초기화 race 회피 2회 주입) · ESC/배경클릭/스크롤락 · **라우트 변경 시 자동 닫기**(결과 클릭=SPA 이동인데 모달이 안 닫혀 "이동 안 됨"처럼 보이던 것) · 상단 우측 닫기 × · 하단 키 힌트
- [x] /search 페이지([SearchPage 스위즐](/src/theme/SearchPage/index.jsx)): 직접 진입 대비 좌상단 "뒤로" 버튼 유지

### 드롭다운/모달 디자인 (Mintlify 톤 + 인디고 — [custom.css](/src/css/custom.css))
> 모듈 클래스가 **해시**되어 직접 타겟 불가 → `--search-local-*` CSS 변수 + `[class*='...']` 부분일치 셀렉터로 제어.
- [x] mark 하이라이트: 브라우저 기본 노랑 → **인디고 틴트**(`--indigo-50`+인디고). 본문 mark 미사용이라 전역 안전
- [x] 드롭다운: surface 배경·1px 슬레이트 테두리·radius 14·인디고 커서 틴트(솔리드X)
- [x] 결과 행: 화살표(↵) 제거(Mintlify는 footer만) · 아이콘 16px(20→) · 제목 줄 정렬 · 제목 600 · 행 패딩 10 · 행 간격 2px
- [x] **트리 자식만 경로 숨김**: `:has(> [class*='hitTree'])` — 최상위 행은 전체 경로 유지
- [x] ⚠️ **중대 버그 교훈**: `[class*='suggestion']`은 행(`suggestion`)뿐 아니라 **컨테이너(`suggestions` 복수형)까지 매칭** → `:has`가 컨테이너에 걸려 전 경로 숨김. 행 전용 규칙엔 **반드시 `:not([class*='suggestions'])`**
- [x] 빈 결과: 아이콘 40→26px, 패딩 상하좌우 균등
- [x] 사이드바 드롭다운: `max-height:calc(100vh-130px)`+스크롤 (12개일 때 뷰포트 넘쳐 잘리던 것)
- [x] 모달: 입력/푸터 동일 높이(52px) + **세로 중앙 정렬**(화면 상·하 여백 대칭)

### 남은 것
- [ ] **모바일 실기기 검증** — 모달·드롭다운 실기기 픽셀 확인(스샷 미수신, P4 모바일과 함께 차단)
- 데스크탑은 스샷 반복 대조로 검증 완료

---

## 현재 탭/타이포 최종값 (P15~P18 누적)

> 박스형 탭 + 확대된 제목. 새 세션 참고용 스냅샷.

| 항목 | 값 |
|------|-----|
| 탭 컨테이너 | 테두리 박스(radius 14, surface bg) |
| 탭 헤더 바 | 배경 = 표 th(`mint-gray-100`/다크 `mint-gray-800`), padding 7/12, border-bottom |
| 탭 버튼 | 텍스트만(배경 없음), 활성 = 인디고 텍스트, 사이 `/`(좌우 9px) |
| 탭 패널 | padding 8/18/14 (헤더와 8px 간격) |
| 제목 | h1 28 / h2 24 / h3 19 / h4 17 / 본문 16px |
| 설명문(lead) | 16px / slate-400 / line-height 1.7 |

---

## 권장 실행 순서

1. **P1 전체** (타이포·본문폭·배경·radius) → 스샷 대조. 여기서 인상 80% 좁혀짐
2. **P2** 컴포넌트(KPI·카드·스텝·콜아웃·코드 순)
3. **P3** 사이드바/TOC 디테일 + 펼침/chevron 결정
4. **P4** 모바일·검색 (별도)
5. **P5** 정리

각 단계: `custom.css`(또는 해당 스위즐) 편집 → `npm run build` → 사용자 스샷 → 체크 → 다음.
전체 원본 감사표(MATCH 포함 상세)는 본 세션 로그 참조.
