---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: 부서별 활동 테이블
harness: [H-DASH-01, H-DASH-03, H-DASH-04, H-DASH-10, H-DASH-13, H-CAND-audit-appmode-missing, H-CAND-audit-wf-debug-filter-missing]
design_image: "images/설계/(화면 설계) 대시보드.png"
reference_image: "images/Dify/(Dify) 모니터링 - 챗봇.png"
date: 2026-04-29
last_updated: 2026-05-22
---
# 부서별 활동 테이블 — Requirements

> 관련: [[3. 프로젝트/spx-agent/hdd/specs/design/dept-activity.md|Design]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dept-activity.md|Tasks]]

## 화면 요구사항

테이블 형태. 페이지 헤더 슬롯에 `dashboard-controls`가 별도로 존재하므로 본 표는 데이터 영역만 담당.

```
┌─ 부서별 신규 생성 — 지난 30 일 ──────────────────────── ┐
│ 부서       앱    지식   도구   신규                      │
│ IT 본부    +3    +2     +1     +6                        │
│ 마케팅팀   +1    +0     +0     +1                        │
│ 미배정     -     -      -      -                         │
│ 인사팀     +2    +0     +1     +3                        │
│ 재무팀     +0    +1     +0     +1                        │
└──────────────────────────────────────────────────────────┘
```

> 5/22 코드 정합: 종합 뷰 하단 표는 KPI 1(`objects`) 메트릭의 드릴스루 표 `dept-new-creations-table`을 기본 표시로 사용. KPI 2·3·4 메트릭 활성화 시 해당 메트릭의 드릴스루 표로 교체 ([[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-drill-through.md|kpi-drill-through requirements § 메트릭 카탈로그]] 참조).

| 컬럼 | 설명 |
|------|------|
| 부서 | `departments.name` (RBAC) — 미배정 포함. **부서명 가나다순 기본 정렬** (`localeCompare('ko-KR')`) |
| 앱 | 기간 내 해당 부서에 소유권 등록된 App 수 (`resource_ownership`, `resource_type='app'`, `created_at BETWEEN :start AND :end`). 0이면 `-`. i18n: `common.objectType.app` |
| 지식 | 동일 패턴, `resource_type='dataset'`. i18n: `common.objectType.kb` |
| 도구 | 동일 패턴, `resource_type='tool'`. i18n: `common.objectType.tool` |
| 신규 | 3종 합계 (강조 색상). 0이면 `-` |

> 호출 수 / 토큰 사용 컬럼은 5/22 코드 기준 부서별 신규 생성 표에서 제거됨 (호출 정보는 `dept-call-rps-table`로, 토큰 정보는 `model-tokens` 차트로 분리). 종합 뷰에서 동일 화면에 두는 정보 위계는 KPI 카드(상단) + 부서별 오브젝트 차트(좌) + 모델별 토큰 차트(우) + 부서별 신규 생성 표(하단) 구성.

> ⚠️ **PM 확인 항목**: 신규 App/KB/Tool의 "신규" 정의 — 현재 `resource_ownership.created_at`(소유권 등록 시점) 기준. Dify 앱 생성 시점(`apps.created_at`) 기준인지 PM 확정 필요.

## 기간/필터

- `dashboard-controls`의 기간 선택기는 URL query string 단일 진실. 본 컴포넌트는 query string에서 `start`/`end`를 직접 읽어 API에 전달
- 기본 표시: 전체 부서 (페이지네이션 없음 — Phase 1 단순 전체 표시). 부서 수 폭증 시 페이지네이션 도입 검토
- 정렬 디폴트: **부서명 가나다순** (`localeCompare('ko-KR')`) — 부서 그레인 표 4종 공통 컨벤션 (5/22 코드 정합)

## 표시 규칙

> 📌 본 표(전체 부서 전수 + 0이면 `-` + 골격 유지)는 `conventions.md § 빈 상태 / 골격 / 결측치 정책`의 **검증된 레퍼런스 예시**다. 다른 표/차트는 이 패턴을 따른다.

- **제목 옆 기간 라벨**: "부서별 활동 — 지난 30 일" 형식 (KPI 카드 패턴과 동일)
- **숫자 포맷 (호출/토큰)**: 전역 규칙 — `<10K` raw + 콤마, `≥10K` K/M 압축 + hover 정확값 (`hdd/design.md § 6.5`)
- **신규 App/KB/Tool**: raw 숫자 표시 (소규모 카운트라 K/M 압축 불필요). 0이면 `-`
- **0 값 처리**: 호출/토큰 0은 회색 dash(`-`). 신규 0도 `-`. 부서 자체가 활동 없으면 행은 표시하되 dash 컬럼
- **"미배정" 행**: `owner_department_id IS NULL` 앱의 활동 집계. 항상 가장 하단(소속 부서가 아님)

## 인터랙션

- **표 행 클릭 없음** (전역 정책 — 표 행 클릭은 KPI 4번 인기 호출 앱만, `conventions.md § 인터랙션 정책`)
- **헤더 정렬/필터**: 후보 — 전역 표 헤더 정책 확정 후 적용 (Phase 1에서는 호출 수 디폴트 내림차순만)

## 비즈니스 규칙

**부서 기준 (단일 기준):**
- 본 표의 모든 활동 컬럼은 **앱 소유 부서**(`resource_ownership.owner_department_id`) 기준
- "어떤 부서가 가진 앱이 이만큼의 활동을 받았다" 의 의미
- ~~actor 기준 토글~~ 폐기 — 본 표는 owner 단일 기준. 사용자(actor) 차원이 필요하면 별도 spec(향후 dept-user-activity)으로 분리

