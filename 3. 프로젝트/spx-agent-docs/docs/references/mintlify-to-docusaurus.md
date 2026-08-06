# Mintlify → Docusaurus 전환 감사 보고서

> 조사 일자: 2026-06-01
> 범위: `en/use-dify/` (106 MDX 파일)
> 목적: 사내 Gitolite → Mintlify Cloud 연동 불가 확정에 따라, Docusaurus 자체 호스팅 전환 시 작업량·리스크 산정

---

## 1. Mintlify 전용 MDX 컴포넌트 인벤토리

### 사용 현황 (`en/use-dify/` 범위)

| 컴포넌트                               | 사용 횟수    | 파일 수 | Docusaurus 대응                      | 변환 방식      |
| ---------------------------------- | -------- | ---- | ---------------------------------- | ---------- |
| `<Frame>`                          | 191      | 32   | 커스텀 래퍼 (이미지 캡션/확대)                 | 자동 (래퍼 등록) |
| `<Info>`                           | 127      | 41   | `:::info` 또는 JSX 래퍼                | 자동 (스크립트)  |
| `<Steps>` / `<Step>`               | 35 / 87  | 20   | 커스텀 컴포넌트 또는 `<ol>` + CSS           | 자동 (래퍼 등록) |
| `<Tip>`                            | 60       | 26   | `:::tip` 또는 JSX 래퍼                 | 자동 (스크립트)  |
| `<Tabs>` / `<Tab>`                 | 32 / 54  | 24   | `<Tabs>` / `<TabItem>` (import 필요) | 자동 (래퍼 등록) |
| `<Note>`                           | 36       | 24   | `:::note` 또는 JSX 래퍼                | 자동 (스크립트)  |
| `<Warning>`                        | 30       | 23   | `:::warning` 또는 JSX 래퍼             | 자동 (스크립트)  |
| `<Card>` / `<CardGroup>`           | 54 / 12  | 12   | 커스텀 컴포넌트                           | 자동 (래퍼 등록) |
| `<Accordion>` / `<AccordionGroup>` | 33 / 7   | 11   | `<details>` 또는 커스텀                 | 자동 (래퍼 등록) |
| `<Check>`                          | 7        | 5    | `:::tip` 또는 커스텀 admonition         | 자동 (스크립트)  |
| `<CodeGroup>`                      | 2        | 2    | `<Tabs>` + 코드블록 조합                 | 수동 (2건)    |
| `<Callout>`                        | 2        | 2    | `:::note` 계열                       | 자동 (스크립트)  |
| **합계**                             | **~769** | —    | —                                  | —          |

### 미사용 컴포넌트 (변환 불필요)

`<ParamField>`, `<ResponseField>`, `<Mermaid>`, `<Snippet>`, `<Icon>`, `<Expandable>` — `en/use-dify/` 범위에서 사용하지 않음.

---

## 2. 프론트매터 차이

| Mintlify 키 | 사용 수 | Docusaurus 대응 키 | 변환 |
|-------------|--------|-------------------|------|
| `title` | 106 | `title` | 동일 (변환 불필요) |
| `icon` | 52 | *(해당 없음)* | 제거 또는 무시 (빌드 에러 없음) |
| `description` | 42 | `description` | 동일 |
| `sidebarTitle` | 27 | `sidebar_label` | 키 이름 치환 (sed) |
| `tag` | 3 | `tags: [...]` | 배열 형태 변환 |
| `mode` | 1 | *(해당 없음)* | 커스텀 CSS로 대체 또는 제거 |

**변환 스크립트 난이도**: 낮음 — 정규식 일괄 치환으로 처리 가능.

```bash
# 예시: sidebarTitle → sidebar_label
sed -i 's/^sidebarTitle:/sidebar_label:/' en/use-dify/**/*.mdx
```

---

## 3. 네비게이션 설정 변환

### Mintlify `docs.json` 구조

```
docs.json
├── tabs: [{tab: "문서", groups: [...]}]
│   └── groups: [{group: "지식", icon: "book", pages: [...]}]
│       └── pages: ["ko/use-spx-agent/knowledge/readme"]
├── navbar: {primary: {label, href}}
├── logo: {light, dark}
├── colors: {primary, light, dark}
└── versions: [...]
```

