--- A-kpi-cards [Completed] ---
세 파일 모두 갱신 완료. 코드 변경이 없어 lint 대상 파일이 아니므로 `pnpm lint`는 생략.

## 변경 보고

**`.claude/hdd/specs/requirements/kpi-cards.md`**
- 변경 섹션: frontmatter(harness/candidates/last_updated), 화면 요구사항(4-Axis 표 신설), 마운트 환경/접근 권한(신설), 기간 선택, 표시 규칙(고정 높이 정책 신설), 클릭 인터랙션(신설), KPI 4 drill-through 레이아웃(신설), 데이터 소스/비즈니스 규칙(audit_events 단일 SoT), Harness 방어 + 후보(2종), PM 확정 필요 항목(신설)
- 의도: KPI 세트를 4축(Volume/Breadth/Depth/Application Analysis)으로 재정의하고, 톱레벨 라우트·로그인 사용자 전체·audit_events SoT를 반영. KPI 4 drill-through 4구역 레이아웃과 표 행 클릭(KPI 4만 Dify 이동) 명시.

**`.claude/hdd/specs/design/kpi-cards.md`**
- 변경 섹션: API 엔드포인트(`/console/api/dashboard/kpi`), Response 타입(KpiResponse 4 필드 재정의), Pydantic 스키마(에러율 → top_app_calls/dept_adopted_apps/api_calls), 쿼리 설계(audit_events 기반 + AppMode/디버그 필터 + RBAC JOIN), 프론트엔드 컴포넌트(KpiCardProps에 subtitle/active 추가, valueSuffix/invertColor 제거), TanStack queryKey `['dashboard', 'kpi', params]`, 고정 높이 토큰, 디자인 토큰
- 의도: KPI 4종을 audit_events SoT 기반 쿼리로 재설계하고, 4축 응답 스키마와 API 경로/queryKey를 admin prefix 폐기 기준에 맞춤. KPI 4 drill-through 상세는 별도 엔드포인트로 분리.

**`.claude/hdd/specs/tasks/kpi-cards.md`**
- 변경 섹션: 1-A 카드/레이아웃(고정 높이·subtitle·active), 1-B 4축 KPI mock 갱신 + 에러율 잔존 코드 제거, 1-C URL 상태/queryKey, 2단계 백엔드(audit_events 기반 서비스 + 신 API 경로 + 권한 데코레이터 교체), 3단계 연동(drill-through·표 클릭·일괄 invalidate 검증), 4단계 테스트(KPI 2 부서 미소속/다중 소속, KPI 4 Top 앱 없음, 고정 높이 회귀, 2개 candidate 검증)
- 의도: 4축 KPI 세트 + 톱레벨 라우트 + audit_events SoT로 구현 체크리스트를 재구성하고 에러율 카드 잔재 제거.

--- B-dept-objects [Completed] ---
세 파일 업데이트 완료. spec 문서만 수정했으므로 lint 대상이 아님(`pnpm lint` 불필요).

## 변경 요약

**`.claude/hdd/specs/requirements/dept-objects.md`**
- 섹션: 화면 요구사항(미배정 통합), 레이아웃(신규 — 고정 높이+내부 스크롤), 인터랙션(드로어 보류), 비즈니스 규칙(state 본질 / `resource_ownership` 직접 / H-DASH-04 강화), 보조 뷰 표(top-owners owner_account_id, new-creations PM 후보), Harness 표(H-DASH-20 추가)
- 의도: "외부" → "미배정" 통합, 차트 드로어 보류 반영, 데이터 소스를 audit 미사용·`resource_ownership` 직접으로 명문화

**`.claude/hdd/specs/design/dept-objects.md`**
- 섹션: API 엔드포인트(`/console/api/dashboard/...`로 admin prefix 폐기, 로그인 전체 허용), Response 타입/스키마(`departments[]`에 미배정 행 통합), 쿼리 설계(스켈레톤 + `apps LEFT JOIN resource_ownership` UNION ALL 운영 방어), 보조 뷰 SQL(top-owners `owner_account_id`, new-creations PM 확인 후보), 프론트(클릭 핸들러 제거, 고정 높이 레이아웃 예시), queryKey `['dashboard', 'dept-objects']`
- 의도: 5/13 라우트·API·queryKey·레이아웃·드로어 보류 결정 일괄 반영, H-DASH-04 강화된 운영 패턴 명문화

**`.claude/hdd/specs/tasks/dept-objects.md`**
- 섹션: 모든 단계의 파일 경로/API/queryKey 갱신, 막대 클릭 핸들러 비추가 명시, `apps LEFT JOIN resource_ownership` 패턴 명시, 고정 높이 스크롤 검증 추가, 보조 뷰 옵션 체크리스트 분리
- 의도: 구현 순서를 새 결정에 정합시키고 H-DASH-04 운영 방어를 백엔드 단계의 기본 경로로 격상

