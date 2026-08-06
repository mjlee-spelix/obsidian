---
tags: [프로젝트, dify, AI-Agent, HDD]
type: harness/sensor
date: 2026-04-29
---
# Quality Criteria — 자기 평가 기준

> 구현 완료 후 에이전트가 **스스로 품질을 검증**하기 위한 문서.
> Generator(구현)와 Evaluator(검증)를 분리하여 객관적 판단을 유도한다.

## 1. 컴포넌트별 품질 등급

**Phase 1 (정적 대시보드)**

| 컴포넌트 | 프론트 | 백엔드 | 연동 | 테스트 | Harness | 등급 |
|---------|--------|--------|------|--------|---------|------|
| 대시보드 컨트롤 | ✅ | n/a | ✅ | 🔧 | n/a | **B** |
| KPI 카드 | ✅ | ✅ | ✅ | ✅ | ✅ | **A** |
| 부서별 오브젝트 차트 | ✅ | ✅ | ✅ | ✅ | ✅ | **A** |
| 모델별 토큰 차트 | ✅ | ✅ | ✅ | ✅ | ✅ | **A** |
| 부서별 활동 테이블 | ✅ | ✅ | ✅ | ✅ | ✅ | **A** |
| 설정 모달 사이드바 | - | - | - | - | - | **D** (태영님 수령 대기) |

**Phase 2 (동적 인터랙션)**

| 컴포넌트     | 프론트 | 백엔드 | 연동  | 테스트 | Harness | 등급                                   |
| -------- | --- | --- | --- | --- | ------- | ------------------------------------ |
| KPI 드릴스루 | ✅ (mock) | -   | -   | -   | -       | **C** (1단계 인터랙션 골격 + 2단계 차트 8종+표 4종 정정 완료, 백엔드 미착수. RBAC 8종 블로커, audit·messages 4종 즉시 가능) |
| 차트 드로어   | -   | -   | -   | -   | -       | **D** (spec 완료, CSV는 화면만)            |
| 컨텍스트 바   | -   | n/a | n/a | -   | n/a     | **D** (spec 완료, slim 모델 — 활성 메트릭 칩만) |

> 각 셀: ✅ 완료, 🔧 진행중, ❌ 미흡, `-` 미착수

### 등급 기준

| 등급 | 조건 |
|------|------|
| **A** | 프론트 + 백엔드 + 연동 완료, 테스트 존재 및 통과, Harness 방어 전부 적용 |
| **B** | 프론트 + 백엔드 + 연동 완료, 기본 테스트 존재 |
| **C** | 일부 레이어만 구현 (예: 프론트 목업만, 백엔드만) |
| **D** | 미착수 |

### 등급 판정 규칙

- 구현 단계 하나 끝날 때마다 이 테이블 갱신
- **B 이상**이어야 해당 컴포넌트 "완료"로 간주
- **A**를 받으려면 공통 체크리스트 + 컴포넌트별 체크리스트 전부 pass

---

## 2. 공통 검증 체크리스트

> 모든 컴포넌트에 적용. 하나라도 fail이면 등급 A 불가.

### 백엔드

- [ ] **디버깅 필터**: 집계 쿼리에 `invoke_from != 'debugger'` 필터 존재 (H-DASH-03)
- [ ] **AppMode 분기**: ADVANCED_CHAT은 `messages`만 읽음, `workflow_runs` 미참조 (H-DASH-01)
- [ ] **미배정 처리**: `spx_resource_ownership`에 없는 앱은 LEFT JOIN + "미배정" fallback (H-DASH-04)
- [ ] **`spx_` 접두사**: 신규 테이블/모델은 전부 `spx_` 접두사 (RBAC·마트 공통)
- [ ] **Dify 무수정**: 기존 Dify 파일 수정 없이 신규 모듈만 추가
- [ ] **파일 위치**: `controllers/console/dashboard/`, `services/admin/`에 생성 (architecture.md 준수)
- [ ] **네이밍**: snake_case 파일, PascalCase 클래스, `@classmethod` Service (conventions.md 준수)
- [ ] **데코레이터**: `setup_required`, `login_required`, `account_initialization_required` 적용
- [ ] **tenant 필터 적용**: 컨트롤러는 `current_user.current_tenant_id`를 service에 전달, service는 tenant_id로 쿼리 필터링 (Dify 표준 패턴, 단일 tenant 환경에서도 코드는 다중 tenant 대응 작성)
- [ ] **마트 게이트 (2026-05-18 신설, PR 직전 필수)**: `hdd/specs/tasks/data-mart.md § 6단계 30번` grep 검증 4종 모두 통과. 30-1/30-2/30-3은 0 hit, 30-4는 hit 있음. 1건이라도 위반 시 PR 차단. 위반의 의미: 마트 인터페이스 우회 → H-DASH-01/03 방어 누락 가능
- [ ] **drift 게이트 (2026-05-19 신설)**: `hdd/design.md § 2.5.4` DDL 코드 블록 vs 실제 DB DDL/마이그레이션 본문 1:1 정합. **검증 수단**: (a) `scripts/check-mart-drift.ps1`(작성 예정) 자동화 또는 (b) 수동 — `docker compose exec db_postgres psql -d dify -c "\d+ public.spx_mv_audit_enriched"` 출력 vs design.md DDL grep diff. **불일치 발견 시**: 작업 즉시 중단 + 사용자 보고, 자동 보정 금지(설계 의도 ↔ 마이그레이션 어느 쪽이 진실인지 결정 필요). 본 갱신/마이그레이션 후속 작업 동시 처리 의무 (5/15 audit rename / 5/18 drift 1·2차 재발 방지). 관련 결함: H-DOC-01

