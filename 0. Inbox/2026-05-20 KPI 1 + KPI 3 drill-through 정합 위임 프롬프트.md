# 위임 프롬프트 — KPI 1 + KPI 3 drill-through 정합 (5/14 누락 트랙 청산 후속)

> 작성일: 2026-05-20
> 작업 위치: `C:\Users\Administrator\Projects\spx-agent\`
> 위임 범위: **KPI 1 (objects) + KPI 3 (calls) drill-through 표/차트의 spec ↔ 코드 정합 점검 + 발견된 drift 보정**. spec 갱신 필요 시 본 위임에서 동시 진행 (KPI 2/4 패턴 동일)
> 선행: 5/20 KPI 2 + KPI 4 정합 위임 완료. 같은 5/14 누락 트랙에서 KPI 1·3도 옛 명세 잔존 가능성 — 본 위임으로 청산
> 가드: `hdd/delegation-standard.md` § 1·2·3·4·5 전부 적용

---

## 배경 — 5/14 누락 트랙 청산 마무리 라운드

5/13 이사님 결정으로 KPI 4종 전부 재정의됐고 spec 3파일은 갱신됐지만, **코드 mock은 5/4 옛 명세 그대로** 6일간 잔존했습니다 ([[1. Daily/2026-05-14.md]] line 27).

5/20 1차 청산 = KPI 2 + KPI 4 완료 (SESSION_HISTORY 2026-05-20 entry). 본 위임 = **KPI 1 + KPI 3 같은 누락 트랙 점검 + 청산 마무리**.

같은 시기 결정사항이라 같은 누락 패턴 가능성 큽니다:
- 5/13 KPI 3번 명명 명확화 ("API 호출" → "부서별 앱 호출", RPS 평균 유지)
- 5/13 라우트 `/dashboard` + API `/console/api/dashboard/` 통일
- 5/13 queryKey `['dashboard', ...]` 접두사 통일
- 5/13 좌하 차트 클릭 없음 (chart-drawer 보류 — H-DASH-20)
- 5/13 표 행 클릭 KPI 4만 (KPI 1·3 표 행 클릭 = 없음)
- 5/13 고정 높이 + 내부 스크롤

5/19 부하 테스트에서 KPI 3 계열 endpoint (`dept-call-count` / `model-call-share` 등)는 5/20 인덱스 추가로 base 응답 시간은 통과한 상태. 단 **표 백엔드가 N+1 패턴인지 단일 쿼리인지 미검증** — KPI 2 dept-activity처럼 N+1 잔존 가능성.

---

## 사전 점검 — 의심점 (위임 측이 검증할 것)

| 영역 | 의심 | 확인 방법 |
|---|---|---|
| **KPI 3 표 컴포넌트 파일명** | `web/app/components/admin/drill-tables/dept-call-rps-table.tsx` 박혀있는데, spec design.md § 컴포넌트 구조 line 103은 `dept-call-table.tsx`로 박힘 — 명명 drift 가능성 (5/4 옛 명세 흔적) | 파일 본문 컬럼이 spec design.md § calls 표 6컬럼(부서/호출/사용자 수/에러/평균 RPS/마지막 사용)과 일치하는지 |
| **KPI 3 백엔드 N+1 잔존** | `dashboard_drill_calls_service.py` 단일 쿼리인지 분리 다중 쿼리인지 | 코드 직접 확인. N+1이면 단일 쿼리로 정합 (KPI 2 패턴) |
| **KPI 3 표 컬럼** | 5/4 옛 명세 잔존 가능성 (예: "DAU" / "RPS" 컬럼만 / 사용자 수 누락 등) | spec design.md § calls 표 vs 코드 1:1 비교 |
| **KPI 3 우하 차트** | spec은 "모델별 호출 분포 또는 부서별 에러율" 둘 중 하나 — 현 코드는 `model-call-share.tsx`로 박힘. 다만 부서별 에러율 폐기됐는지 코드에 잔존하는지 확인 | `useDrillDeptErrorRate`는 5/20 dead code 청산에서 제거됨 → 우하는 `model-call-share` 단독으로 통일된 듯, 정합 확인만 |
| **KPI 1 표 컬럼** | spec design.md § objects 표 5컬럼 (부서/App/KB/Tool/신규) vs 코드 `dept-new-creations-table.tsx` | 1:1 비교 |
| **KPI 1 백엔드** | `dashboard_drill_objects_service.py` — `resource_ownership` 직접 조회. FILTER 패턴(`COUNT(*) FILTER (WHERE ro.resource_type = 'app' AND ro.created_at BETWEEN :start AND :end)`) 단일 쿼리 vs N+1 | 코드 확인 + 단일 쿼리 권장 |
| **KPI 1 PM 확인 항목 잔존** | "신규" 시점 정의 = `resource_ownership.created_at` vs `apps.created_at` (PM 미확정) | 코드 어느 쪽으로 구현됐는지 확인 후 사용자 보고 (수정 안 함, PM 결정 사항) |
| **KPI 1/3 표 행 클릭** | KPI 4 외 다른 표 행 클릭 동작 없어야 함 (5/13 전역 정책). 잔존 시 제거 | 코드 grep |
| **KPI 1/3 좌하/우하 차트 클릭** | 5/13 chart-drawer 보류로 좌하·우하 클릭 인터랙션 없음 (H-DASH-20). 잔존 시 제거 | 코드 grep |
| **queryKey 정합** | `['dashboard', 'drill', metric, chart, params]` 통일. `['admin', 'dashboard', ...]` 5/4 옛 접두사 잔존 여부 | grep `'admin'` in queryKey 패턴 |
| **API 경로 정합** | `/console/api/dashboard/drill/...` 통일. `/console/api/admin/...` 5/4 옛 경로 잔존 여부 | controllers/console/dashboard/dashboard.py + hooks endpoint 확인 |

---

## 진입점 (순서대로 정독 — spec 본문이 단일 진실)

본 프롬프트는 명세 중복 박지 않음. 모든 컬럼/SQL/응답 스키마는 spec 본문이 정답:

1. `CLAUDE.md` — 프로젝트 개요 + 아키텍처 불변식 + 5/13 결정 사항
2. `SESSION_HISTORY.md` 2026-05-20 entries (4개) — 인덱스 추가 / 부하 테스트 / spec 갱신 / KPI 2/4 정합 완료
3. **`hdd/specs/requirements/kpi-drill-through.md`**:
   - § 메트릭 카탈로그 (metric=objects, calls 행)
   - § 비즈니스 규칙
4. **`hdd/specs/design/kpi-drill-through.md`** (구현 단일 진실):
   - § 영역 매핑 (objects, calls)
   - § 컴포넌트 구조 (`drill-charts/objects/`, `drill-charts/calls/`, `drill-tables/`)
   - § chart-area.tsx 설계 + `DRILL_SLOT_MAP`
   - § 표 4종 컬럼 명세 — **objects 표** + **calls 표**
   - § 공통 JOIN 패턴 (audit 단일 SoT) + § objects 전용 (resource_ownership)
   - § API 엔드포인트 + § 응답 스키마
5. **`hdd/specs/tasks/kpi-drill-through.md`**:
   - § 2-A objects (task 15~17)
   - § 2-C calls (task 21~23)
   - § 3 백엔드 (task 35, 37)
6. `hdd/specs/requirements/kpi-cards.md § 4-Axis` — KPI 1·3 카드 본문 정의
   - ⚠️ **stale 참고, 본 위임 청산 범위 외**: line 31/38/97/203/208 + 프로젝트 CLAUDE.md 프로젝트 개요에 "API 호출" 옛 명칭 잔존 (5/13 → "부서별 앱 호출 수"로 결정됐으나 미정합). 본 위임은 KPI 1·3 drill-through 정합만, "API 호출" 명칭 정정은 별도 PR 트랙
7. `hdd/quality-criteria.md § 2 백엔드` — drift 게이트
8. `hdd/delegation-standard.md` § 1·2·3·4·5

---

## 작업 단계

### 1. spec 진입점 정독 + 코드 현 상태 점검 (drift 추출)

진입점 4·5번 본문 정독 후, 다음 파일 본문과 1:1 대조:

**KPI 1 (objects)**:
- 표: `web/app/components/admin/drill-tables/dept-new-creations-table.tsx`
- 좌하: `web/app/components/admin/drill-charts/objects/dept-cumulative.tsx`
- 우하: `web/app/components/admin/drill-charts/objects/top-owners.tsx`
- 백엔드: `api/services/admin/dashboard_drill_objects_service.py`
- 컨트롤러: `api/controllers/console/dashboard/dashboard.py` (drill objects 3종 엔드포인트)
- 프론트 hook: `web/service/use-admin-drill.ts` (drill objects 3종)

**KPI 3 (calls)**:
- 표: `web/app/components/admin/drill-tables/dept-call-rps-table.tsx` (⚠️ spec 박힌 `dept-call-table.tsx`와 명명 차이 — 확인)
- 좌하: `web/app/components/admin/drill-charts/calls/dept-call-count.tsx`
- 우하: `web/app/components/admin/drill-charts/calls/model-call-share.tsx`
- 백엔드: `api/services/admin/dashboard_drill_calls_service.py`
- 컨트롤러: 같은 dashboard.py (drill calls 3종 엔드포인트)
- 프론트 hook: 같은 use-admin-drill.ts

drift 발견 시 매트릭스로 정리 (spec / 현 코드 / 차이 / 처리 방향).

### 2. drift 처리 분기

| drift 유형 | 처리 |
|---|---|
| 코드만 옛 명세 잔존 | 코드 수정 → spec 정합 |
| spec과 코드 둘 다 일관성 깨짐 (다른 spec과 충돌) | spec 먼저 갱신 → 코드 후속 정합. 사용자 보고 |
| spec에 결정 없는 영역 (예: PM 확인 항목 잔존) | 코드 수정 안 함, 사용자 보고만 (PM 컨펌 후 후속 PR) |
| 명명 drift (파일명 spec 박힌 거와 다름) | 사용자 결정 필요 — 코드 파일명 변경 or spec 갱신 둘 중 (사용자 보고) |

### 3. KPI 1 정합 처리
- 점검 결과 drift 발견 시 표 컬럼 / 백엔드 쿼리 / queryKey / API 경로 정합
- "신규" 시점 정의가 코드에 어떻게 박혔는지 raw 인용 보고 (PM 컨펌 사항)
- 백엔드 단일 쿼리 (FILTER 패턴) 권장 — N+1 잔존 시 단일 쿼리로 정합

### 4. KPI 3 정합 처리

> **표 컬럼 단일 진실 = `hdd/specs/design/kpi-drill-through.md` line 237-244 (6컬럼: 부서/호출/사용자 수/에러/평균 RPS/마지막 사용)**.
> SESSION_HISTORY 2026-05-20 KPI 2 entry의 "KPI 3 패턴 부서/호출/RPS(평균)/Top 앱/추세 (5컬럼)" 표현 = KPI 2 정합 맥락 한정 비교 기술 (5/13 결정의 calls 표 6컬럼과 다름). **위임 측은 design.md 6컬럼 본문을 정답으로 채택**.

- 표 컬럼 5/4 옛 명세 잔존 가능성 점검 (DAU/WAU 등) — design.md 6컬럼으로 정합
- 백엔드 N+1 잔존 시 단일 쿼리 또는 CTE로 정합 (KPI 2 dept-activity 패턴 참조)
- 표 컴포넌트 명명 정합 (`dept-call-rps-table.tsx` vs `dept-call-table.tsx` — 사용자 보고 후 결정)
- 우하 `model-call-share.tsx` 데이터 소스 정합 점검 (5/20 dead code 청산 후 잔존 상태)

### 5. 부수 정합 — 5/13 전역 정책 위반 점검
- KPI 1·3 표 행 클릭 동작 잔존 여부 → 제거 (KPI 4만 행 클릭 허용)
- 좌하/우하 차트 클릭 인터랙션 잔존 → 제거 (H-DASH-20 chart-drawer 보류)
- queryKey 접두사 `'admin'` 잔존 → `'dashboard'`로 정합
- API 경로 `/console/api/admin/...` 잔존 → `/console/api/dashboard/...`
- 고정 높이 + 내부 스크롤 정책 누락 영역 박기

### 6. spec 갱신 (drift 발견 시)
- requirements / design / tasks 3파일 일괄 갱신
- frontmatter `last_updated: 2026-05-20`
- 갱신 사유 본문 박음 (KPI 2 design.md § users 표 5/20 갱신 박스 패턴 참조)
- **`hdd/specs/design/kpi-drill-through.md § 응답 스키마`에 objects / calls 표 본문 신설** — 현재 line 424 "objects / calls 표 응답 스키마는 본 spec 범위 외" 명시 상태라 KPI 1·3 정합 검증 시 정답 부재. KPI 2/4 (`DeptActivityResponse` / `AppStatsResponse`) 패턴으로 `DeptNewCreationsResponse` / `DeptCallTableResponse` (또는 spec design.md § calls 표 명세에 맞는 정확한 이름) TypeScript 타입 본문 박기

### 7. 테스트 신설/갱신
- 변경 컴포넌트 vitest 케이스 신설 (KPI 2 `dept-users-table.spec.tsx` 패턴 참조)
- 변경 백엔드 unit test 갱신 (`test_dashboard_drill_users_service.py` 패턴 참조)

---

## 검증 (필수, raw 출력 첨부 의무 — delegation-standard § 2)

### V.1 drift 매트릭스 + 정합 결과
- spec vs 코드 1:1 비교 결과 raw
- 발견된 drift / 해소된 drift / 보류된 drift (PM 컨펌 필요) 분리

### V.2 백엔드 단일 쿼리 (N+1 잔존 회피)
- KPI 1 dept-new-creations + KPI 3 dept-call-table 백엔드 SQL log raw
- 단일 쿼리 패턴 확인. N+1 잔존 시 회귀 보고

### V.3 응답 시간
- KPI 1·3 표 endpoint 응답 시간 5회 평균 (PowerShell `Measure-Command` 또는 k6)
- 5/20 인덱스 효과 유지 확인

### V.4 행 클릭 / 차트 클릭 정책 정합
- KPI 1·3 표 행 클릭 동작 없음 확인 (vitest 또는 코드 grep)
- 좌하/우하 차트 클릭 인터랙션 없음 확인

### V.5 5/13 전역 정합
- queryKey `'admin'` 0건 (grep raw)
- API 경로 `/console/api/admin/dashboard/` 0건 (grep raw)
- 고정 높이 + 내부 스크롤 적용 (CSS / Tailwind 클래스 확인)

### V.6 vitest 통과
- 변경 컴포넌트 단위 테스트 raw 출력
- `pnpm vitest run` (TypeScript 빌드 ≠ vitest, 별개 검증)

### V.7 신규 환경 시뮬레이션
- `docker compose down -v && docker compose up -d && docker compose exec dify-audit npx prisma migrate deploy` raw
- 마트 DDL 변경 없으니 마이그레이션 신설 0건 확인

---

## 진행 금지

- **마트 view/MView DDL 변경** (불필요 — KPI 1은 `spx_resource_ownership` 직접 조회, KPI 3은 `spx_mv_audit_enriched` enriched)
- **신규 인덱스 추가** (5/20 추가 인덱스로 충분, INCLUDE는 별도 PR 보류)
- **KPI 2·4 변경** (5/20 1차 청산 완료, 본 위임 비범위)
- **dept-activity (정적 메인 표) 변경**
- **Gunicorn/psycogreen 변경** (별도 트랙)
- **PM 확인 항목 임의 결정** (예: KPI 1 "신규" 시점 정의 — 코드 어느 쪽인지 보고만, 변경 X)
- **spec 본문 우회한 임의 컬럼/SQL 작성** — spec과 어긋나면 spec이 정답. drift 발견 시 사용자 보고 후 진행 (delegation-standard § 1)
- **자가 검증 거짓** — "정합 완료" 보고 전 V.1~V.7 raw 첨부 (delegation-standard § 2)

---

## 출력 (작업 결과 보고)

`SESSION_HISTORY.md` 2026-05-20 entry 추가 (기존 4개 entry 다음):

1. **drift 점검 매트릭스** — spec vs 코드 비교 표 (KPI 1 + KPI 3 각 영역별)
2. **spec 갱신** (drift 발견 시) — req/design/tasks 3파일 diff 요약
3. **코드 정합 작업** — KPI 1 / KPI 3 변경 파일 목록 + 변경 내용
4. **부수 정합** — 5/13 전역 정책 위반 발견 + 처리
5. **테스트 신설/갱신** — vitest + 백엔드 unit test
6. **검증 V.1~V.7 실측 출력**
7. **보류 / PM 컨펌 필요 항목** — KPI 1 "신규" 시점 정의 등
8. **다음 작업 안내** — 부하 테스트 재측정 권장 / context-bar spec 라벨 정정 (5/19 이월) / CLAUDE.md 불변식 3번 갱신 등

---

## 5/19~5/20 학습 적용 (반드시 의식)

- **자가 검증 거짓 가능성** — "정합 완료" 보고 전 raw 첨부 의무
- **drift 게이트** — spec ↔ 코드 disconnect 시 작업 중단 + 사용자 보고 (H-DOC-01)
- **컨테이너 build 잔존** — 백엔드 변경 후 api/dify-audit 재빌드 검증 (V.7 + H-INFRA-02·03)
- **PowerShell 한글 인코딩** — 보고 시 한글 정상 출력 확인 (delegation-standard § 5)
- **N+1 쿼리 회귀 금지** — KPI 1/3 백엔드가 7쿼리 분리되어 있으면 단일 쿼리 또는 CTE로 정합 (KPI 2 dept-activity 5.8초 병목 교훈)
- **임의 범위 확장 금지** — 본 위임 외 영역 (KPI 2/4 재변경 / context-bar / CLAUDE.md 불변식) 발견 시 보고만, 별도 PR 권장

---

작업 위치: `C:\Users\Administrator\Projects\spx-agent\`

---

## 사용 메모

- KPI 2/4 위임 완료 검증 후 본 파일을 `PROMPT.md`로 이관해서 사용
- 또는 별도 위임 슬롯에 본 파일 그대로 박아서 사용
- 본 위임 종료 후 결과에 따라 부수 정합 (context-bar / CLAUDE.md 불변식) 별도 PR 작성 위임 프롬프트 신설
