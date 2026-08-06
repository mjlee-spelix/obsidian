---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: 모델별 토큰 사용량
harness: [H-DASH-01, H-DASH-02, H-DASH-03, H-DASH-09, H-DASH-11, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_image: "images/Dify/(Dify) 모니터링 - 챗봇.png"
date: 2026-04-29
last_updated: 2026-05-22
---
# 모델별 토큰 사용량 — Requirements

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/model-tokens.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/model-tokens.md|Tasks]]

## 화면 요구사항

수평 막대 차트. Y축 = 모델명, X축 = 토큰 사용량. 로컬 모델은 "(로컬)" 라벨 구분. 차트 하단에 합계 표시. 좌하 영역 — 클릭 인터랙션 없음 (차트만 표시).

```
┌─ 모델별 토큰 사용량 ─── 지난 30 일 ──── 합계: 28.4M ┐
│ GPT-4o          ████████████████  12.4M            │
│ Claude 3.5      ███████████        8.7M            │
│ Gemini 1.5      ██████             5.2M            │
│ Llama 3 (로컬)   ███                2.1M            │
│ 미분류 (Workflow) ██                 ?              │
└────────────────────────────────────────────────────┘
```

| 구성요소 | 설명 |
|---------|------|
| 수평 막대 | 모델별 토큰 합산 (내림차순 정렬) |
| 정확값 라벨 | 각 막대 우측에 토큰 수 표시 (K/M 압축, hover 시 raw 정확값) |
| 합계 표시 | **카드 헤더 우측** — `합계: {formatCount(total_tokens)}` (단위 텍스트 "tokens" 미부착). 차트 본문 hover 툴팁에는 `토큰: {raw}` 형태로 단위 노출 |
| 로컬 라벨 | 로컬 provider 모델에 "(로컬)" 표시 (H-DASH-09) |
| "미분류 (Workflow)" 막대 | WORKFLOW 모드 토큰 — 모델 분리 불가 (H-DASH-02) |
| 색상 | 모델별 색상 구분 — Dify 디자인 토큰 팔레트 사용 (`util-colors-*`). 우선 단색 + 로컬 모델만 다른 색 |
| 차트 영역 높이 | 고정 — 내용 초과 시 영역 내부 스크롤 (전역 레이아웃 정책) |
| 클릭 인터랙션 | 없음 — 차트 막대/축/하단 합계 모두 클릭 동작 없음 (차트 드로어 보류, H-DASH-20) |

## 기간/필터

- 페이지 헤더의 `DashboardControls` 기간 선택기에서 받은 `period` prop 사용 — URL query string이 단일 진실 (`?start=...&end=...`)
- 페이지 헤더 새로고침 버튼은 `['dashboard']` prefix를 일괄 invalidate (`['dashboard', 'model-tokens', params]` 포함)

## 데이터 소스

**`audit_events` 단일 SoT** — 마트 입력은 audit_events에서 추출 (2026-05-12 결정). messages/workflow_runs OLTP 테이블 직접 조회 금지.

사용 action: `message_send`, `workflow_execute`.

| 키 | 출처 action | 가용성 | 비고 |
|----|------------|--------|------|
| `details->>'modelProvider'` | `message_send`만 | ✅ | workflow_execute에는 없음 → "미분류" 버킷 (H-DASH-02) |
| `details->>'modelId'` | `message_send`만 | ✅ | 동상 |
| `details->>'totalTokens'` | `message_send` + `workflow_execute` | ✅ | 두 action 모두 보유 |
| `details->>'appMode'` | 모든 action | 🔴 **P0 보강 후** | collector가 `app.mode`를 details에 넣어야 함 (H-CAND-audit-appmode-missing) |
| `details->>'invokeFrom'` | `message_send`만 | ✅ | 디버깅 필터 |
| `details->>'triggeredFrom'` | `workflow_execute`는 🔴 **P0 보강 후** | 부분 | workflow_runs collector가 SELECT 안 함 (H-CAND-audit-wf-debug-filter-missing) |
| `occurred_at` (top-level) | 모든 action | ✅ | 기간 필터 |
| `actor_type` (top-level) | 모든 action | ✅ | end_user/account 구분 — 본 차트는 미사용 |

