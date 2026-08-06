---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: 부서별 오브젝트 분포
harness: [H-DASH-04, H-DASH-13, H-DASH-20]
date: 2026-04-29
last_updated: 2026-05-13
---
# 부서별 오브젝트 분포 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/dept-objects.md|Design]]
> 순서: 프론트엔드(목업) → 백엔드 → 연동 → 테스트

## 1단계: 프론트엔드 (목업 데이터)

- [ ] 1. `web/app/components/admin/dept-objects/dept-objects-chart.tsx` — ECharts 수평 스택 막대 (App/KB/Tool 3색)
- [ ] 2. 목업 데이터로 차트 확인 (부서 3~4개 + "미배정" 행 하드코딩)
- [ ] 3. "미배정" 막대 별도 스타일 (점선 테두리 또는 라벨 강조)
- [ ] 4. 로딩 / 에러 / 데이터 없음 상태 처리
- [ ] 5. 카드 컨테이너 **고정 높이 + 내부 스크롤** 적용 (전역 레이아웃 정책)
- [ ] 6. 막대 클릭 핸들러는 **추가하지 않음** (차트 드로어 보류 — H-DASH-20)

## 2단계: 백엔드

- [ ] 7. SQLAlchemy 모델 import 확인 (`Department`, `ResourceOwnership`, `Account`, `DepartmentMember` + Dify의 `App`, `Dataset`, `ToolProvider`) — H-DASH-13
- [ ] 8. `DashboardDeptObjectsService` 클래스 생성
  - [ ] 8-1. `get_dept_objects(tenant_id)` — **`apps LEFT JOIN spx_resource_ownership` 패턴** + datasets/tool_providers UNION ALL (H-DASH-04 강화 — 레거시 누락 0건)
  - [ ] 8-2. COALESCE 미배정 fallback 및 부서별 피봇 (resource_type → app_count/kb_count/tool_count)
- [ ] 9. API 엔드포인트 등록 — `GET /console/api/dashboard/dept-objects` (admin prefix 폐기, 로그인 사용자 전체 허용)

## 3단계: 연동

- [ ] 10. `useDeptObjects` 훅 — `queryKey: ['dashboard', 'dept-objects']`, `staleTime: 5분`
- [ ] 11. 목업 → 실제 API 교체 후 차트 검증
- [ ] 12. 페이지 헤더의 새로고침 버튼이 `['dashboard']` prefix invalidate로 본 차트도 갱신되는지 확인

## 4단계: 테스트

- [ ] 13. H-DASH-04: `resource_ownership`에 행이 없는 레거시 앱이 "미배정"에 포함되는지 (mock에서 의도적으로 한 건 비워서 검증)
- [ ] 14. 차트 합계 = KPI 카드 "총 오브젝트" 수치와 일치하는지
- [ ] 15. 부서가 0개일 때 (데이터 없음) 차트가 깨지지 않는지
- [ ] 16. 부서 수가 임계치를 넘었을 때 카드 영역 내부에서 스크롤되는지 (고정 높이 유지 확인)

## 보조 뷰 (선택 — 필요 시 별도 위젯/엔드포인트로 추가)

- [ ] (옵션) top-owners: `owner_account_id` 기준 Top N 표 — `created_by` 사용 금지
- [ ] (옵션) dept-new-creations: `resource_ownership.created_at` 기반 기간별 신규
  - PM 확인 필요: `created_at` 의미 = 소유권 등록 시점(기본) vs 앱 생성 시점(apps JOIN 필요)

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/dept-objects.md|Design]]
- [[3. 프로젝트/spx-agent/references/objects-charts-feasibility.md|objects-charts-feasibility.md]]