> ⚠️ **잔존 stale 추적**: (1) ~~`sp_` 접두사 / RBAC=prefix 없음~~ → **2026-06-09 정정 완료** — RBAC 5종·마트 객체 **전부 `spx_` 접두사** (위 체크리스트 반영됨, 근거: `references/rbac-schema.md` 🔴 배너). (2) ~~**미정정 잔존**: `controllers/console/admin/` → `controllers/console/dashboard/`~~ → **2026-06-10 정정 완료** (L62 경로 갱신). 단, `api/controllers/console/admin/`은 Dify 원본 관리자 모듈(admin_required 등)이므로 삭제 대상 아님.

### 프론트엔드

- [ ] **3상태 처리**: 로딩 / 에러 / 데이터 없음 각각 UI 존재
- [ ] **빈 상태 골격 유지**: 데이터 없어도 **제목**(표는 +컬럼, 디멘전 차트는 +축 이름) 항상 렌더. `if (length===0) return <맨 박스>`로 **제목까지 날리는 early-return 금지**. 결측치 `0`/없음 → `-`(tertiary). 부서=전수, 앱·사용자·모델=활성만+안내문구. → `conventions.md § 빈 상태 / 골격 / 결측치 정책` (SoT)
- [ ] **staleTime 5분**: TanStack Query 훅에 `staleTime: 5 * 60 * 1000` 설정
- [ ] **queryKey 표준화**: `['admin', 'dashboard', '<component>', params]` — 헤더 새로고침 일괄 invalidate 대비
- [ ] **컴포넌트 구조**: `index.tsx` 메인 + 하위 컴포넌트 분리 (conventions.md 준수)
- [ ] **네이밍**: kebab-case 파일, PascalCase 컴포넌트, camelCase 훅
- [ ] **디자인 토큰 사용**: raw 색상값 금지, `text-text-*` / `bg-components-*` / `util-colors-*` 토큰 사용 (다크모드 자동 대응)
- [ ] **숫자 포맷**: `<10K` raw + 콤마, `≥10K` K/M 압축 + hover 정확값 (전역 규칙)
- [ ] **화면 설계 1:1 대조**: spec frontmatter `design_image` 경로의 이미지를 읽고 비교 (페이지 헤더, 색상, 라벨, 포맷 누락 없는지 점검)
- [ ] **상태 관리 환경 매칭**: 마운트 환경(설정 모달/독립 페이지)에 따라 URL state vs React state 적절히 선택 (모달 환경에서는 useSearchParams 불가)
- [ ] **마운트 환경 자체 UI 충돌 없음**: 모달 헤더가 자동 렌더하는 제목과 본 컴포넌트의 제목/헤더 영역 중복 없음
- [ ] **ESC 자체 핸들링 금지**: 모달 환경의 `@base-ui/react/dialog` ESC=닫기 UX 계약 양보 (drill-through, drawer, context-bar 모두 ESC 미사용 — 클릭/버튼/포커스로 대체) — CLAUDE.md 마운트 환경 주의 참조
- [ ] **카드 컨테이너 토큰 표준** (2026-05-04 갱신): `rounded-xl border-[0.5px] border-divider-regular bg-components-card-bg px-6 py-4 shadow-xs` (PROVIDER 탭과 동일 패턴, 모달 톤 통일)
- [ ] **기존 토큰/컴포넌트 우선** (conventions.md § 4): 신규 className/컴포넌트 만들기 전 (a) 디자인 토큰 grep (b) `base/` 컴포넌트 확인 (c) 모달 내 다른 탭 동일 패턴 확인. 새 도입은 기존에 없을 때만, 도입 시 표준으로 등록.
- [ ] **부서 식별자 필드명**: 응답·prop·SQL 모두 `department_id` / `department_name` (DDL 정합, snake_case 통일)

