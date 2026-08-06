# spx-agent 문서 이미지 교체/제거 작업 설계

> 2026-06-17 작성. ko 문서의 Dify 원본 이미지를 spx-agent 화면으로 교체하거나 제거하기 위한 작업 설계.
> 판단 근거: 세션 논의 + 리포 정찰(이미지 참조 형태·`images/` 구조 확인).
> 운용 방침 확정: [[decisions#결정 15]] · [[decisions#결정 16]].
> 관련: [[decisions]] · [[progress]] · [[conventions]] · [[spx-edition-feature-gating]]

## 0. 운용 방침 (2026-06-17 확정 — [[decisions#결정 16]])

| 항목 | 결정 |
|------|------|
| 스트림 순서 | **A(기존 이미지 교체/제거) 먼저**, B(이미지 0개인 spx 신규 챕터 캡처)는 분리·후속 |
| 애매 처리 | 빈 `review`로 미루지 않고 **교체/제거 권장값 추정 + `suggested` 플래그** → 사람은 확인/뒤집기만 |
| 진행 단위 | **파일럿 1~2챕터 먼저** → 검증 후 전체 확대 |
| 캡처 방식 | **수동 캡처 + 에이전트 후처리**. Playwright 자동 캡처 도입 안 함 |

> 신규 챕터(대시보드·권한·부서)는 이미지 0개 확인 → 인벤토리(기존 참조 추출)로는 안 잡히므로 **B 스트림에서 분석 문서 기반 별도 캡처**.

---

## 1. 목표

- ko 문서에 박혀 있는 **Dify 원본 이미지(전부 영문 UI)** 를 spx-agent 실제 화면으로 **교체**하거나, 불필요한 경우 **제거**한다.
- 전수 교체(549장 규모)는 비현실적이므로, **본문이 실제로 이미지를 필요로 하는 자리에만** 캡처를 투입해 최소 노동으로 최대 가치를 확보한다.
- 사람의 노동을 "캡처"와 "소수 검토"로 한정하고, 식별·배치·삽입·검증은 에이전트가 처리하도록 작업을 구조화한다.

---

## 2. 작업 배경

### 2.1 현황 (수치)

| 구분 | 문서 | 렌더링 이미지 | 이미지 포함 문서 | 비고 |
|------|------|--------------|-----------------|------|
| ko | 88 | 313 | 54 | `<video>` 2개 별도 |
| en | 157 | 549 | 96 | 원문(소스) |

- ko 81개 문서가 en 원문과 대응. 이 중 **14개 파일에서 이미지 82개가 빠진 상태** (기준 없이 작업하다 누락).
- 이미지 참조 형태 3종 혼재: `![alt](/images/x.png)`, `<img src="/images/x.png">`, 대부분 `<Frame>`로 감쌈.
- 저장 위치: 평면 `images/` 폴더 한 곳.

### 2.2 작업 이유

- 현재 이미지는 **전부 Dify 원본**(영문 UI, 일부 Dify 로고 노출). 한국어 spx-agent 화면이 아니라 신뢰도·정합성을 해친다.
- spx-agent 전용 신규 챕터(대시보드/감사로그/권한/부서관리)는 애초에 대응 원문이 없어 **신규 캡처가 필수**.

### 2.3 전제 (세션에서 확정된 판단)

> 아래 전제들이 작업 흐름의 단순화를 이끌었다. 변경 시 본 설계 재검토 필요.

1. **모든 기존 이미지는 Dify 원본이고 사용자가 의도적으로 삽입한 것이 아니다.**
   → 파일명(해시/CleanShot/의미명)은 Dify의 작명 습관일 뿐 "보존 의도" 신호가 아니다. **파일명 기반 분류 폐기.**
2. **모든 이미지의 언어는 영어다.**
   → "언어무관이라 유지"하는 예외가 사실상 없다. 로고 유무도 어차피 교체/제거 둘 중 하나로 가면 판정에 영향 없음.
3. **본문 텍스트는 이미 spx-agent 기준으로 작성되어 있다.**
   → spx에 없는 기능은 본문에도 없다. 따라서 "기능 보유 여부"를 코드와 대조할 필요 없음. 기능 게이트는 이미 텍스트에 반영 완료.
4. 누락된 82개는 "원래 있어야 했던 것"이 아니라 그냥 빠진 것 → **자동 0순위가 아니다.** 새 기준으로 다시 판별한다.

### 2.4 핵심 결론

위 전제로 판단 축이 **단일 이분(二分)으로 붕괴**한다:

> **이 자리에 이미지가 필요한가?** → 필요하면 **교체**(spx 화면 재캡처), 불필요하면 **제거**.

그리고 이 판별은 **본문 텍스트의 속성**이므로 **텍스트 스캔이 비전 스캔보다 효율적·정확**하다. (비전은 "기존 이미지에 무엇이 들었나"를 답할 뿐, "본문이 이미지를 필요로 하나"는 답하지 못함.) 비전은 폐기하지 않고 **제거 후보의 최종 가드**로만 좁게 사용한다(§4.3).

---

## 3. 판별 기준

판별 축은 하나 — **본문이 그 이미지를 필요로 하는가.**

### 3.1 교체 (재캡처)

본문이 *구체적 UI를 찾거나 조작*하게 만들 때. 이미지가 없으면 독자가 헤맨다.

- "○○ 버튼 클릭", "○○ 패널에서", "아래처럼 설정" 등 **행동 지시**
- 단계별 **절차** (설정 화면, 노드 패널, 폼 입력)
- 화면 **레이아웃/위치** 인지가 필요한 설명 ("좌측 상단에 표시됨")
- 실행 결과·그래프 등 **눈으로 확인**시켜야 하는 화면

### 3.2 제거 (이미지 참조 + 고아 파일만 삭제, 본문 유지)

이미지가 장식·중복일 때.

- 순수 **개념 설명**이라 글만으로 자족
- 인접 이미지와 **중복**되거나 같은 화면 반복
- 단순 아이콘·로고·표지성 이미지
- 본문이 UI를 언급하지 않는데 끼워넣은 분위기 컷

> ⚠️ 본문은 이미 spx 기준이므로 "제거"는 원칙적으로 **이미지만** 빼는 것이다. 텍스트째 삭제하지 않는다.

### 3.3 애매한 경우 — 권장 표시 (`suggested`)

경계가 애매한 행(개념 설명이지만 다이어그램이 도움될 수 있는 경우 등)도 **빈 `review`로 미루지 않는다.** 교체/제거 중 더 그럴듯한 쪽을 **권장값으로 추정**하고 `suggested: true` 플래그를 단다. 사람은 권장값을 **확인하거나 뒤집기만** 하면 된다(처음부터 판단하지 않음).

> 예외: §4.3 제거 가드(명시적 참조·alt)에 걸리는 제거 후보는 권장값을 `replace`로 안전하게 틀거나, 그래도 애매하면 비전 스팟체크 대상으로 표시한다.

---

### 3.4 🔴 제거 우선 정책 (2026-06-18 개정 — 실제 렌더 화면 피드백)

> 실제 노드/페이지 화면을 본 뒤 방침 **대전환**. 이전(§3.1~3.3)의 "절차형은 교체" 기조를 덮어쓴다. **기본값 = 제거.** A·B 스트림 공통. 교체/추가는 "텍스트로 표현 불가능한" 극소수만.

**무조건 제거 — 이미지 + 해당 텍스트까지 (`remove_scope: "section"`):**
- **예시/데모 워크플로우 섹션** — 헤딩에 "예시"가 있는 섹션(예: `예시 워크플로우`, `분류 워크플로우 예시`, `조건 처리 예시`, `기본/고급 루프 예시`, `긴 글 생성 예시`)은 **섹션 전체**(헤딩+본문+이미지) 제거. **노드·비노드 공통**.

**이미지만 제거 — 텍스트 유지 (`remove_scope: "image"`):**
- **영상(`<video>`)·gif** — 전부 제거.
- **캡처 불가 화면** — 오류 화면 등 spx 재현 불가 → 제거.
- 주변 텍스트(설정값·단계·설명)만으로 정보가 충분한 일반 스크린샷 → 이미지만 제거.

**유지/교체(재캡처) — 텍스트로 표현 불가능한 극소수만:**
- 복잡한 레이아웃·찾기 어려운 UI 위치·핵심 설정 토글 등 글로 설명이 안 되는 화면. (예: iteration `병렬 모드 옵션` 1장)
- `action: "replace"`/`"new"`는 reason에 **"텍스트로 왜 표현 불가한지"** 명시 필수.

**매니페스트 필드 (개정):**
- `action`: `"remove"`(기본) | `"replace"` | `"new"`
- `remove_scope`: `"image"`(이미지만) | `"section"`(헤딩+텍스트+이미지). remove일 때 필수.
- section 제거 시 `section_anchor`(제거할 섹션 헤딩) 기재. 같은 섹션의 여러 이미지는 동일 section_anchor로 묶어 한 번에 제거.

**B 스트림도 동일** — 데이터 시각화라도 ①의미②왜③활용 텍스트로 충분하면 이미지 생략. 전체 화면/핵심 차트 등 **텍스트로 못 보여주는 것만** 남긴다.

**판정 우선순위**: 예시 섹션? → section 제거. 영상/gif/캡처불가? → image 제거. 텍스트로 충분? → image 제거. 텍스트로 불가능? → 유지/교체. (애매하면 제거 쪽, `suggested:true`)

## 4. 작업 흐름

```
[1] 인벤토리 (결정적 스크립트, 공짜)
    ko MDX 전수 → 이미지 참조 추출
    = {문서경로, 앵커(섹션), 기존 파일명, alt, 주변 본문 N줄, 명시적참조여부}
    → image-manifest 초안(action 미정)

[2] 텍스트 스캔 등급화 (Claude Code 서브에이전트 병렬)
    각 참조의 "주변 본문"을 읽어 §3 기준 적용
    → action: "replace" | "remove"  (항상 권장값을 채움, 빈 review 없음)
       suggested: true/false        (애매하면 true = 사람 확인 필요)
       replace : 찍을 화면 설명 + 제안 파일명(spx-<챕터>-<주제>-NN) + 한국어 alt
       remove  : 사유

[3] 사람 검토 (suggested:true 행 + replace 목록 훑기)
    권장값 확인/뒤집기 → 캡처 대상 확정

[4a] 제거 처리 (에이전트 일괄)
     §4.3 가드 통과 확인 → 이미지 참조 삭제 → 고아 PNG 파일 삭제

[4b] 교체 처리
     사람   : 매니페스트 따라 spx 화면 캡처 (replace 행만)
     에이전트: 리네임 → static/images/ 배치(루트 images/ 사본 안 함) → 참조 교체 → 한국어 alt → 빌드 검증

[5] 마감
     npm run build 깨짐 검증 / 고아 자산 정리 / ja·zh 동기화(또는 보류)
```

### 4.1 왜 텍스트 스캔인가 (비전 스캔 대비)

| | 비전 스캔 | 텍스트 스캔 |
|---|---|---|
| 답하는 질문 | "기존 이미지에 **무엇이 들었나**" | "이 본문이 **이미지를 요구하나**" |
| 필요성과의 관계 | 어긋남 (필요성은 본문의 속성) | 일치 |
| 비용 | 313장 무겁고 느림 | 가볍고 병렬 빠름 |
| 정확도 | 엉뚱한 대상(기존 이미지)을 봄 | 근거(본문)를 직접 봄 |

→ **교체/제거 판별엔 텍스트 스캔이 주력.** 313장 전수 열람 단계가 사라진다.

### 4.2 매니페스트 한 행 예시

```json
{
  "doc": "ko/use-spx-agent/analytics-audit/dashboard/readme.mdx",
  "anchor": "## KPI 카드",
  "old_file": "/images/853427c8...png",
  "alt": "부서별 KPI 카드",
  "context": "상단 KPI 4개 카드는 ... 클릭하면 드릴다운 ...",
  "explicit_ref": false,
  "action": "replace",
  "suggested": false,
  "capture": "관리자 대시보드 상단 KPI 4카드",
  "new_file": "spx-dashboard-kpi-01.png",
  "reason": "절차/화면 위치 인지 필요"
}
```

### 4.3 제거(remove)의 비전 가드 — 비가역 결정이므로

제거는 영구 삭제라, 기존 이미지가 *텍스트에 없는 정보*(다이어그램·비교표·주석 스크린샷)를 담고 있으면 텍스트 스캔이 이를 놓칠 수 있다. 다음 순서로 막는다.

1. **명시적 참조 스캔** — 이미지 주변에 "아래/다음/그림/표/처럼" 패턴이 있으면 본문 의존 → **자동 제거 금지**, `review`로.
2. **alt 텍스트를 내용 프록시로 사용** — alt가 "구조도/비교/아키텍처/흐름" 류면 자동 제거 금지, `review`로.
3. 위 1·2를 통과하고도 애매한 **잔여 소수만 비전 스팟체크**.

> 교체(replace)는 어차피 재캡처하므로 기존 이미지 내용을 몰라도 안전 → 비전 불필요.
> 비전은 전면 폐기가 아니라 **제거 후보 잔여분에만** 좁게 쓴다.

---

## 5. 주의 사항

- **원본 보존 원칙** ([[CLAUDE.md]]) — `en/` MDX·`writing-guides/`·원본 자산은 미수정. 본 작업은 `ko/`와 그 참조 이미지에 한정.
- **CC BY 4.0 / NOTICE** — 교체·제거는 원본 변경이므로 작업 사실을 `NOTICE.md`에 반영. 제거한 Dify 원본 파일은 git 히스토리로 추적 가능.
- **파일명 규칙** — 신규 캡처는 `spx-<챕터>-<주제>-NN.png`. 원본 Dify 자산과 한눈에 구분되어 추적·롤백이 쉽다.
- **🔴 이미지 배치 위치 = `static/images/`만** — Docusaurus는 `/images/...` 참조를 **`static/images/`** 에서 서빙한다(빌드 필수). 루트 `images/`는 Mintlify 시절 원본 사본(보존용)이며, **신규 이미지는 루트 `images/`에 사본을 두지 않는다**(2026-06-17 결정). 여기에만 넣으면 빌드가 깨지므로 반드시 `static/images/`에 둘 것. (파일럿에서 루트 images/만 넣었다가 빌드 실패 → static/images/로 해결.)
- **제거는 비가역** — §4.3 가드(명시적 참조 + alt + 잔여 비전)를 반드시 통과시킨 뒤 삭제. 본문 텍스트는 건드리지 않는다.
- **고아 이미지** — 교체/제거 후 어떤 MDX도 참조하지 않는 `images/` 파일은 정리. 단, ja/zh가 참조 중이면 보류 판단(§ ja·zh는 빌드 대상 아님이나 파일 보존 방침).
- **빌드 검증** — Docusaurus 기준 `npm run build`로 깨진 참조(404) 검출. `docs.json`은 동결(read-only), 네비게이션은 `sidebars.js`.
- **캡처 환경** — spx-agent는 Dify CE self-hosted([[spx_edition_ce]]). 관리자 화면(대시보드·감사로그)은 Keycloak 로그인·시드데이터 필요 → 수동 캡처가 현실적. 공개 화면 일부만 Playwright(`webapp-testing`) 자동 캡처 검토.
- **언어** — alt·캡션은 한국어로 작성. 캡처 화면도 한국어 UI 기준.

---

## 6. 작업 체크리스트

> 본 체크리스트는 **A 스트림(기존 이미지 교체/제거)** 기준. **파일럿 1~2챕터**로 1회전 검증 후 전체 확대. B 스트림(신규 챕터 캡처)은 별도 진행.

### 사전 (1회)
- [ ] 본 설계 + [[decisions]] + [[conventions]](파일명·alt 규칙) 확인
- [ ] `images/` 현재 자산 목록·참조 관계 파악

### [1] 인벤토리
- [ ] ko 전 MDX에서 이미지 참조 추출 스크립트 작성 (3종 형태 모두)
- [ ] 주변 본문 N줄 + alt + 명시적 참조 여부까지 매니페스트 초안 JSON 생성

### [2] 등급화
- [ ] 서브에이전트 병렬로 §3 기준 적용 → 항상 `replace`/`remove` 권장값 + `suggested` 플래그 + 사유 채움
- [ ] `remove` 후보에 §4.3 가드 1·2 자동 적용 (걸리면 `replace`로 안전 전환 또는 비전 스팟체크 표시)

### [3] 검토
- [ ] `suggested:true` 행 사람 확인/뒤집기 → 최종 action 확정
- [ ] `replace` 캡처 목록 확정 (찍을 화면·파일명)

### [4a] 제거
- [ ] 잔여 `remove` 비전 스팟체크 통과 확인
- [ ] 이미지 참조 삭제 + 고아 PNG 삭제

### [4b] 교체
- [ ] (사람) 매니페스트 따라 spx 화면 캡처
- [ ] (선택) 비전 대조 — 캡처가 슬롯·순서와 맞는지 확인 (순서 섞임 = 비가역 오류 방지)
- [ ] (에이전트) 리네임 → **`static/images/` 배치(루트 `images/` 사본 안 함)** → 참조 교체 → 한국어 alt
- [ ] `npm run build`로 깨진 참조 0 검증

### [5] 마감
- [ ] `npm run build` 전체 깨짐 0 확인
- [ ] 고아 자산 정리 / NOTICE 반영
- [ ] [[progress]]에 진척 반영, 신규 용어/규칙 [[conventions]] 갱신

---

## 7. 전체 스윕 (A 스트림) — 통합 매니페스트

> 전체 스윕은 **그룹별 매니페스트**(`image-work/image-manifest-<group>.json`)로 관리한다. 파일럿 네이밍(`image-manifest-quickstart.json` 등)과 일관. 등급화는 **서브에이전트 팬아웃**으로 수행(원문은 서브에이전트 context에서 소모, 메인은 카운트 요약만 받음). 통합·병합 단계 없음 — 그룹 단위로 캡처·삽입·이어가기.

### 대상 (A 스트림, 이미지 보유 ko 파일)

이미지가 있는 디렉터리: `debug/`, `knowledge/`, `nodes/`, `getting-started/`, `tutorials/` (총 49파일). `quick-start`·`key-concepts` 완료 → **잔여 47파일**. `workspace/`·`publish/`·`monitor/`·`analytics-audit/`는 현재 이미지 0개(→ B 스트림).

### 등급화 배치 (서브에이전트 단위)

| 배치 | 범위 | 산출 매니페스트 |
|------|------|-----------|
| 1 | `nodes/**/*.mdx` | `image-work/image-manifest-nodes.json` |
| 2 | `tutorials/workflow-101/*.mdx` | `image-manifest-tutorials-wf101.json` |
| 3 | `tutorials/*.mdx`(101 외) | `image-manifest-tutorials-root.json` |
| 4 | `knowledge/**/*.mdx` | `image-manifest-knowledge.json` |
| 5 | `debug/*.mdx` + `getting-started/introduction.mdx` | `image-manifest-debug-misc.json` |

각 서브에이전트는 **자기 그룹 매니페스트를 직접 작성**하고 메인엔 카운트 요약만 반환. 병합 없음.

### 항목 스키마 (그룹 매니페스트)

```json
{
  "doc": "ko/use-spx-agent/nodes/llm.mdx",
  "line": 42,
  "anchor": "## 모델 설정",
  "ref_syntax": "md | img | frame-md",
  "old_file": "/images/xxx.png",
  "alt": "원본 alt 텍스트",
  "explicit_ref": false,
  "action": "replace | remove",
  "suggested": false,
  "capture": "찍을 화면 설명 (replace) | null",
  "new_file": "spx-<chapter>-<topic>-NN.png (replace) | null",
  "reason": "판정 근거",
  "status": "pending"
}
```

- **`status`** = `pending | captured | inserted | removed | deferred`. 다른 세션 이어가기의 핵심 — 새 세션은 통합 매니페스트에서 `pending`만 골라 진행.
- **파일명 규칙**: `spx-<파일명 마지막 세그먼트 약칭>-<주제>-NN.png` (예: `nodes/llm.mdx` → `spx-llm-01.png`, `knowledge-pipeline-orchestration.mdx` → `spx-kp-orchestration-01.png`). 원본 Dify 자산과 구분.
- **등급화는 grading만** — 서브에이전트는 MDX 편집·이미지 복사 금지(캡처·삽입은 별도 세션).

### 🔴 외부 URL 이미지도 스코프 (2026-06-17 스윕에서 확인)

다수 페이지(특히 `nodes/`·`tutorials/`)는 이미지가 로컬 `/images/`가 아니라 **외부 `https://assets-docs.dify.ai/...` `<img>` 태그**다. 이것도 **명백한 Dify 영문 자산이라 교체 대상**(외부 의존 제거 효과까지). 매니페스트엔 `is_external: true`, `old_file`엔 전체 URL을 기록한다.
- 교체 처리: spx 화면 캡처 → `static/images/`에 로컬 저장 → `<img src="https://assets-docs.dify.ai/...">` 를 `src="/images/spx-...png"` 로 변경.
- **작업량 주의**: 외부 URL이 전체 교체 건의 다수(예: nodes 63건 중 57건 외부). 캡처 볼륨 산정 시 반영.

### 그룹별 등급화 결과 (2026-06-17 스윕)

| 그룹 | 파일 | 참조 | replace | remove | suggested | 외부URL |
|------|-----:|-----:|--------:|-------:|----------:|-------:|
| nodes | 21 | 63 | 63 | 0 | 0 | 57 |
| tutorials-wf101 | 10 | 98 | 98 | 0 | 0 | 0(로컬) |
| tutorials-root | 4 | 32 | 30 | 2 | 9 | 31 |
| knowledge | 7 | 47 | 47 | 0 | 4 | 3 |
| debug-misc | 5 | 17 | 17 | 0 | 1 | ~16 |
| **합계** | **47** | **257** | **255** | **2** | **14** | — |

- 제거 2건: 모두 `build-ai-image-generation-app.mdx`(인트로 장식 컷 + Stability 외부 사이트 캡처). suggested 14건은 사람 확인 대상.
- 코드 예시용 `![alt](url)`(import-text-data·orchestration 등)·`<video>`는 실제 이미지에서 제외됨.

### 이어가기 (다른 세션)

1. [[image-replacement-plan]] + 해당 그룹 `image-work/image-manifest-<group>.json` 로드
2. `status: pending` 항목만 필터 → 캡처([4b])·제거([4a]) 진행
3. 처리 항목 `status` 갱신 + `npm run build` 검증

### 🔴 누락 이미지 검증 (en↔ko diff) — A 스트림 사각 보완

> **문제**: 2026-06-17 등급화는 ko에 *이미 있는* 이미지 참조만 봤다. en/use-dify 원문엔 있으나 **ko에 아예 누락된 이미지**(브리프상 14파일 82건)는 ko에 ref가 없어 **이 스윕으로는 안 잡힌다.** 별도 패스 필요.

**검증 방법 (en↔ko diff)**:

1. **파일 매핑**: `ko/use-spx-agent/<rest>` ↔ `en/use-dify/<rest>`. (경로가 use-spx-agent ↔ use-dify만 다름. Option α로 이동된 일부 경로는 best-match. **spx 전용 챕터는 en 대응 없음 → 스킵**.)
2. 각 쌍에서 **en 이미지 + 섹션 컨텍스트**를 추출 → ko 대응 섹션에 이미지가 있는지 대조.
3. **분류**:
   - en有 + ko 같은 섹션에도 有 → 이미 A 스트림에서 처리됨 (중복 아님)
   - en有 + ko 같은 섹션엔 無 + **그 섹션이 ko에 존재** → **누락 후보** → reader-value 판정 → 필요시 `action: "new"`
   - en 섹션 자체가 ko에 없음(기능 게이트로 삭제됨) → 누락 아님(의도된 제거) → 스킵
4. **산출**: 해당 그룹 매니페스트에 `action: "new"`, `source: "en-missing"`, `en_ref`(원본 en 이미지 경로), `anchor`(ko 삽입 위치) 항목 추가.

**주의**:
- en 이미지도 Dify 영문 자산 → 누락분도 **그대로 가져오지 않고 spx 한국어 캡처**로 채운다. en_ref는 "무엇을 찍을지" 참조용일 뿐.
- "누락"과 "의도된 제거"를 반드시 구분 — ko에 섹션 자체가 없으면 누락이 아니다(CE 미보유 기능 등).
- **실행**: 서브에이전트 팬아웃, 파일쌍 단위로 en+ko 동시 읽기. (A 스트림 등급화와 별개 패스)

#### 누락 검증 결과 (2026-06-17 실행)

| 그룹 | 쌍 | new(추가) | skip | dropped sections |
|------|---:|--------:|----:|----------------:|
| nodes | 23 | 0 | 0 | 4 |
| tutorials-wf101 | 10 | 0 | 1 | 0 |
| tutorials-root | 4 | 0 | 0 | 2 |
| knowledge | 18 | **13** | 34 | 5 |
| debug-misc | 5 | 0 | 0 | 0 |
| **합계** | **60** | **13** | **35** | **11** |

산출: `image-manifest-<group>-missing.json` 5개 (각 `missing[]` + `dropped_sections[]`).

- **누락은 knowledge에 집중.** 특히 `metadata.mdx`(en 30장 전부 ko 미포팅 → 7장 new) + `integrate-knowledge-within-application.mdx`(en 14장 → 3장 new) + `create-knowledge/introduction`·`setting-indexing-methods`(3장). 나머지 그룹은 누락 0.
- **dropped sections 11건은 전부 의도된 제거**(CE/Cloud 게이트·Marketplace·웹크롤·외부KB 등) → 이미지 작업 대상 아님. "단순 en有 ko無"로 셌으면 11건을 오탐할 뻔.
- skip 35건 = 중복/장식/개념 자족이라 추가 불필요(section-aware 판정으로 걸러짐).
- suggested 8건(wf101 1 + knowledge 7)은 사람 확인.
- 브리프상 "82건 누락"은 raw filename diff(드롭 섹션·외부URL·중복 포함)였고, **section-aware + reader-value 판정 후 실제 추가 가치 = 13건**으로 수렴.

---

## 8. B 스트림 (신규 챕터 — 이미지 0개)

> spx 전용 신규 챕터(대시보드·감사로그·권한 설정·사용자/부서)는 기존 이미지가 0개라 "참조 추출 → 교체/제거"가 불가. **처음부터 삽입 위치를 설계하는 캡처 계획**. 결정: A 스트림 이후 진행([[decisions#결정 16]]).

### 입력

| 챕터 | 분석 문서 |
|------|----------|
| 대시보드 | [[spx-dashboard-analysis]] |
| 감사로그 | [[spx-audit-log-analysis]] |
| 권한 설정 | [[spx-app-permissions-analysis]] · [[spx-knowledge-permissions]] |
| 사용자·부서 | [[spx-departments-management]] · [[spx-department-filter-analysis]] |

### 설계 절차

1. 챕터 MDX + 분석 문서를 읽어 **"이 섹션엔 어떤 화면/차트를 새로 넣을까"** 식별. 판정 기준은 A와 동일(reader value)이나, 데이터 시각화·관리자 화면이라 이미지 가치가 높음.
2. **`action: "new"` 항목** 생성 — A와 달리 `old_file`·remove 없음:
   ```json
   { "doc": "...", "anchor": "## KPI 카드", "position": "after-heading | after-paragraph",
     "capture": "찍을 spx 화면", "new_file": "spx-dashboard-NN.png",
     "alt": "한국어 alt", "status": "pending" }
   ```
3. **A와 결정적 차이 = 삽입 위치가 기존 ref가 아니라 '고를 앵커'**. 매니페스트에 위치 + 삽입 형태(`<Frame>` 래핑)를 명시.
4. 사람 캡처 → 에이전트가 앵커에 `<Frame>![alt](/images/spx-...)</Frame>` 삽입 → `static/images/` 배치 → 빌드 검증.

### 주의

- 캡처가 **관리자 화면**(대시보드·감사로그)이라 Keycloak 로그인·시드데이터 필요 → 전부 수동 캡처.
- 데이터 시각화 챕터는 분석 문서가 "KPI/차트마다 ①의미 ②왜 ③활용" 서술 구조([[conventions]] 데이터 시각화 원칙) → **이미지를 그 서술과 1:1 정렬**, 남발 주의.
- 매니페스트는 **챕터별 분리** 권장(`image-manifest-dashboard.json` 등). 챕터 성격이 제각각.
- 등급화(삽입 위치 제안)는 A처럼 **서브에이전트 팬아웃 가능**(챕터 MDX + 분석 문서 동시 읽기).

### B 스트림 캡처 계획 결과 (2026-06-17)

| 챕터 | new | suggested | 매니페스트 |
|------|----:|----------:|-----------|
| 대시보드 | 7 | 2 | `image-manifest-dashboard.json` |
| 감사로그 | 4 | 1 | `image-manifest-audit-log.json` |
| 권한 설정 (앱+지식) | 5 | 1 | `image-manifest-permissions.json` |
| 사용자·부서 | 5 | 2 | `image-manifest-departments.json` |
| **합계** | **21** | **6** | — |

- 전부 관리자 전용 화면(Keycloak 로그인+시드데이터) → **수동 캡처**. action 전부 `new`(제거 없음).
- 절제 적용: 대시보드는 KPI 4·드릴 4를 1:1 남발 않고 전체 1 + 주요 차트/표/컨트롤마다 1로 수렴. 권한·부서는 공용 화면 중복 캡처 회피.
- 분석 문서가 명시한 **비존재 UI 제외**: DENY 행(API 전용)·소유권 이전(UI 미구현)·KC 그룹 동기화(대응 화면 없음).
- ⚠️ 부서 챕터: 헤더 부서 셀렉터 절은 readme.mdx에 미수록(Phase 4 deferred) → 앵커 없어 항목 생성 안 함. 본문 작성 후 보완 필요.

### ⚠️ 추가 커버리지 공백 — workspace/publish/monitor 이식 페이지

A 스트림 등급화·누락 검증은 5그룹(debug·knowledge·nodes·getting-started·tutorials)만 다뤘다. **`workspace/`(readme·app-management·model-providers·personal-settings)·`publish/**`·`monitor/**`의 이식 페이지**는 ko 이미지가 0개라 기존-참조 등급화에서 빠졌고, 누락 검증(en↔ko diff)도 미실행 = **양쪽 패스 모두 누락**. (단 spx 전용인 permissions·departments·dashboard·audit-log는 위 B 스트림에서 처리됨.)
- **2026-06-17 실행 완료** (en↔ko 누락 검증, §7 방식):

| 그룹 | 쌍 | new | skip | dropped |
|------|---:|----:|----:|--------:|
| workspace(이식) | 4 | 6 | 2 | 1 |
| publish | 7 | 8 | 6 | 1 |
| monitor | 3 | 1 | 0 | 0 |
| **합계** | **14** | **15** | **8** | **2** |

산출: `image-manifest-{workspace,publish,monitor}-missing.json`.
- new 15: workspace(앱정보 편집·DSL 내보내기/가져오기·모델 자격증명 3종), publish(MCP 설정·챗플로우 webapp 4·워크플로우 webapp 4), monitor(분석 대시보드 1).
- dropped 2 = 의도된 제거: model-providers **로드 밸런싱**(유료), web-app-settings **Advanced Access Management**(Enterprise). 둘 다 CE 미보유.
- `developing-with-apis`·`embedding-in-websites`·`web-app-settings`·monitor `logs`·`annotation-reply`는 en도 이미지 0(코드/텍스트 문서) → 후보 없음.

---

## 9. 처리 우선순위 (캡처·삽입 순서)

> 등급화·계획이 끝난 항목들을 *어떤 순서로 캡처·삽입*할지. 사람 캡처가 병목이므로 "공백 크고, 한 화면에서 연속 캡처되고, 확실한" 것부터.

### 판단 기준 (6축)

| 축 | 우선 방향 |
|----|----------|
| 1. 독자 공백 | **누락(new) > 교체(replace)** — 이미지가 아예 없는 곳이 더 손해 |
| 2. 진입·핵심 경로 | getting-started · 핵심 노드/지식 먼저 |
| 3. 완주 가능성·사용자 노력 | 캡처 불필요(제거)·소량 먼저 |
| 4. 캡처 효율 | 한 화면 흐름에서 연속 캡처되는 묶음 먼저 |
| 5. 확정도 | `suggested` 적은 것 먼저 |
| 6. 트랙 분리 | 스크린샷 vs 도식 재작성(다이어그램)은 별도 트랙 |

### 추천 순서

| 순위 | 항목 | 근거 |
|------|------|------|
| 🟢 즉시 | 제거 2건 (`build-ai-image-generation-app` L9·L27) | 캡처 0, 참조 삭제만. L9=중복(L114 재등장), L27=Stability 외부(재캡처 불가) |
| 1 | `knowledge/metadata.mdx` 누락 7건 | 이미지 0장(en 30장) = 공백 최대 + 메타데이터 한 흐름 연속 캡처 = 효율 최고 |
| 2 | `getting-started/introduction` 1건 | 진입 페이지, 1건 즉시 완주 |
| 3 | `integrate-knowledge-within-application` 3건 | metadata와 같은 기능 맥락 → 1순위 직후 같은 세션 캡처 (2건 suggested) |
| 4 | `tutorials-wf101` 98건 | 가치 높음, lesson 단위 완주(워크플로우 빌드하며 연속 캡처). 볼륨 커서 집중 세션 |
| 4 | `nodes` 63건(외부 57) | 핵심 노드(llm·agent·knowledge-retrieval·http-request·code) 먼저, 나머지 후순위 |
| ⏸ 별도 | `setting-indexing-methods` Q to P/Q to Q 다이어그램 | 스크린샷 아님 = 도식 재작성 트랙 |
| ⏸ 보류 | suggested 8건(누락) + 14건(교체) | 캡처 전 사람 판정 |

핵심: **공백 → 진입 → 같은-기능-묶음 → 대량-그룹** 순. `metadata.mdx` 7건이 1순위.

> ⚠️ **§9는 2026-06-18 제거 우선 재등급화(§3.4·§10)로 대부분 무효화됨.** 캡처 대상이 ~23건으로 급감(metadata 등 누락 new는 전부 skip 전환). 실제 우선순위는 §10 참조: ① 제거 일괄(캡처 0) → ② 잔여 replace/new ~23건 캡처.

---

## 10. 🔴 제거 우선 재등급화 결과 (2026-06-18)

> 실제 화면 피드백(§3.4)으로 전 매니페스트 재등급화. **기조 역전**: "거의 다 교체" → "거의 다 제거". 캡처 부담 급감, 작업의 대부분이 *삭제*(캡처 불필요).

### A 스트림 (기존 ko 이미지) — 교체/제거

| 그룹 | refs | replace | remove_image | remove_section |
|------|----:|--------:|-------------:|---------------:|
| nodes | 63 | 3 | 50 | 10 |
| tutorials-wf101 | 98 | 11* | 87 | 0 |
| tutorials-root | 32 | 1* | 31 | 0 |
| knowledge | 47 | 0 | 47 | 0 |
| debug-misc | 17 | 0 | 17 | 0 |
| **build** 🆕 | 23 | 3 | 14 | 6 |
| **합계** | **280** | **18** | **246** | **16** |

\* wf101 replace 11 중 10·tutorials-root 1은 `suggested`(토폴로지 컷, 검토자가 제거로 뒤집을 수 있음). 실제 firm replace는 ~7건.

- **section 제거(헤딩+텍스트+이미지) 16건** = 예시/데모 섹션: nodes(doc-extractor 활용 예시·list-operator 혼합 파일 처리 예시·loop 기본 루프 예시·iteration 긴 글 생성 예시·question-classifier 분류 예시·variable-aggregator 분류/조건 예시) + build(version-control 예시 워크플로우).
- **build 그룹은 이번에 신규 추가**(이전 스윕 누락). orchestrate-node는 사용자 지시로 첫 이미지 유지(나머지 suggested).

### A 누락(en↔ko) + 커버리지 갭 — 전부 skip

knowledge-missing 13 new·workspace/publish/monitor-missing 등 **모든 누락 후보 → skip**(텍스트 자족). 제거 우선에서 신규 추가 0.

### B 스트림 (신규 챕터) — 대폭 축소

| 챕터 | 이전 new | 재등급 new | skip |
|------|--------:|----------:|----:|
| 대시보드 | 7 | 2(+1 sugg) | 5 |
| 감사로그 | 4 | 0 | 4 |
| 권한 설정 | 5 | 1(sugg) | 4 |
| 사용자·부서 | 5 | 2 | 3 |
| **합계** | 21 | **5** | 16 |

생존 new = 전체 화면 레이아웃·형태를 봐야 하는 차트·복잡 다이얼로그 등 텍스트 불가 항목만.

### 최종 작업량 (재등급 후)

- **제거 ≈ 262건** (image 246 + 누락 텍스트 포함 section 16) — **캡처 불필요, 삭제만**
- **교체(재캡처) ≈ 18건** (firm ~7 + suggested ~11)
- **신규 캡처(B) = 5건**
- → **실제 캡처 필요 ≈ 23건** (대폭 감소). 나머지는 삭제 + skip.

### 권장 실행 순서 (재등급 반영)

1. **제거 일괄** (캡처 0) — image 제거 246 + section 제거 16. 그룹별 manifest의 remove 항목 처리(참조/섹션 삭제) → 빌드 검증. 비가역이라 section 제거는 사람 1회 확인.
2. **잔여 캡처** ~23건 — firm replace(~7) + B new(5) 먼저, suggested(~11)는 검토 후.

### 실행 진척

- ✅ **2026-06-18 이미지 제거 완료** (`remove_scope:"image"`): 6그룹 ~253건 + 비디오 1(loop 고급) 제거, manifest `status:"removed"`, `npm run build` 성공. replace·section 항목은 보존.
- ✅ **2026-06-18 섹션 제거 완료**(사용자 컨펌): 9개 예시/데모 섹션(헤딩+텍스트+이미지) 통째 삭제 — nodes 8(doc-extractor 활용 예시·iteration 긴 글 생성 예시·list-operator 혼합 파일 처리 예시·loop 기본/고급 루프 예시·question-classifier 분류 예시·variable-aggregator 분류/조건 예시) + build 1(version-control 예시 워크플로우). manifest section 항목 `status:"removed"`, `npm run build` 성공.
- ✅ **2026-06-18 대시보드·감사로그 캡처 삽입 완료** (13장): 사용자 판단 — **신규 기능(대시보드·감사로그)은 §3.4 제거 우선의 예외로 화면 유지**. 비전 스캔(서브에이전트, 누적 이미지 예산 문제로 위임)으로 13장을 섹션 매핑 후 삽입.
  - 대시보드 9: 소개(종합뷰 전체)·KPI 카드·부서별 오브젝트·모델별 토큰·부서별 활동·드릴스루 4(총오브젝트/이용앱/호출량/인기앱 상세) → `static/images/spx-dashboard-01..09.png`
  - 감사로그 4: 화면 구성·이벤트 목록·이벤트 상세·시스템 로그 → `static/images/spx-audit-01..04.png`
  - `npm run build` 성공. (주의: dashboard/audit 매니페스트의 skip 등급은 이 사용자 예외로 무효 — 실제 13장 삽입됨)
- ✅ **2026-06-18 B 스트림(부서·권한·배포) 11장 삽입 완료**: 비전 스캔(서브에이전트) 매핑 후 각 섹션 설명 문단 뒤(중간)에 삽입, `static/images/`만 배치, 빌드 성공.
  - 사용자·부서 5: 멤버 목록·부서 셀 배정·부서 목록·부서 추가·구성원 관리 → `spx-departments-01..05.png`
  - 권한 설정 5: 공개 범위·권한 부여 모달(사용자/부서 탭 2장)·권한 탭 전체·앱 생성 시 권한 → `spx-permissions-01..05.png`
  - 배포 1: 워크플로 탭 → `spx-deploy-01.png`
  - 미커버(추가 캡처 시): 지식 권한 전용 화면, deploy 설정/활동 탭·상단 환경 카드, 대시보드 기간 드롭다운 단독, 감사로그 내보내기 다이얼로그.
- ✅ **대시보드 9장 본문 중간 재배치 완료**(각 섹션 리드 문장 직후).
- ✅ **2026-06-18 튜토리얼 2종 캡처 삽입**: 문서 리더(article-reader 6) + 간단한 챗봇(simple-chatbot 4). 비전 스캔 매핑 후 각 노드 섹션에 삽입(`spx-article-reader-01..06`, `spx-simple-chatbot-01..04`). simple-chatbot 완성 워크플로우는 잔존 Dify 이미지 교체. ⚠️ article-reader **최종 답변 노드(-06)는 대응 헤딩 없어 `### LLM` 끝에 잠정 배치 + `{/* TODO(text-session) */}` 주석** — 텍스트 세션에서 답변 노드 절 신설 후 이동. 텍스트 정렬은 다른 세션 예정.
- 🗑️ **build-ai-image-generation-app.mdx 삭제됨**(사용자) → `sidebars.js` 184행 참조 제거, 빌드 복구. (tutorials-root 매니페스트의 build-ai 항목은 이제 무효.)
- ✅ **2026-06-18 감사로그 정리**: spx-audit-02 제거(참조+파일), 01/03/04 중간 이동.
- ✅ **2026-06-18 모니터·지식설정·디버그 19장 삽입**: 모니터 8(analysis 3·logs 5, 오버뷰 134321/134513은 중복·저해상도라 제외)·지식설정 3(chunking 1·indexing 2)·디버그 8(variable-inspect 3·step-run 3·history-logs 2, 변수상세 중복 173007 제외). 전부 mid-section, `static/images/`만, 빌드 성공.
  - **미첨부 보류**: `130400`(데이터 소스/업로드 화면 — indexing/chunking 부적합 → import-text-data 페이지 후보), monitor 오버뷰 2장, var-detail 중복 1장. annotation-reply는 적합 이미지 0(별도 캡처 필요).
- ✅ **2026-06-18 build·nodes·튜토리얼 교체/삽입**: orchestrate-node 순차·병렬 2(교체)·첫 오버뷰 이미지 제거 / iteration 병렬 모드 토글(교체) / trigger-overview 토글(교체) / webhook-trigger 구성 패널(교체) / customer-service-bot 5(신규) / import-text-data 1(130400 신규) / 디버그 재사용 3(history-logs·step-run에 추가).
- ✅ **2026-06-18 debug·annotation·검색테스트 보정**: history-and-logs 추적↔노드 실행 기록 이미지 교환 + 상세 이미지 추가(`spx-history-logs-03`) / variable-inspect 2번째 이미지 변수 초기화로 이동 / step-run 입력패널·마지막실행 상세 2장(기존 step-run-02 교체) / annotation-reply '주석 응답'→'어노테이션 답변' + 설정·직접입력 2장 / test-retrieval 검색 테스트 1장 / key-concepts 슬래시 삽입(n7) 이미지+텍스트 제거.
- 🗑️ **workflow-101 전체 삭제됨**(사용자) → sidebars.js·디렉터리 제거 확인. wf101 매니페스트(`image-manifest-tutorials-wf101*.json`)는 무효.

---

## 11. ✅ 최종 상태 및 남은 작업 (2026-06-18 기준)

### 이미지 작업 = 사실상 완료

- **검증(grep)**: ko 전체에서 **Dify 영문 이미지(외부 `assets-docs.dify.ai`·원본 로컬·`<video>`) 잔존 0** 확인. 남은 이미지는 전부 `static/images/spx-*` (한국어 spx 캡처).
- A 스트림: 제거(image ~253 + section 9·예시) 완료 / 교체(orchestrate·iteration·trigger·webhook·simple-chatbot 등) 완료.
- B 스트림 신규 챕터: 대시보드·감사로그·부서·권한·배포·모니터(analysis/logs)·annotation·지식설정·검색테스트·import 삽입 완료.
- `npm run build` 반복 통과.

### ✅ 텍스트 수정 완료 (2026-06-18)

1. **quick-start.mdx 노드명 정렬 완료** — 각 노드 추가 단계에 "이름을 ~로 변경" 지시 + 변수 프리픽스가 한국어 캡처명으로 정렬(플랫폼 추출·플랫폼 검증·오류 출력·이미지/문서 분리·문서 텍스트 추출·정보 통합·플랫폼별 반복·스타일 분석·콘텐츠 생성·결과 서식화·최종 출력). 마지막으로 누락됐던 IF/ELSE→`플랫폼 검증` rename 추가. 본문↔캡처 일치.
2. **article-reader.mdx 답변 노드 절 완료** — 사용자가 `## 5단계: 최종 답변 노드 추가` 절로 재작성, `spx-article-reader-06` 정위치 배치, TODO 주석 제거됨.

### ⏳ 남은 작업 (이미지 — 선택/대기)

3. **신규 캡처 대기**: `monitor/annotation-reply` 추가 화면(동작 원리·등록 대화에서/일괄·품질 관리·분석 — 현재 설정·직접입력 2장만, 나머지는 텍스트 자족), `knowledge/permissions`(앱과 동일이라 별도 불필요로 확정).
4. **사용자 중단(캡처 안 함)**: 워크스페이스 배포 설정/활동 탭·환경 카드, 통계감사 대시보드 기간 드롭다운.

### 🧹 정리 후보 (저우선)

5. **무효 매니페스트**: `image-manifest-tutorials-wf101.json`·`-missing`(페이지 삭제), tutorials-root 매니페스트의 build-ai 항목(파일 삭제). 감사 추적용으로 보존 중 — 정리하려면 삭제 가능.
6. **매니페스트 status 동기화**: 사용자 제공 캡처로 삽입된 B 스트림/교체 항목은 매니페스트 `status`가 갱신 안 됨(실제 삽입은 §10·§11 로그가 권위). 재개 시 §11 로그 우선 참조.

### ✅ UI·콘텐츠 보정 (2026-06-18 마지막 라운드)

7. **setting-indexing-methods 이미지 제거** — `spx-indexing-01`·`02` 참조 삭제 + orphan 파일 삭제.
8. **knowledge-pipeline-orchestration 데이터 소스 정리** — `### 온라인 문서`·`### 웹 크롤러`·`### 온라인 드라이브` 3개 섹션 삭제, 1단계 도입 문장을 "데이터 소스로 **파일 업로드**를 사용합니다"로 보정(spx-agent는 파일 업로드만).
9. **🔴 Card 아이콘 깨짐 전역 수정** (`src/theme/MDXComponents/Card.jsx`) — lucide에 없는 FontAwesome/Mintlify 아이콘명이 텍스트로 새던 문제. `FA_TO_LUCIDE` 별칭 맵 추가(window→AppWindow·mobile→Smartphone·microphone→Mic·message/comment→MessageSquare·comments→MessagesSquare·layer-group→Layers·id-card→Contact). 미해결 ASCII 이름은 null(텍스트 누출 방지), 이모지(비-ASCII)는 그대로. **모든 `<Card>`에 적용**.

---

## 12. ✅ 작업 종료 (2026-06-18)

이미지 교체/제거/삽입 + 노드명 정렬 + UI(Card 아이콘) 보정까지 완료. ko 전체 **Dify 영문 이미지 잔존 0**, `npm run build` 통과. 남은 것은 §11의 선택/대기(annotation-reply 추가 캡처, 무효 매니페스트 정리)뿐이며 본 작업 라운드는 종료.
