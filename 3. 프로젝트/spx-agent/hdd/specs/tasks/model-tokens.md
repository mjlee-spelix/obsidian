---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: 모델별 토큰 사용량
harness: [H-DASH-01, H-DASH-02, H-DASH-03, H-DASH-09, H-DASH-11, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
date: 2026-04-29
last_updated: 2026-05-13
---
# 모델별 토큰 사용량 — Tasks

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/requirements/model-tokens.md|Requirements]] · [[3. 프로젝트/spx-agent/hdd/specs/design/model-tokens.md|Design]]
> 순서: 프론트엔드(목업) → 백엔드 → 연동 → 테스트

## 0단계: 선행 조건 (collector 보강)

- [ ] 0-1. H-CAND-audit-appmode-missing P0 보강 — `messages.ts`/`workflow-runs.ts`/`workflow-nodes.ts`/`conversations.ts` collector가 `details.appMode` 채우는지 확인 (`references/audit-details-spec.md § 4`)
- [ ] 0-2. H-CAND-audit-wf-debug-filter-missing P0 보강 — `workflow-runs.ts` collector가 `details.triggeredFrom` 채우는지 확인
- [ ] 0-3. 보강 미완 시 백엔드 쿼리에 fallback 주석 + 미분류 합계가 디버깅 실행을 포함할 가능성 명시

## 1단계: 프론트엔드 (목업 데이터)

- [ ] 1. `web/app/components/admin/model-tokens-chart/index.tsx` — ECharts 수평 막대 차트 (고정 높이)
- [ ] 2. 목업 데이터로 차트 확인 (모델 5~6개 + "미분류 (Workflow)" 하드코딩)
- [ ] 3. 로컬 모델 "(로컬)" 라벨 + 색상 구분 (H-DASH-09)
- [ ] 4. 내림차순 정렬
- [ ] 5. 차트 영역 클릭 인터랙션 제거 (`silent: true` 등 — H-DASH-20 보류 반영)
- [ ] 6. 로딩/에러/데이터 없음 상태 처리

## 2단계: 백엔드

- [ ] 7. `audit_events` 모델 import 확인
- [ ] 8. `DashboardModelTokensService` 클래스 생성 (`api/services/admin/`)
  - [ ] 8-1. `get_message_send_tokens()` — `action='message_send'`, `details->>'invokeFrom' != 'debugger'`, GROUP BY `details->>'modelProvider'`, `details->>'modelId'` (H-DASH-03)
  - [ ] 8-2. `get_unclassified_workflow_tokens()` — `action='workflow_execute'`, `details->>'appMode' = 'workflow'`, `details->>'triggeredFrom' != 'debugging'` (H-DASH-01, H-DASH-02, H-DASH-03)
  - [ ] 8-3. 로컬 provider 판별 로직 — `LOCAL_PROVIDERS` set (H-DASH-09)
  - [ ] 8-4. display_name 생성 — 로컬이면 "(로컬)" 라벨 추가
  - [ ] 8-5. `total_tokens` = models 합계 + unclassified
- [ ] 9. API 엔드포인트 등록 (`GET /console/api/dashboard/model-tokens`, `api/controllers/console/dashboard/`)

## 3단계: 연동

- [ ] 10. `web/service/use-model-tokens.ts` 훅 작성 — queryKey `['dashboard', 'model-tokens', params]`, staleTime 5분
- [ ] 11. 목업을 실제 API로 교체, 페이지 헤더 새로고침이 `['dashboard']` invalidate로 동작하는지 확인
- [ ] 12. 실제 데이터로 차트 검증

## 4단계: 테스트

- [ ] 13. H-DASH-01: ADVANCED_CHAT 토큰이 `message_send`에서만 카운트되는지 (workflow_execute 미분류로 안 빠지는지)
- [ ] 14. H-DASH-02: WORKFLOW 모드 토큰이 "미분류"에 포함되는지
- [ ] 15. H-DASH-03: `invokeFrom='debugger'` 메시지가 제외되는지, `triggeredFrom='debugging'` 워크플로우가 제외되는지
- [ ] 16. H-DASH-09: 로컬 provider 모델에 "(로컬)" 라벨이 붙는지
- [ ] 17. 차트 합계 = `dept-activity` 테이블의 "토큰 사용" 컬럼 합계와 일치하는지

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/specs/requirements/model-tokens.md|Requirements]]
- [[3. 프로젝트/spx-agent/hdd/specs/design/model-tokens.md|Design]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit details 가용성 매트릭스]]