### 테스트

- [ ] **백엔드 단위 테스트**: `api/tests/unit_tests/` 에 `test_*.py` 존재
- [ ] **테스트 통과**: `pytest` 실행 시 전부 pass

---

## 3. 컴포넌트별 검증 체크리스트

### 대시보드 컨트롤

- [ ] 9개 기간 옵션 + 사용자 지정 DatePicker (Dify `LongTimeRangePicker` 동일)
- [ ] 기본값 `last7days`
- [ ] **자체 제목/헤더 없음** — 모달이 "대시보드"를 자동 렌더, 우리는 컨트롤만
- [ ] **모달 헤더 우측 슬롯에 조건부 삽입** (PROVIDER 탭 SearchInput 패턴)
- [ ] 새로고침 버튼 — `invalidateQueries(['admin', 'dashboard'])`
- [ ] `useIsFetching` 기반 회전 애니메이션
- [ ] period 상태는 React state로 관리 (모달 환경 제약)
- [ ] period prop으로 자식 컴포넌트(KPI 등)에 전달

### KPI 카드

- [ ] 증감률 분모 0 처리: 이전 기간 값 0 → "+신규" 또는 "-" 표시 (H-DASH-07)
- [ ] 이전 기간 데이터 없음: 증감률 "N/A" 표시 (H-DASH-05)
- [ ] NULL 사용자 제외: `from_end_user_id IS NOT NULL` 필터 (H-DASH-08)
- [ ] 4개 카드 값 정합성: 총 오브젝트, 활성 사용자, API 호출, 토큰 사용 각각 올바른 테이블에서 집계
- [ ] 제목 옆 (?) 정보 아이콘 + hover 툴팁 (카드 의미 설명)
- [ ] 제목 밑 기간 라벨 ("지난 7 일")
- [ ] 부가 정보(App/KB/Tool) 색상 구분 (blue/teal/orange — 부서별 차트와 통일)
- [ ] 큰 숫자 hover 시 정확값 툴팁 (≥10K일 때)

### 부서별 오브젝트 차트

- [ ] **수평 스택 막대** (Y=부서, X=수치) — 화면 설계 방향 일치
- [ ] 3색 스택: App=blue / KB=teal / Tool=orange (KPI 카드 부가 정보와 색상 통일)
- [ ] 막대 우측에 부서별 합계 숫자 표시
- [ ] 범례 우상단 (App / KB / Tool)
- [ ] 미배정 오브젝트: "미배정" 막대로 별도 표시 (H-DASH-04)
- [ ] 부서 구조: 플랫 구조 (parent_id 트리 미사용)
- [ ] 기간 무관 (현재 시점 누적)

### 모델별 토큰 차트

- [ ] 수평 막대 + 내림차순 정렬
- [ ] 각 막대 우측에 정확값 라벨 (K/M 압축, hover raw)
- [ ] 차트 하단에 합계 표시 ("합계: 28.4M tokens / 지난 30 일")
- [ ] WORKFLOW "미분류 (Workflow)" 버킷 별도 막대 (H-DASH-02)
- [ ] 로컬 모델 라벨: `LOCAL_PROVIDERS` 매칭 시 "(로컬)" 라벨 + 색상 구분 (H-DASH-09)
- [ ] 이중카운트 방지: ADVANCED_CHAT 앱의 토큰이 messages에서만 집계 (H-DASH-01)

### 부서별 활동 테이블

- [ ] CTE 쿼리 성능: 부서 10개, 앱 100개, 메시지 10만건 기준 응답 2초 이내 (H-DASH-10)
- [ ] 제목 옆 기간 라벨 ("부서별 활동 — 지난 30 일")
- [ ] 신규 컬럼 `+` 부호 prefix (App/KB/Tool)
- [ ] 호출/토큰 K/M 압축 + hover 정확값 (전역 규칙)
- [ ] 기본 표시: 상위 10개 + "전체 보기 →" 링크 (우상단)
- [ ] 페이지네이션: 전체 보기 시 동작
- [ ] 합계 정합성: 전체 부서 합계 = KPI 카드의 호출/토큰 수치

### 설정 모달 사이드바

- [ ] 탭 추가: `constants.ts` + `index.tsx` 2곳 수정으로 등록 (H-DASH-15)
- [ ] Git 충돌 최소화: 김이사님/승랑님 작업 영역과 분리

---

## 4. 평가 절차

