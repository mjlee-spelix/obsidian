---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: 부서별 활동 테이블
harness: [H-DASH-01, H-DASH-03, H-DASH-04, H-DASH-10, H-DASH-13, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
date: 2026-04-29
last_updated: 2026-05-13
---
# 부서별 활동 테이블 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/dept-activity.md|Design]]
> 순서: 프론트엔드(목업) → 백엔드 → 연동 → 테스트

## 1단계: 프론트엔드 (목업 데이터)

- [ ] 1. `web/app/components/admin/dept-activity/index.tsx` — 컨테이너
  - [ ] 1-1. URL query string에서 `start`/`end` 읽기 (페이지 헤더 `dashboard-controls`가 단일 진실)
  - [ ] 1-2. `useDashboardDeptActivity` 훅 호출 (목업 단계는 fixture 데이터로 대체)
- [ ] 2. `table.tsx` — 고정 높이 + sticky header + 본문 스크롤
  - [ ] 2-1. 컬럼: 부서 / 신규 App / 신규 KB / 신규 Tool / 호출 수 / 토큰 사용
  - [ ] 2-2. 호출/토큰: 숫자 포맷 `formatCount` 적용 + hover `title` 정확값
  - [ ] 2-3. 신규 App/KB/Tool: raw 숫자. 0이면 `-`
  - [ ] 2-4. "미배정" 행은 항상 마지막 + 회색 처리 (`text-text-tertiary`)
  - [ ] 2-5. 행 클릭 핸들러 없음 (cursor default)
- [ ] 3. 목업 데이터 (부서 5~6개 + "미배정") + 활동 없는 부서 1개 포함
- [ ] 4. 로딩/에러/데이터 없음 상태 처리
- [ ] 5. 외곽 카드 고정 높이 검증 — 데이터 양과 무관하게 크기 고정 (전역 레이아웃 정책)

## 2단계: 백엔드

- [ ] 6. SQLAlchemy 모델 import 확인
  - [ ] `Department` (departments)
  - [ ] `ResourceOwnership` (resource_ownership, `resource_type='app'`, `owner_department_id`)
  - [ ] `AuditEvent` (audit_events) — top-level `event_type` / `target_id` / `occurred_at` / `details` JSONB
- [ ] 7. `DashboardDeptActivityService` 클래스 생성
  - [ ] 7-1. `get_dept_activity()` — CTE 3단계 쿼리
    - [ ] CTE 1: `app_dept` (resource_ownership.resource_type='app')
    - [ ] CTE 2: `dept_new_objects` (resource_ownership 직접 — `created_at BETWEEN`, resource_type별 FILTER)
    - [ ] CTE 3: `dept_activity` (audit_events LEFT JOIN app_dept — 호출/토큰)
    - [ ] 최종: `departments` LEFT JOIN dept_new_objects + dept_activity + UNION ALL "미배정" 행
  - [ ] 7-2. 디버깅 필터 (H-DASH-03)
    - [ ] `message_send`: `details->>'invokeFrom' != 'debugger'`
    - [ ] `workflow_execute`: `details->>'triggeredFrom' != 'debugging'` — collector 보강 후 가동(H-CAND-audit-wf-debug-filter-missing)
  - [ ] 7-3. AppMode 분기 (H-DASH-01) — collector `appMode` 보강(H-CAND-audit-appmode-missing) 완료 후 ETL 본가동. 보강 전 임시 OLTP 우회 시 `apps.mode` JOIN
  - [ ] 7-4. "미배정"은 `owner_department_id IS NULL` (H-DASH-04)
- [ ] 8. API 엔드포인트 등록 — `GET /console/api/dashboard/dept-activity` (Blueprint: `controllers/console/dashboard/`)
  - [ ] 8-1. 라우트 가드: 로그인 사용자 전체 — 관리자 가드 추가 안 함
  - [ ] 8-2. Pydantic 응답 검증

> ※ CSV 내보내기는 본 spec에서 제외. Phase 1 정적 대시보드 범위 외.

## 3단계: 연동

- [ ] 9. `web/service/use-dashboard-dept-activity.ts` 훅 — 목업을 실제 API로 교체 (`staleTime: 5 * 60 * 1000`)
- [ ] 10. queryKey 확인 — `['dashboard', 'dept-activity', params]` (페이지 헤더 새로고침이 일괄 invalidate)
- [ ] 11. 실제 데이터로 표 검증 — 부서별 신규 App·KB·Tool + 호출/토큰 표시 + "미배정" 마지막 행

## 4단계: 테스트

- [ ] 12. H-DASH-01 회귀: ADVANCED_CHAT 앱이 message_send + workflow_execute 양쪽에 잡히지 않는지 (collector 보강 후 본 검증, 보강 전엔 mock으로 분기 검증)
- [ ] 13. H-DASH-03 회귀: 디버깅 이벤트가 호출/토큰에서 제외되는지 (message_send `invokeFrom='debugger'` + workflow_execute `triggeredFrom='debugging'` mock 데이터)
- [ ] 14. H-DASH-04 회귀: `resource_ownership`에 없는 앱 활동이 "미배정" 행에 집계되는지 + 표 마지막에 위치하는지
- [ ] 15. H-DASH-10 성능: 부서 10개, 앱 100개, 이벤트 10만건 기준 쿼리 응답 2초 이내
- [ ] 16. H-DASH-13 회귀: SQLAlchemy 모델 클래스 경유로 컬럼명 직접 노출 없음
- [ ] 17. 전체 부서 합계 + "미배정" = KPI 카드 "API 호출" 수치와 ±0.1% 이내
- [ ] 18. 활동 없는 부서가 행으로 표시되되 신규/호출/토큰 모두 `-`
- [ ] 19. owner_dept 기준 회귀: 외부 end_user 호출이 owner 부서로 카운트 (actor 부서 아님)
- [ ] 20. "미배정" ≠ "외부 사용자" 라벨 분리 — 본 표엔 "외부" 라벨 미사용
- [ ] 21. PM 확인 대기 항목 명시 — api_call(nginx) 포함 여부 결정 전까지 표 캡션에 "콘솔/엔드유저 채널 한정" (선택)

## PM 확인 후보 (구현 전 결정 필요)

- [ ] **신규 App/KB/Tool "신규" 시점 정의** — `resource_ownership.created_at`(소유권 등록 시점) vs Dify 앱 생성 시점(`apps.created_at`). 현재 ownership 등록 기준으로 설계
- [ ] **api_call(nginx) 부서 분류 포함 여부** — 현재 audit_events 단계에서 부서 분류 불가(`details.targetAppId` 없음). nginx 별도 ingestion으로 보강할지, 본 표에선 제외할지 결정
- [ ] **헤더 정렬/필터** — 전역 표 헤더 정책 확정 후 적용 (Phase 1은 호출 수 디폴트 내림차순만)
- [ ] **선택 컬럼 후보** — 응답시간/만족도/TPS/앱타입 등은 audit 가용성 별도 검증 후. 본 Phase 1 범위 외

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/dept-activity.md|Design]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit details 매트릭스]]