### Docusaurus 대응

| Mintlify | Docusaurus | 파일 |
|----------|-----------|------|
| `tabs` | 사이드바 named export 또는 navbar docs dropdown | `sidebars.js` |
| `groups` | `category: { label, items }` | `sidebars.js` |
| `pages` | doc ID (파일 경로, 확장자 제외) | `sidebars.js` |
| `navbar.primary` | `themeConfig.navbar.items` | `docusaurus.config.js` |
| `logo` | `themeConfig.navbar.logo` (srcDark 지원) | `docusaurus.config.js` |
| `colors` | CSS 커스텀 속성 | `custom.css` |
| `versions` | `versions.json` + `versioned_docs/` | Docusaurus CLI |
| i18n (ko) | `i18n: { defaultLocale, locales }` | `docusaurus.config.js` |
| `icon` (그룹) | 기본 미지원 → 커스텀 CSS/컴포넌트 | swizzle 필요 |

**변환 난이도**: 중간 — `docs.json` → `sidebars.js` 변환 스크립트 작성 가능 (JSON → JS 객체 매핑). 그룹 아이콘은 커스텀 작업 필요.

---

## 4. 링크 형식 전수 조사

### 내부 링크 (62건)

- **형식**: 절대 경로, 확장자 없음 — 예: `/en/use-dify/nodes/user-input`
- **Docusaurus 호환성**: Docusaurus도 확장자 없는 경로를 사용하므로 **기본 호환**
- **변환 필요 사항**: 경로 prefix `/en/use-dify/` → `/ko/use-spx-agent/` (ko 문서 기준)
- **상대 경로 사용**: 0건 (모두 절대 경로)

### 외부 링크 (286건)

| 도메인 | 건수 | 비고 |
|--------|------|------|
| assets-docs.dify.ai | 182 | 이미지 CDN (아래 이미지 섹션 참조) |
| github.com | 14 | 플러그인 리포 |
| dify.ai | 14 | 블로그, 가격 |
| i.ibb.co | 14 | 이미지 호스팅 |
| marketplace.dify.ai | 7 | spx-agent에서 삭제 대상 |
| 기타 | ~55 | LangSmith, Langfuse, MCP 등 |

**변환 스크립트 가능성**: 높음 — 내부 링크는 prefix 치환으로 일괄 처리 가능.

```bash
# 내부 링크 prefix 변환
sed -i 's|(/en/use-dify/|(/ko/use-spx-agent/|g' ko/use-spx-agent/**/*.mdx
```

---

## 5. MDX 빌드 호환성

### 버전 차이

| | Mintlify | Docusaurus v3 |
|--|---------|--------------|
| MDX 버전 | v2 (`next-mdx-remote` v4.4.1 기반) | **v3** |
| 파싱 엄격도 | 느슨 | 엄격 (`{`, `<` escape 필요) |
| 컴포넌트 임포트 | 자동 (빌트인) | 명시적 import 또는 `MDXComponents` 글로벌 등록 |

### 주요 깨지는 부분

1. **`{` 문자**: JS 표현식으로 엄격 해석 → 설정값 `{example}` 등에서 컴파일 에러
2. **`<` 문자**: JSX 태그 시작으로 엄격 파싱 → 비교 연산/HTML 엔티티 에러
3. **컴포넌트 자동 임포트 제거**: 769회 사용된 Mintlify 빌트인 컴포넌트가 모두 `undefined`

### 대응 전략

- **`docusaurus-mdx-checker`**: 마이그레이션 전 전체 파일에 대해 v3 호환성 검사 실행
- **`MDXComponents` 글로벌 등록**: Mintlify 컴포넌트명을 유지하면서 Docusaurus 네이티브 컴포넌트로 래핑 → MDX 본문 수정 최소화

---

## 6. 이미지/에셋 경로 패턴

### 이미지 참조 총 384건