--- C-model-tokens [Completed] ---
spec 파일은 마크다운만 변경되어 별도 lint 대상 없음. 작업 완료 보고:

## 변경 결과 — model-tokens 컴포넌트 (3파일)

**`.claude/hdd/specs/requirements/model-tokens.md`**
- Sections: frontmatter(harness/last_updated), 화면 요구사항(고정 높이/클릭 없음), 기간/필터(URL query string), 데이터 소스(audit_events 가용성 표 신설), 비즈니스 규칙, 사용자 범위, 방어할 Harness, PM 확인 후보
- 한 줄 의도: messages/workflow_runs OLTP 직접 조회 → audit_events 단일 SoT로 전환하면서 P0 collector 보강 의존성을 명시

**`.claude/hdd/specs/design/model-tokens.md`**
- Sections: frontmatter, API 엔드포인트(`/console/api/dashboard/model-tokens`), 데이터 소스, 쿼리 설계(audit_events SQL 2종), 서비스 레이어, 프론트엔드 컴포넌트(경로/`silent:true`/queryKey/staleTime), 레이아웃 정책, 클릭 인터랙션
- 한 줄 의도: API 경로·queryKey·쿼리 SQL을 audit_events 기준으로 재작성하고 차트 드로어 보류(H-DASH-20)에 따라 클릭 인터랙션 제거

**`.claude/hdd/specs/tasks/model-tokens.md`**
- Sections: frontmatter, 0단계(collector 보강 선행 조건 신설), 1~4단계(경로·queryKey·디버깅 필터·"미분류" 검증 갱신)
- 한 줄 의도: collector 보강을 0단계 게이트로 분리하고 H-CAND-audit-* 두 항목을 테스트 체크리스트에 반영

세 파일 모두 "was X, now Y" 이력 없이 현재 결정만 본문에 기술. 다른 컴포넌트 spec / CLAUDE.md / architecture.md / conventions.md / design.md / defect-catalog.md / SESSION_HISTORY.md는 미수정.

--- D-dept-activity [Completed] ---
세 파일 모두 갱신 완료. spec 파일은 lint 대상이 아니므로 (마크다운) `pnpm lint` 검증은 해당 없음.

## 변경 요약

**`requirements/dept-activity.md`**
- 변경 섹션: frontmatter(harness/last_updated), 화면 요구사항(컬럼 재정의), 비즈니스 규칙(전면 재작성), 인터랙션, 마운트 환경, 방어할 Harness
- 의도: 신규 컬럼 폐기 + audit_events 단일 SoT + owner_department_id 단일 기준 + nginx 부서 분류는 PM 확인 후보로 명시

**`design/dept-activity.md`**
- 변경 섹션: API 경로(`/console/api/dashboard/`), Response 스키마(컬럼 축소 + `last_usage` 추가), 데이터 소스, 쿼리(CTE 5단 → 2단으로 audit 기반 축소), 컴포넌트 구조, TanStack queryKey, 레이아웃(고정높이+sticky+스크롤)
- 의도: 단순화된 audit 쿼리와 고정 높이 표 레이아웃을 1:1 구현 가능 수준으로 명세

**`tasks/dept-activity.md`**
- 변경 섹션: 전 단계 체크리스트 재작성, PM 확인 후보 섹션 신설
- 의도: audit 모델 사용 / API 경로·queryKey 갱신 / collector 보강 의존(H-CAND-*) 가드 / 신규 컬럼 검증 제거

## 핵심 결정 반영
- 부서 기준: owner_department_id 단일 (actor 토글 폐기)
- 데이터 소스: audit_events + RBAC LEFT JOIN
- 컬럼: 부서 / 호출 수 / 토큰 사용 / 마지막 사용 (신규 3컬럼 제거)
- 행 클릭 없음, 헤더 정렬/필터는 후보, 외부사용자는 drill-through에서만
- api_call(nginx) 부서 분류 불가 — PM 확인 후보로 명시
- H-CAND-audit-appmode-missing / H-CAND-audit-wf-debug-filter-missing 의존 명시

--- E-dashboard-controls [Completed] ---
3개 spec 파일 재작성 완료. lint 검증은 `.md` 파일이라 해당 없음 (pnpm lint는 코드 대상).

## 변경 보고

