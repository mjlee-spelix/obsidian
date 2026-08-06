# Mintlify 원본 스타일 기준값 (Docusaurus 복제용 baseline)

> 측정일: 2026-06-08 / **라이트모드 기준** / 측정 페이지: getting-started/introduction + quick-start
> 측정 방법: [[mintlify-to-docusaurus#14. 스타일 픽셀 검증 절차 (Mintlify ↔ Docusaurus computed-style diff)]] §14.5 스니펫
> 목적: 원본(Mintlify) 실측 computed 값을 박제 → `src/css/custom.css` 수정의 **정답지**. 재측정 없이 이 노트만 보고 수정 가능.

## 측정 환경

| | 원본 (기준) | 대상 |
|--|-----------|------|
| 도구 | Mintlify `mintlify dev --port 3001` | Docusaurus `npm start` |
| URL | `http://localhost:3001/en/use-dify/...` | `http://localhost:3000/use-spx-agent/...` |
| 모드 | 라이트 (다크는 `html.dark` 클래스 — 측정 시 `classList.remove('dark','twoslash-dark')`로 라이트 전환) | 라이트 |

> ⚠️ **모드 통일 필수**: 원본은 기본 다크라, 색상 비교 시 반드시 라이트로 맞춰 측정. px(폰트·여백·폭)는 모드 무관.

## 셀렉터 매핑 (재측정용)

| 요소 | Mintlify (3001) | Docusaurus (3000) |
|------|-----------------|-------------------|
| 본문 컨테이너 | `.mdx-content` (=`.prose`) | `.theme-doc-markdown`, `.markdown` |
| h1 | `#page-title` | `.markdown h1` |
| h2/h3 | `.mdx-content h2` / `h3` | `.markdown h2` / `h3` |
| 본문 문단 | `.mdx-content p` | `.markdown p` |
| inline code | `.mdx-content p code` | `.markdown p code` |
| 코드블록 | `.mdx-content pre` | `.markdown pre` |
| 콜아웃 | `.mdx-content [class*="callout"]` | `.theme-admonition`, `[class*="mintlify-callout"]` |
| 사이드바 | `#sidebar`, `aside`, `nav#navigation-items` | `.theme-doc-sidebar-container` |
| TOC | (셀렉터 재확인 필요) | `.table-of-contents` |

## 원본 Mintlify 기준값 (★ = custom.css 정답)

> 형식: `font-size / line-height / font-weight / color / 기타`

| 요소 | 원본 기준값 ★ |
|------|--------------|
| **본문 컨테이너** | max-width **576px** (고정) · margin 32px(top)/56px(bottom) · 본문 글자색 **#3E4146** · 16px/28px/400 |
| **h1** | 30px / **36px** / 600 / **#16191E** · margin 0/0 |
| **h2** | 24px / **32px** / 600 / **#111827** · margin-top 48 / margin-bottom 16 |
| **h3** | 20px / **28px** / 600 / **#111827** · margin-top 48 / margin-bottom 12 |
| **본문 문단(p)** | 16px / 28px / 400 / #3E4146 · margin-bottom 20 |
| **콜아웃** | **16px** / **28px** / 400 / #3E4146 · 배경 **#FAFAFA** · border 0.73px **#E5E5E5** · radius **16px** · padding 16/20 · margin 16/16 |
| **inline code** | 14px / 21px / **500** / #111827 · 배경 **rgba(238,240,245,0.5)** (반투명) · radius 6px · padding-left 8px · **border 없음** |
| **코드블록(pre)** | 14px / 24px / 400 / #1F2328 · (radius는 pre 자체 0 — 실제 둥근 모서리는 부모 wrapper에 있음, 재확인 필요) |
| **사이드바** | width **304px** (19rem) · 메뉴 글자색 **#6F7277** |

> 색상 hex↔rgb: #16191E=22,25,30 / #111827=17,24,39 / #3E4146=62,65,70 / #6F7277=111,114,119 / #FAFAFA=250,250,250 / #E5E5E5=229,229,229 / #EEF0F5=238,240,245

## 현재 Docusaurus와의 diff + custom.css 수정 체크리스트

> 우선순위순. [x] = 적용 완료. **2026-06-08 1차 수정 + 재측정 검증 완료** (custom.css)

- [x] **🔴 본문 max-width 576px 고정** — `article { max-width: 576px }` + 2xl(≥1536px)에서만 672px 미디어쿼리. 재측정 576 ✅
- [x] **콜아웃 폰트 크기** — `.mintlify-callout` font-size 1rem / line-height 1.75. 재측정 16/28 ✅ (단 **글자색 #262626 → #3E4146은 미적용**, 아래 잔여 참조)
- [x] **제목 line-height** — h1 1.2(36) / h2 1.333(32) / h3 1.4(28). 재측정 일치 ✅
- [x] **제목 margin** — h2 margin-bottom 1rem(16) / h3 margin-top 3rem(48)·margin-bottom 0.75rem(12). 재측정 일치 ✅
- [x] **h1 margin-top** — 0. 재측정 0 ✅
- [x] **inline code** — `code` border 제거, font-weight 500, 배경 rgba(238,240,245,0.5), padding-left 8px. 재측정 일치 ✅
- [x] **콜아웃 글자색** — info callout #262626 → **#3E4146** 적용·검증 ✅ (2차)
- [x] **사이드바 메뉴색** — `.menu__link` → **#6F7277 (gray-500)** 적용·검증 ✅ (2차)
- [x] **코드블록 내부 배경** — 라이트 **#fff** / 다크 **#0B0C0F**, 외곽 프레임(#F2F5FA·radius16 / 다크 투명) 안에 중첩(radius14). 적용·검증 ✅ (2차)
- [ ] **제목 색** — h2·h3 #16191E vs 목표 #111827 (**보류 확정** — 육안차 미미 + `--mint-gray-900` 팔레트 일관성)
- [ ] **사이드바 폭** — 300 vs 304px (`--doc-sidebar-width:304` 설정했으나 미반영, 원인 불명. 4px → 보류)

## 이미 일치하는 항목 (건드리지 말 것)

- 콜아웃 배경 #FAFAFA · 테두리 #E5E5E5 · radius 16px ✓
- 본문 컨테이너 margin 32/56 ✓
- h1·h2·h3 font-size (30/24/20) · font-weight 600 ✓
- inline code radius 6px ✓

## 코드블록 구조 (2차에 확정)

Mintlify 코드블록은 **2겹 중첩**:

| 레벨 | Mintlify 클래스 | 라이트 | 다크 |
|------|----------------|--------|------|
| 외곽 프레임 | `.code-block` | bg #F2F5FA · radius 16 · border 0.73px rgba(10,13,17,.1) | bg rgba(255,255,255,.05) · border rgba(255,255,255,.1) |
| 내부 코드 영역 | `.w-0…py-3.5 px-4` | bg **#fff** · radius 14 | bg **#0B0C0E** |

→ Docusaurus 매핑: 외곽 = `codeBlockContainer`, 내부 = `codeBlockContent`. 둘 다 적용 완료.

## 다크모드 기준값 (3차 확정)

> 라이트 ↔ 다크 색상 대조. custom.css `[data-theme='dark']` 규칙에 반영 완료.

| 요소 | 라이트 | 다크 |
|------|--------|------|
| 본문 p | #3E4146 (gray-700) | **#DEE1E6 (gray-200)** |
| 제목 h1~h3 | #16191E | **#FFFFFF (순백)** |
| 콜아웃 info 글자 | #3E4146 | **#9EA1A6 (gray-400)** |
| 콜아웃 배경 | #FAFAFA | rgba(255,255,255,0.1) |
| 콜아웃 테두리 | #E5E5E5 | #404040 |
| 코드 외곽 프레임 | #F2F5FA | rgba(255,255,255,0.05) |
| 코드 내부 영역 | #FFFFFF | **#0B0C0F** |
| inline code 배경 | rgba(238,240,245,0.5) | **rgba(255,255,255,0.05)** |
| 사이드바 메뉴 글자 | #6F7277 (gray-500) | #9EA1A6 (gray-400) |
| 페이지 배경 | #FFFFFF | #1B1B1D (원본 측정 불가 — 아래 참조) |

## 미검증 / 표본 부족 (다음 측정 시 보강)

- **링크 색** — 양쪽 다 본문 인라인 링크 대신 제목 앵커(`.mdx-content a` 첫 매치)가 잡힘. 색·underline 비교하려면 본문 단락 내 링크 표본 필요 (현재 Docusaurus는 #0060FF + underline로 의도 설정됨)
- **TOC non-active 색** — 원본은 첫 항목이 active(파랑)라 비활성 색 미확보. active 색은 양쪽 #0060FF로 일치. 폭은 원본 264 vs 현재 224
- **사이드바 메뉴 font-size** — 원본 16px로 측정됐으나 표본(width 62px) 의심. 현재 13.6px. 실제 메뉴 항목으로 재측정 필요
- **본문 문단 표본 이슈** — quick-start 첫 문단이 굵은 lead(600)라 weight가 600으로 잡힘. 일반 문단은 400/#3E4146
- ~~**다크모드 텍스트 색**~~ → **3차 해결** ✅: 본문 #DEE1E6(gray-200) / 제목 #FFFFFF / 콜아웃 #9EA1A6(gray-400) / inline code bg 0.05·색 gray-200. 정밀 측정 후 적용·검증 (위 다크모드 기준값 표 참조)
- **다크모드 페이지 배경색** — Mintlify는 `background-color`를 안 씀 (`color-scheme: dark` 캔버스 또는 background-image 추정). html·body·elementFromPoint 모두 투명으로 잡혀 **RGB 목표값 확보 불가** → 현재 Docusaurus #1B1B1D 유지로 보류
- **사이드바 메뉴 font-size** — 라이트·다크 모두 원본 16px로 측정되나 표본 신뢰도 낮음(짧은 항목 width 62px). 현재 13.6px. 실제 메뉴 항목 다건 평균으로 재측정 필요

## 갱신 이력

- 2026-06-08 (1차): 최초 작성 + custom.css 1차 수정. 본문폭 576 / 제목 line-height·margin / 콜아웃 폰트 16·28 / inline code 적용·검증
- 2026-06-08 (2차): 콜아웃 글자색 #3E4146 · 사이드바 메뉴색 #6F7277 · 코드블록 2겹 배경(라이트 #fff / 다크 #0B0C0F) 적용·검증. 다크모드 1차 스캔
- 2026-06-08 (3차): **다크모드 정밀 측정·수정** — 본문 gray-200 / 제목 순백 / 콜아웃 gray-400 / inline code bg 0.05. 적용·검증 ✅. 다크 배경색은 원본 측정 불가로 보류