> 상세 가용성 매트릭스: `references/audit-details-spec.md § 3`

## 비즈니스 규칙

**CHAT 계열 모델별 토큰 (`message_send` action 기반):**
- `details->>'modelProvider'`, `details->>'modelId'`로 GROUP BY
- AppMode 분기: `details->>'appMode'`로 ADVANCED_CHAT 식별 → `message_send`에서만 카운트 (H-DASH-01)
  - WORKFLOW 모드의 `message_send`는 존재하지 않음 (Dify 데이터 모델상 WORKFLOW는 messages 미생성)
- 디버깅 실행 제외: `details->>'invokeFrom' != 'debugger'` (H-DASH-03)

**WORKFLOW 토큰 ("미분류"):**
- `workflow_execute` action의 `details->>'totalTokens'` SUM
- AppMode 분기: `details->>'appMode' = 'workflow'`로 WORKFLOW만 필터 — ADVANCED_CHAT의 workflow_execute는 H-DASH-01에 따라 제외
- `workflow_execute`에는 `modelProvider`/`modelId` 없음 → 모델별 분리 불가 (H-DASH-02)
- 차트에 "미분류 (Workflow)" 막대로 합산 표시
- 디버깅 실행 제외: `details->>'triggeredFrom' != 'debugging'` (H-DASH-03 — P0 collector 보강 후 가용. 보강 전에는 PM 확인 후 보강 일정 결정)
- P1 선택 보강(`workflow_node_executions.process_data` JSON 파싱): 현재 범위 외 — "미분류" 버킷 유지가 1차 방어 (H-DASH-02 기존 방어 유지)

**로컬 모델 구분:**
- 로컬 provider 목록: `ollama`, `xinference`, `localai` 등
- 해당 provider의 모델명 뒤에 "(로컬)" 라벨 추가 (H-DASH-09)
- 로컬 provider 목록은 설정으로 관리하거나 Dify 모델 provider 유형 활용

**데이터 정합성:**
- KPI 카드 4종에 "토큰 사용" 카드 없음 → KPI 카드와의 직접 정합성 검증 대상 아님
- `dept-activity` 테이블의 "토큰 사용" 컬럼 합계와 일치 (동일 기간 + 동일 H-DASH 필터)

## 사용자 범위

- 로그인 사용자 전체 (관리자 전용 폐기). `dataset_operator` 권한도 접근 허용 — `RoleRouteGuard` 추가 없음.

## 방어할 Harness

| ID | 결함 | 이 화면에서의 방어 |
|----|------|---------------|
| H-DASH-01 | ADVANCED_CHAT 이중카운트 | `details->>'appMode'`로 분기 — ADVANCED_CHAT은 `message_send`만 |
| H-DASH-02 | WORKFLOW 모델별 분리 불가 | **핵심 함정** — "미분류 (Workflow)" 버킷으로 합산 |
| H-DASH-03 | 디버깅 혼입 | `details->>'invokeFrom' != 'debugger'` (messages) + `details->>'triggeredFrom' != 'debugging'` (workflow, P0 보강 후) |
| H-DASH-09 | 로컬 모델 구분 | provider 목록 기반 "(로컬)" 라벨 |
| H-DASH-11 | JSON 파싱 성능 | P1 선택 보강 시 Redis 캐시 필수 |
| H-CAND-audit-appmode-missing | audit details에 appMode 누락 | P0 collector 보강 필요 — 보강 전 H-DASH-01 방어 불가 |
| H-CAND-audit-wf-debug-filter-missing | workflow_execute details에 triggeredFrom 누락 | P0 collector 보강 필요 — 보강 전 workflow 디버깅 필터 불가 |

## PM 확인 후보

- 없음 — 본 컴포넌트의 모든 변수는 collector 보강 일정에 종속

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit details 가용성 매트릭스]]
- [[3. 프로젝트/spx-agent/references/dify-app-modes.md|AppMode 규칙]]
- [[4. 지식노트/Dify - workflow_node_executions에서 모델별 토큰 추출.md]]