| 유형 | 건수 | 비율 | 예시 |
|------|------|------|------|
| 로컬 절대 경로 (`/images/...`) | 181 | 47% | `/images/deeper_dive_workflow_overview.png` |
| 원격 CDN (`assets-docs.dify.ai`) | 182 | 47% | `https://assets-docs.dify.ai/2025/06/...png` |
| 외부 호스팅 (`i.ibb.co` 등) | 21 | 6% | `https://i.ibb.co/...png` |

### Docusaurus 변환

| Mintlify 경로 | Docusaurus 경로 | 변환 방식 |
|--------------|----------------|----------|
| `/images/foo.png` | `/img/foo.png` (static 폴더) 또는 상대 경로 | sed 일괄 치환 |
| `https://assets-docs.dify.ai/...` | 로컬 다운로드 → `/img/` 또는 그대로 유지 | 결정 필요 |

**권장**: 로컬 경로는 `/images/` → Docusaurus `static/images/`로 복사 후 경로 유지. 원격 CDN은 spx-agent 화면 교체 대상이므로 Phase 5에서 일괄 처리.

---

## 7. 검색 인프라 후보

> 전제: 사내망(Gitolite) 배포 → Algolia DocSearch 사용 불가

| 플러그인 | 주간 DL | Stars | 한국어 | 특징 |
|---------|---------|-------|--------|------|
| **`@easyops-cn/docusaurus-search-local`** | ~50k | 792 | **지원** (v0.25+) | TypeScript, lunr 기반, 완전 오프라인, i18n 연동 |
| `@cmfcmf/docusaurus-search-local` | ~21k | 468 | 제한적 | 원본 포크, 기본적 |
| `docusaurus-lunr-search` | — | — | 제한적 | Lunr 기반, V3 지원 |

**권장: `@easyops-cn/docusaurus-search-local`** — 한국어 인덱싱 지원, 인트라넷 완전 오프라인 동작, 가장 활발한 유지보수.

---

## 8. API 레퍼런스 자동생성

> 현재 scope에서 API Reference는 **제외** (`.claude/CLAUDE.md` scope-mapping 참조).

참고용 비교:

| | Mintlify | Docusaurus |
|--|---------|-----------|
| 도구 | 내장 OpenAPI 렌더러 | `docusaurus-plugin-openapi-docs` (PaloAltoNetworks) |
| 스펙 | OpenAPI 3.0+ | Swagger 2.0 + OpenAPI 3.x |
| Try-it | 자동 | 테마에서 "Try it" 패널 제공 |
| 생성 방식 | 빌드 시 자동 | CLI `docusaurus gen-api-docs` 수동 실행 |

**결론**: 현재 필요 없음. 향후 필요 시 `docusaurus-plugin-openapi-docs`가 가장 성숙한 옵션.

---

## 9. 컴포넌트별 자동 변환 vs 수동 재작성 분류

### 전략: MDXComponents 글로벌 래퍼 (2-pass 접근)

**1단계 (초기 마이그레이션)**: `MDXComponents`에 Mintlify 호환 래퍼를 등록하여 MDX 본문 수정 최소화
**2단계 (점진 정리)**: 네이티브 Docusaurus 구문(`:::` admonitions 등)으로 선택적 전환

### 분류표

| 컴포넌트 | 분류 | 방식 | 상세 |
|----------|------|------|------|
| **콜아웃 계열** (`<Info>`, `<Tip>`, `<Note>`, `<Warning>`, `<Check>`, `<Callout>`) | **자동** (래퍼) | `MDXComponents`에 `@theme/Admonition` 래퍼 등록 | 262건 — 본문 수정 0, 래퍼 컴포넌트 6개 작성 |
| `<Frame>` | **자동** (래퍼) | CSS 기반 이미지 래퍼 컴포넌트 등록 | 191건 — `<figure>` + 캡션/확대 기능 |
| `<Steps>` / `<Step>` | **자동** (래퍼) | 커스텀 컴포넌트 (`<ol>` + CSS counter) | 122건 — 래퍼 2개 작성 |
| `<Tabs>` / `<Tab>` | **자동** (래퍼) | `@theme/Tabs` + `@theme/TabItem` 래핑 | 86건 — Tab → TabItem 래퍼 |
| `<Card>` / `<CardGroup>` | **자동** (래퍼) | CSS Grid 기반 카드 컴포넌트 | 66건 — 래퍼 2개 작성 |
| `<Accordion>` / `<AccordionGroup>` | **자동** (래퍼) | `<details>` / `<summary>` 래핑 | 40건 — 래퍼 2개 작성 |
| `<CodeGroup>` | **수동** | `<Tabs>` + `<TabItem>` + 코드블록 조합으로 재작성 | **2건** — 수동 작업량 미미 |