```
컴포넌트 구현 완료
  ↓
1. 공통 체크리스트 검증 → fail 항목 나열
  ↓
2. 해당 컴포넌트 체크리스트 검증 → fail 항목 나열
  ↓
3. 등급 판정
   - 전부 pass + 테스트 통과 → A
   - 전 레이어 완료 + 기본 테스트 → B
   - 일부만 → C
  ↓
4. 테이블 갱신 (섹션 1)
  ↓
5. fail 항목 있으면 수정 후 재검증
```

> **중요**: 등급 A가 아니면 해당 컴포넌트는 "검증 미완료" 상태.
> fail 항목을 구체적으로 보고하고, 수정 후 재평가한다.

---

## 5. Spec 작성 검증 게이트 (2026-05-06 신설)

> **배경**: 2026-05-06 H-DASH-16 발견 — 5/4 작성 `kpi-drill-through` design.md가 화면 설계 PDF와 어긋나 있었으나, 본인이 작성하고 본인이 검토하는 구조라 spec 단계에서 잡히지 않고 구현(Phase 2 2단계) 후 사용자 시각 검토 시점에 발견됨. 손실: 차트 4개 작성 후 폐기 + 표 4개 추가 작성 + spec 3파일 정정.
> **이 게이트는 spec 작성 후 즉시 적용**되어, 구현 시작 전에 spec 자체의 정합성을 검증한다.

### 5.1 외부 정합성 — 화면 설계 1:1 대조 게이트

> spec frontmatter의 `design_image:` 또는 `reference_pdf:`가 있으면 **구현 시작 전에 의무 통과**.

- [ ] **이미지/PDF 직접 읽기**: spec 작성자(또는 별도 평가자)가 frontmatter의 이미지/PDF 페이지를 **눈으로 직접 확인**
- [ ] **위치/형태 대조**: 화면 설계의 영역 배치(좌/우/상/하), 컴포넌트 종류(차트/표/카드/리스트)가 spec과 일치
- [ ] **차트 타입 대조**: 화면 설계가 막대/도넛/라인/표 중 무엇을 명시하는지 → spec과 일치
- [ ] **컬럼/필드 대조**: 표가 있다면 화면 설계의 컬럼 헤더/순서/포맷이 spec의 컬럼 명세와 일치
- [ ] **고정 영역 명세 확인**: 화면 설계에 "고정된 N개 영역" 같은 동작 명세가 있으면 spec이 이를 위반하지 않는지
- [ ] **불변식 확인**: 화면 설계에 "위치/행 수/색상 체계 그대로" 같은 명시 제약이 있으면 spec이 이를 보존하는지

### 5.2 내부 정합성 — Spec 3파일 상호 검증 게이트

> requirements ↔ design ↔ tasks가 **같은 추상 수준에서 동일한 내용**을 표현하는지.

- [ ] **메트릭 카탈로그 일치**: requirements의 메트릭 표 N행 = design의 컴포넌트 매핑 N행 = tasks의 task 항목 N개
- [ ] **컴포넌트 이름 일치**: 세 파일에 나오는 컴포넌트 이름이 동일 (rename 누락 없음)
- [ ] **고정 영역 일치**: 한쪽이 "차트 2 + 표 1"이면 다른 쪽도 "차트 2 + 표 1"
- [ ] **불변식 흡수**: requirements의 명시 제약("위치 유지" 등)이 design의 코드 구조에 반영
- [ ] **Phase 1 확장 정합성**: Phase 2 spec이 Phase 1 컴포넌트(KPI 카드, dept-objects 등)를 확장한다면 Phase 1 spec의 prop/구조와 호환

### 5.3 적용 시점

| 시점 | 게이트 적용 |
|---|---|
| Spec 신규 작성 직후 | 5.1 + 5.2 전부 |
| Spec 일부 갱신 시 | 갱신 영역의 5.1 + 5.2 해당 항목 |
| 구현 시작 전 | 5.1 + 5.2 재검증 (작성 시점 통과 사실 신뢰하지 말고 다시) |
| 사용자 보고로 어긋남 발견 | 사후 재정정 + defect-catalog 등록 (현 사례 H-DASH-16) |

### 5.4 자기 검토 한계 인지

본인이 작성한 spec을 본인이 검토하면 같은 추상 수준에서 같은 함정에 빠질 수 있다. 다음 중 하나로 강제 분리 권장:

1. **외부 산출물(이미지/PDF) 직접 비교** — 가장 확실. 추상 수준이 다른 산출물끼리 비교하면 어긋남이 즉시 보임.
2. **AI 에이전트에게 검토 위임** — "이 spec과 화면 설계 이미지를 1:1 비교하고 어긋남 모두 나열" 같은 단순 위임. 본인 추상에 갇히지 않음.
3. **시간차** — 작성 후 4시간 또는 다음 날 다시 보기. 단기 기억의 "내가 의도한 것"과 "실제 쓴 것"이 분리됨.