**`.claude/hdd/specs/requirements/dashboard-controls.md`**
- 섹션 변경: frontmatter (`mount_environment`, `last_updated`), `마운트 환경`, `기간 옵션`(5개로 축소), `기간 max 정책`(90일 cap), `상태 관리`(URL query string), `새로고침 버튼`(queryKey `['dashboard']`)
- 의도: 모달 마운트 패턴 제거, audit 90일 cap에 맞춰 옵션 5개 + URL state 전환

**`.claude/hdd/specs/design/dashboard-controls.md`**
- 섹션 변경: `컴포넌트 구조`(`use-period-query.ts` 도입), `마운트 통합`(페이지 헤더 슬롯), `상태 관리 흐름`(URL query 기반), `use-period-query` 훅 본문 신규, `새로고침 버튼` queryKey 갱신, `기간 옵션 상수`(5개), `DatePicker` 90일 검증, 컨테이너 layout `<h1>` 제거
- 의도: 페이지 헤더 슬롯 + URL state + 90일 cap에 맞춘 설계 본문 재작성 (nuqs/커스텀 훅 양옵션 제시)

**`.claude/hdd/specs/tasks/dashboard-controls.md`**
- 섹션 변경: 1단계(옵션 5개 축소 + `use-period-query` 신규 + 기존 `use-period.ts` 제거), 2단계(페이지 헤더 슬롯 통합), 3단계(테스트 갱신 — `last3months` 클릭 제거, queryKey 이름 수정, `usePeriodQuery` mock), 4단계(검증 — URL state / 새로고침 / 뒤로가기 시나리오 추가)
- 의도: 기존 모달 통합 task를 페이지 헤더 슬롯 + URL state 전환 task로 교체, 기존 6/6 테스트 신규 정책 반영 절차 명시

## PM 확인 후보 (specs에 명시)

- `requirements.md`: custom 범위 90일 초과 시 UX 디테일 (잠금 vs 자동 클램프 vs 토스트) — 우선 disabled로 진행

Harness 방어 표는 "해당 없음" 유지 (컨트롤만 담당, 데이터 집계 없음). 90일 cap은 옵션 노출 차단으로 사전 방어.

--- F-kpi-drill-through [Completed] ---
세 파일 모두 Option B 스타일로 재작성 완료. 코드 변경이 아니라 spec 문서만 수정이라 lint 대상 없음.

## 변경 요약

**`.claude/hdd/specs/requirements/kpi-drill-through.md`**
- Sections: frontmatter / 마운트 환경 (신규) / 화면 요구사항 / 메트릭 카탈로그 / 종합 뷰 복귀 / 데이터 소스 규칙 (신규) / 비즈니스 규칙 / Harness 적용·후보
- Intent: `/dashboard` 톱레벨 라우트 환경 + ESC 자유 사용 + 4종 KPI 새 메트릭 카탈로그(`objects`/`users`/`calls`/`apps`) + chart-drawer 보류 + KPI 4 단독 행 클릭 + audit 단일 SoT를 반영

**`.claude/hdd/specs/design/kpi-drill-through.md`**
- Sections: 상태 모델 / 훅(URL state + ESC) / 영역 매핑 / 컴포넌트 구조 / chart-area / 표 4종 컬럼 명세 / KPI 카드 확장 / 데이터 소스(audit + resource_ownership 분기) / API 엔드포인트 11종 / 데이터 페칭
- Intent: URL query string 단일 진실 상태 + audit_events 단일 SoT + RBAC JOIN 쿼리 패턴 + queryKey 표준(`['dashboard','drill',...]`) + 11종 엔드포인트 + KPI 4 행 클릭 라우팅을 설계로 확정

**`.claude/hdd/specs/tasks/kpi-drill-through.md`**
- Sections: 1단계 인터랙션 골격 / 2단계 드릴 컴포넌트(objects/users/calls/apps 4그룹) / 3단계 백엔드 / PM·audit 보류 항목 / 4단계 연동 / 5단계 테스트 / 6단계 catalog 등록
- Intent: 새 컴포넌트 매핑(차트 8종 + 표 4종, KPI 4 좌상 카드)으로 체크리스트 갱신 + 백엔드 11종 엔드포인트 + audit 의존 후속 PM 확인 항목 명시

모든 [x] 마크는 컴포넌트 매핑이 바뀌면서 무효해졌으므로 빈 체크박스로 재시작 (이전 5/6 작업물의 일부는 재활용 가능하지만 spec 본문에 history는 남기지 않음 — Option B).

--- G-chart-drawer [Completed] ---
3개 파일 모두 보류 처리 완료했습니다. lint 대상 코드 변경은 없습니다 (spec 마크다운만).

## 변경 요약