### 래퍼 컴포넌트 작성 목록 (1단계)

```
src/theme/MDXComponents/
├── Admonitions.tsx    # Info, Tip, Note, Warning, Check, Callout → Admonition 래퍼
├── Frame.tsx          # <Frame> → <figure> + CSS
├── Steps.tsx          # <Steps>, <Step> → <ol> + CSS counter
├── TabsWrapper.tsx    # <Tabs>, <Tab> → Tabs/TabItem 래핑
├── Card.tsx           # <Card>, <CardGroup> → CSS Grid 카드
└── Accordion.tsx      # <Accordion>, <AccordionGroup> → <details>/<summary>
```

**래퍼 컴포넌트 총 작업량**: ~6개 파일, 각 50~100줄 → **약 0.5일**

---

## 10. 빈도 상위 컴포넌트 Before/After 샘플

### 샘플 1: `<Info>` (127회, 최다 콜아웃)

**Before (Mintlify)**:
```mdx
<Info>
모든 앱 유형은 **Chatflow** 또는 **Workflow** 두 가지 오케스트레이션 모드를 지원합니다.
</Info>
```

**After 옵션 A — MDXComponents 래퍼 (본문 수정 0)**:
```mdx
<!-- MDX 본문 그대로 유지, MDXComponents에서 래핑 -->
<Info>
모든 앱 유형은 **Chatflow** 또는 **Workflow** 두 가지 오케스트레이션 모드를 지원합니다.
</Info>
```

```tsx
// src/theme/MDXComponents/Admonitions.tsx
import Admonition from '@theme/Admonition';

export const Info = ({ children, title }) => (
  <Admonition type="info" title={title}>{children}</Admonition>
);
```

**After 옵션 B — 네이티브 구문 (2단계 점진 전환)**:
```mdx
:::info
모든 앱 유형은 **Chatflow** 또는 **Workflow** 두 가지 오케스트레이션 모드를 지원합니다.
:::
```

### 샘플 2: `<Frame>` (191회, 최다 컴포넌트)

**Before (Mintlify)**:
```mdx
<Frame>
  <img src="/images/workflow-overview.png" alt="워크플로 개요" />
</Frame>
```

**After — MDXComponents 래퍼 (본문 수정 0)**:
```mdx
<!-- MDX 본문 그대로 유지 -->
<Frame>
  <img src="/images/workflow-overview.png" alt="워크플로 개요" />
</Frame>
```

```tsx
// src/theme/MDXComponents/Frame.tsx
export const Frame = ({ children, caption }) => (
  <figure className="mintlify-frame">
    {children}
    {caption && <figcaption>{caption}</figcaption>}
  </figure>
);
```

```css
/* custom.css */
.mintlify-frame {
  border: 1px solid var(--ifm-color-emphasis-300);
  border-radius: 8px;
  overflow: hidden;
  margin: 1rem 0;
}
.mintlify-frame img {
  display: block;
  width: 100%;
}
```

---

## 11. 작업량 산정

### 전체 요약