**데이터 소스 (마트 입력 단일 SoT):**
- `audit_events` 단일 SoT (2026-05-12 이사님 결정, `hdd/design.md § 마트 입력`)
- RBAC `resource_ownership` LEFT JOIN으로 부서 부여
- ~~`messages` / `workflow_runs` 직접 조회~~ 폐기 (오브젝트 차트만 `resource_ownership` 직접 조회)

**신규 App / KB / Tool (resource_ownership 직접 조회):**
- 데이터 소스: `resource_ownership` 직접 (state 본질 — audit 아님)
- SQL 패턴: `COUNT(*) FILTER (WHERE ro.resource_type = 'app' AND ro.created_at BETWEEN :start AND :end)` per resource_type
- `resource_type` 값: `'app'` / `'dataset'`(KB) / `'tool'`
- 부서 기준: `ro.owner_department_id`
- ⚠️ PM 확인 항목: "신규"의 시점 정의 — `resource_ownership.created_at`(소유권 등록 시점) vs Dify 앱 생성 시점(`apps.created_at`). 현재 ownership 등록 시점 기준으로 설계

**호출 수 / 토큰 집계 이벤트:**
- `event_type IN ('message_send', 'workflow_execute')`
- 호출 수: COUNT(*)
- 토큰: `SUM(details->>'totalTokens'::int)` — 둘 다 collector가 totalTokens 적재
- ADVANCED_CHAT 이중카운트 방어 (H-DASH-01): audit는 `app_create` 이외 collector가 `details.appMode`를 적재하지 않음(H-CAND-audit-appmode-missing). collector 보강 전까지는 `message_send`와 `workflow_execute`를 단순 합산하면 ADVANCED_CHAT이 양쪽에 모두 잡힐 수 있어, **collector P0 보강(appMode 추가) 완료 후 운영 마트 ETL 본가동**. 그 전 임시 단계는 OLTP 직접 쿼리(`apps.mode` JOIN)로 우회

**디버깅 필터 (H-DASH-03):**
- `message_send`: `details->>'invokeFrom' != 'debugger'`
- `workflow_execute`: `details->>'triggeredFrom' != 'debugging'` — collector 보강 전까지 적재 안 됨(H-CAND-audit-wf-debug-filter-missing). 보강 후 본가동

**api_call (nginx, REST API 외부 호출):**
- `details.targetAppId` 없음 → audit_events 단계에서는 부서 분류 불가
- ⚠️ **PM 확인 항목**: 부서별 활동 표에 nginx API 호출을 포함할지 여부 결정 필요. 포함 결정 시 nginx 로그/별도 ingestion에서 `app_id` 보강 후 audit로 합류 필요. 미포함이면 표 캡션에 "콘솔/엔드유저 채널 한정" 명시

**미배정 처리 (H-DASH-04):**
- `resource_ownership`에 등록 안 된 앱(`owner_department_id IS NULL`)의 활동은 "미배정" 행으로 집계
- ⚠️ "미배정" ≠ "외부 사용자". 본 표는 owner 기준이라 actor 차원 "외부" 라벨 미사용

**외부 사용자 식별:**
- audit_events top-level `actor_type = 'end_user'` 컬럼으로 구분 가능 (`references/audit-details-spec.md § 5`)
- Phase 1 본 표에는 컬럼 없음 — drill-through에서만 사용

## 마운트 환경

- 라우트: `/dashboard` 톱레벨 페이지의 우측 하단 영역 (H-DASH-19로 모달 마운트 제약 폐기)
- 페이지 헤더 슬롯의 `dashboard-controls`가 기간/새로고침 제공 → 본 표는 자체 컨트롤 없음
- 표시 높이: **고정 높이 + 헤더 sticky + 본문 내부 스크롤** (2026-05-13 전역 레이아웃 정책, `conventions.md § 화면 레이아웃 정책`)
- 부서 수가 컨테이너 높이를 초과해도 카드 외곽 크기 변동 금지

## 방어할 Harness

| ID                                   | 결함                                 | 이 화면에서의 방어                                                                          |
| ------------------------------------ | ---------------------------------- | ----------------------------------------------------------------------------------- |
| H-DASH-01                            | ADVANCED_CHAT 이중카운트                | collector appMode 보강 후 audit ETL에서 분기. 보강 전 임시 OLTP 우회 시 `apps.mode` JOIN           |
| H-DASH-03                            | 디버깅 혼입                             | `details->>'invokeFrom' != 'debugger'` + `details->>'triggeredFrom' != 'debugging'` |
| H-DASH-04                            | 레거시 앱 미배정                          | LEFT JOIN + "미배정" 행 fallback                                                        |
| H-DASH-10                            | 다단 JOIN 성능                         | audit 단일 SoT로 다단 감소. CTE 분리 + Redis 캐시 5분 + 인덱스                                     |
| H-DASH-13                            | RBAC 스키마 변경                        | SQLAlchemy 모델(`Department`, `ResourceOwnership`) 경유                                 |
| H-CAND-audit-appmode-missing         | audit collector가 appMode 미적재       | collector P0 보강 의존 — 보강 완료 전 마트 ETL 본가동 보류 / 임시 OLTP 우회                             |
| H-CAND-audit-wf-debug-filter-missing | workflow_execute에 triggeredFrom 누락 | collector P0 보강 의존 — 보강 전 workflow 디버깅 혼입 위험 명시                                     |

## 관련 노트

- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]]
- [[3. 프로젝트/spx-agent/references/dify-db-schema.md|Dify DB 스키마]]
- [[3. 프로젝트/spx-agent/references/dify-app-modes.md|AppMode 규칙]]
- [[3. 프로젝트/spx-agent/references/rbac-schema.md|RBAC 스키마]]
- [[3. 프로젝트/spx-agent/references/audit-details-spec.md|audit details 매트릭스]]