### `.claude/hdd/specs/requirements/chart-drawer.md`
- frontmatter: `status: "🔒 보류 (2026-05-13 이사님)"` 추가, `last_updated` 갱신
- 본문 상단 보류 안내 블록 삽입
- "활성 사용자" → "사용자 수" (드로어 레이아웃 ASCII, 요약 카드 3종 표)
- "에러 수 카드 / 24h 고정 아님" 표현 → "에러" / "24h" 제거
- queryKey: `['admin', 'dashboard', 'drawer', ...]` → `['dashboard', 'drawer', ...]`

### `.claude/hdd/specs/design/chart-drawer.md`
- frontmatter: `status` 필드 + `last_updated` 갱신
- 본문 상단 보류 안내 블록 삽입
- 컴포넌트 구조 주석: "활성사용자/에러 수" → "사용자 수/에러"
- API 엔드포인트: `/console/api/admin/dashboard/drawer` → `/console/api/dashboard/drawer`, `Authorization: admin` → `로그인 사용자`, "에러율 24h" 표현 제거
- TS/Pydantic 응답 타입 주석: "24h 고정" 제거
- 쿼리 설계: `get_active_users()` → `get_users()`, `sp_object_ownership` → `resource_ownership` (회사 표준 prefix 없음), 미배정 처리 fallback 명칭 동기화
- TanStack queryKey + URL: `'admin'` 세그먼트 제거, `/admin/dashboard/drawer` → `/dashboard/drawer`, more-button 라우터 push `/admin/audit-logs` → `/dashboard/audit-logs`

### `.claude/hdd/specs/tasks/chart-drawer.md`
- frontmatter: `status` 필드 + `last_updated` 갱신
- 본문 상단 보류 안내 블록 삽입
- "24h 고정 정책 확인" 항목 → "사용자 기간 일관성 확인"으로 재구성
- 요약 카드 3종 명칭 "활성사용자/에러 수" → "사용자 수/에러"
- 백엔드 라우트 경로 `controllers/console/admin/dashboard.py` → `controllers/console/dashboard/drawer.py`, `get_active_users()` → `get_users()`, `sp_object_ownership` → `resource_ownership`, "24h 고정 아님" 제거
- queryKey 표준에서 `'admin'` 세그먼트 제거

본문(트리거 차트 연결, 닫기 메커니즘, 포커스 트랩, Harness 후보 등)은 재검토용 base로 그대로 보존했습니다.

--- H-context-bar [Completed] ---
세 파일 갱신 완료. 코드 변경이 없어 lint 검증은 생략합니다 (spec 문서만 수정).

## 변경 요약

**`hdd/specs/requirements/context-bar.md`**
- 섹션 변경: 환경 적응 결정 표 → "범위 (Slim 컨텍스트 바)" — 모달 환경 언급 제거, `/dashboard` 라우트 환경 반영
- 메트릭 라벨 매핑: 4종으로 갱신(`objects/users/calls/apps`), `users` "활성 사용자"→"사용자 수", `errors` 제거 → `apps` "앱별 통계" 추가, 그레인 컬럼 추가(부서명 vs 앱명)
- "마운트 환경 제약" → "마운트 환경" — 페이지 라우트/ESC 자유 반영, 데이터 소스(없음) 명시
- 기존 컴포넌트 영향에 `queryKey` 무영향 명시
- last_updated: 2026-05-13
- 의도: 모달 환경 제약 제거 + KPI 4종(apps) 라벨 정리

**`hdd/specs/design/context-bar.md`**
- 상태 모델: `KpiMetric` 유니온을 `'objects' | 'users' | 'calls' | 'apps'`로 명시
- `METRIC_LABELS` 4종 매핑(`apps: '앱별 통계'`, `users: '사용자 수'`) + 그레인 주석
- DashboardPage 통합 예시 경로를 `app/(commonLayout)/dashboard/page.tsx`로
- API/백엔드 섹션에 queryKey 영향 없음 명시
- last_updated: 2026-05-13
- 의도: 메트릭 4종 신규 셋 + 페이지 라우트 마운트 경로 반영

**`hdd/specs/tasks/context-bar.md`**
- 1단계 #1: 라벨 매핑 4종(`apps` 포함) 명시, 24h 접미사 노트 삭제
- 2단계 #9: "페이지 컨텍스트 영역" 명시
- 3단계 #14: 4종 메트릭 전환 검증 명시
- 3단계 #16: ESC 메시지를 "모달 닫기 충돌" → "drill-through 훅 ESC 일괄 해제와 충돌 없음"으로 수정
- last_updated: 2026-05-13
- 의도: 4종 메트릭/페이지 환경 반영, 모달 잔재 제거

=== ALL SPEC UPDATES COMPLETE in 575s ===
Verify with: git status