| 작업 항목 | 예상 시간 | 비고 |
|----------|----------|------|
| **Docusaurus 프로젝트 초기 셋업** | 0.5일 | `create-docusaurus`, config, i18n, 검색 플러그인 |
| **MDXComponents 래퍼 컴포넌트 6종** | 0.5일 | Admonitions, Frame, Steps, Tabs, Card, Accordion |
| **CSS 스타일링** (래퍼 + 레이아웃) | 0.5일 | Mintlify 디자인 유사도 맞추기 |
| **네비게이션 변환 스크립트** | 0.25일 | `docs.json` → `sidebars.js` 생성기 |
| **프론트매터 변환 스크립트** | 0.25일 | `sidebarTitle` → `sidebar_label`, icon 제거 등 |
| **내부 링크 변환 스크립트** | 0.25일 | prefix 치환 |
| **MDX v3 호환성 수정** | 0.5~1일 | `{`/`<` escape 처리 (docusaurus-mdx-checker 기반) |
| **CodeGroup 수동 전환** | 0.1일 | 2건 |
| **빌드 검증 + 디버깅** | 1일 | 전체 빌드, 링크 깨짐, 렌더링 확인 |
| **검색 설정 + 테스트** | 0.25일 | `@easyops-cn/docusaurus-search-local` |
| **버퍼 (예상 외 이슈)** | 0.5~1일 | MDX 파싱 에지 케이스 등 |
| **합계** | **4~5.5일** | |

### 범위별 시나리오

| 시나리오 | 대상 | 예상 시간 |
|---------|------|----------|
| **A. ko/ 만 전환** (현재 2페이지 + 향후 추가분) | 파일럿 결과물만 | **1~1.5일** |
| **B. ko/ + en/use-dify/ 전환** (106 페이지) | Use Dify 전체 | **4~5.5일** |
| **C. 전체 리포 전환** (en/ + ko/ + ja/ + zh/) | 모든 콘텐츠 | **7~10일** |

**권장: 시나리오 A로 시작** → Docusaurus 셸 + 래퍼 구축 후 ko/ 파일만 옮기고 검증. 이후 Phase 4 본격 포팅 시 en/use-dify/ 원본도 필요하면 시나리오 B로 확장.

---

## 12. 기존 마이그레이션 도구

**Mintlify → Docusaurus 방향의 기성 도구는 존재하지 않습니다.**

- Mintlify `@mintlify/scraping`은 역방향(타 플랫폼 → Mintlify)만 지원
- Docusaurus 공식 마이그레이션은 v1 → v2 전환용
- 실제 사례들은 모두 커스텀 스크립트를 작성

### 권장 커스텀 스크립트 구성

```
tools/migrate/
├── convert-frontmatter.js   # sidebarTitle→sidebar_label, icon 제거
├── convert-nav.js           # docs.json → sidebars.js
├── convert-links.js         # 내부 링크 prefix 치환
├── check-mdx-compat.js      # { / < escape 대상 식별
└── README.md                # 사용법
```

---

## 13. 리스크 및 권장사항

### 리스크

| 리스크 | 심각도 | 대응 |
|--------|--------|------|
| MDX v3 파싱 에러 대량 발생 | 중 | `docusaurus-mdx-checker` 사전 검사 |
| Mintlify 디자인 재현 불완전 | 낮 | 사내용이므로 기능 우선, 디자인 80% 일치로 충분 |
| 래퍼 컴포넌트 동작 차이 | 중 | 파일럿(ko/ 2페이지)에서 먼저 검증 |
| 검색 한국어 품질 | 낮 | `@easyops-cn` 플러그인 한국어 토크나이저 내장 |

### 핵심 권장사항

1. **MDXComponents 글로벌 래퍼 전략 채택** — MDX 본문 수정 최소화, 마이그레이션 속도 극대화
2. **시나리오 A (ko/ 먼저)** — 래퍼 검증 후 범위 확장
3. **2-pass 접근** — 1단계 래퍼로 빠른 전환, 2단계 점진적 네이티브 구문 전환
4. **검색은 `@easyops-cn/docusaurus-search-local`** — 사내망 완전 오프라인 + 한국어 지원

---

## 부록: 전체 Mintlify 컴포넌트 → Docusaurus 매핑 요약

| Mintlify | Docusaurus (1단계 래퍼) | Docusaurus (2단계 네이티브) |
|----------|----------------------|--------------------------|
| `<Info>` | `MDXComponents.Info` → Admonition | `:::info` |
| `<Tip>` | `MDXComponents.Tip` → Admonition | `:::tip` |
| `<Note>` | `MDXComponents.Note` → Admonition | `:::note` |
| `<Warning>` | `MDXComponents.Warning` → Admonition | `:::warning` |
| `<Check>` | `MDXComponents.Check` → Admonition | `:::tip` (커스텀) |
| `<Callout>` | `MDXComponents.Callout` → Admonition | `:::note` |
| `<Frame>` | `MDXComponents.Frame` → `<figure>` | `<figure>` + CSS |
| `<Steps>` | `MDXComponents.Steps` → `<ol>` + CSS | `<ol>` + CSS |
| `<Step>` | `MDXComponents.Step` → `<li>` + CSS | `<li>` + CSS |
| `<Tabs>` | `MDXComponents.Tabs` → `@theme/Tabs` | `import Tabs` |
| `<Tab>` | `MDXComponents.Tab` → `@theme/TabItem` | `import TabItem` |
| `<Card>` | `MDXComponents.Card` → 커스텀 | 커스텀 유지 |
| `<CardGroup>` | `MDXComponents.CardGroup` → CSS Grid | 커스텀 유지 |
| `<Accordion>` | `MDXComponents.Accordion` → `<details>` | `<details>` |
| `<AccordionGroup>` | `MDXComponents.AccordionGroup` → 래퍼 | 래퍼 유지 |
| `<CodeGroup>` | 수동 → `<Tabs>` + `<TabItem>` | `<Tabs>` + `<TabItem>` |

---

## 14. 스타일 픽셀 검증 절차 (Mintlify ↔ Docusaurus computed-style diff)

> 2026-06-08 박제. Docusaurus 테마가 Mintlify와 "눈에 띄게 다른데 원인을 못 잡는" 문제의 해결 절차.
> 한 번 익히면 **브라우저 DevTools 콘솔만으로 누구나 반복 가능** (AI 자동화 도구 불필요).

### 14.1 왜 기존 방식(curl + 클래스 px 추출 + 추정)이 실패하는가

| 실패한 방식 | 구조적 한계 |
|------------|-----------|
| `curl`로 HTML 받아 분석 | curl은 **정적 마크업**만 가져옴. JS 미실행 → SPA hydration 후 적용되는 레이아웃·스타일 누락 |
| Tailwind 클래스명에서 px 추출 | `p-4`·`gap-6` 등 유틸 클래스는 px를 안 담음 — 실제 값은 **테마 토큰(tailwind.config)**에 의존. Mintlify는 커스텀 토큰이라 기본 Tailwind와 다름 → 클래스 추출 = 추정 |
| WebFetch 실패 후 "이런 스타일일 것" 추정 | 디자인 시스템은 수십 개 토큰 조합 — 눈대중으로 못 맞춤 |
| 마크업만 보고 판단 | `::before`/`::after`, 부모 상속, 미디어쿼리, CSS 변수 실제값은 **클래스 리스트에 안 보임** → "다른 데가 너무 많다"의 정체 |

**결론**: 정적 마크업·클래스명·추정으로는 절대 못 맞춘다. **살아있는 DOM의 computed style을 실측**해야 한다.

### 14.2 원칙 — 실측 diff

마크업이 아니라 `getComputedStyle()`로 **실제 계산된 값**을 양쪽에서 뽑아 1:1 비교한다. 어긋난 지점이 숫자로 나오면, 그 값만 `custom.css`에 반영한다 (추정 0).

### 14.3 환경

| | 원본 (기준) | 대상 |
|--|-----------|------|
| 도구 | Mintlify (`mintlify dev --port 3001`) | Docusaurus (`npm start`) |
| URL | `http://localhost:3001` | `http://localhost:3000` |
| 프로젝트 경로 | `C:\Users\Administrator\Projects\dify-docs-en-1.13.3` | `C:\Users\Administrator\Projects\spx-agent-docs` |
| 수정 파일 | — | `src/css/custom.css`, `src/theme/**` |

> 두 서버가 **동시에 떠 있어야** 비교 가능. 같은 성격의 페이지를 양쪽에 띄운다 (예: quick-start).

### 14.4 절차

1. **양쪽에 같은(또는 가장 유사한) 페이지를 띄운다** — 3001 원본, 3000 대상
2. 각 페이지에서 **F12 → Console**에 아래 스니펫 실행 (§14.5). 결과 JSON이 클립보드에 자동 복사됨
3. 두 JSON을 **요소별로 diff** — 값이 다른 속성만 추린다 (예: `h1 font-size 30 vs 28`)
4. **어긋난 값만 `custom.css`에 반영** (또는 해당 `src/theme/*.jsx` 래퍼)
5. 3000 새로고침 후 같은 요소 재측정 → 일치할 때까지 반복
6. **요소 그룹 단위로 좁혀서** 진행: ① 타이포그래피(h1~h4·본문·링크) → ② 레이아웃(사이드바·본문폭·TOC·여백) → ③ 컴포넌트(콜아웃·Steps·Tabs·Frame·코드블록·pagination) → ④ 색상·다크모드

### 14.5 측정 스니펫 (양쪽 콘솔에 붙여넣기)

```js
(() => {
  // 셀렉터는 Mintlify/Docusaurus 마크업이 달라 '(없음)'이 뜰 수 있음 → 그 요소는 구조 확인 후 교체
  const sels = {
    h1:        'article h1, main h1, h1',
    h2:        'article h2, h2',
    h3:        'article h3, h3',
    bodyP:     'article p, .markdown p',
    link:      'article a, .markdown a',
    inlineCode:'code:not(pre code)',
    codeBlock: 'pre',
    callout:   '.admonition, [class*="callout"], [class*="admonition"]',
    sidebar:   'nav[class*="sidebar"], aside, .theme-doc-sidebar-container',
    toc:       '[class*="tableOfContents"], [class*="toc"]',
  };
  const props = ['font-family','font-size','line-height','font-weight',
    'color','background-color','margin-top','margin-bottom',
    'padding-top','padding-right','padding-bottom','padding-left',
    'width','max-width','gap','border-radius','border','box-shadow'];
  const out = {};
  for (const [k, s] of Object.entries(sels)) {
    const el = document.querySelector(s);
    if (!el) { out[k] = '(없음: '+s+')'; continue; }
    const cs = getComputedStyle(el);
    out[k] = Object.fromEntries(props.map(p => [p, cs.getPropertyValue(p)]));
  }
  const json = JSON.stringify(out, null, 2);
  console.log(json);
  try { copy(json); } catch (e) {}   // DevTools copy() — 클립보드 자동 복사
  return out;
})();
```

> ⚠️ **셀렉터 주의**: Mintlify와 Docusaurus는 컨테이너 구조가 달라 같은 "사이드바"도 셀렉터가 다르다. `(없음)`으로 뜨면 그 요소만 DevTools Elements에서 실제 클래스를 확인해 셀렉터를 교체한다. 측정 요소를 늘리려면 `sels`에 항목 추가.

### 14.6 핵심 측정 요소 체크리스트

- [ ] 타이포그래피: h1·h2·h3, 본문 p, 링크, inline code
- [ ] 코드블록: 배경색, padding, border-radius, 폰트(JetBrains Mono 여부)
- [ ] 콜아웃 5종: info·note·tip/check·warning 타입별 배경·border·아이콘
- [ ] 레이아웃: 사이드바 폭·padding, 본문 max-width(672px), 본문↔사이드바 간격(64px), TOC 폭(304px)
- [ ] 컴포넌트: Steps(28px 원·수직선), Tabs(border-bottom), Frame(도트 배경·radius), Pagination
- [ ] 색상: gray 팔레트(`--mint-gray-*`), 다크모드

### 14.7 (선택) AI 자동화

Claude in Chrome 익스텐션(claude.ai/chrome) 연결 시, AI가 두 서버를 직접 띄워 요소별 computed style을 추출·diff까지 수행 가능. 익스텐션 미연결 시에는 위 §14.5 스니펫을 직접 실행하고 결과 JSON을 전달하면 동일하게 diff 분석을 받을 수 있다.
