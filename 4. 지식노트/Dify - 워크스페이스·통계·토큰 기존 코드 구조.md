---
tags: [dify, 개발, AI-Agent]
date: 2026-04-22
---
# Dify - 워크스페이스·통계·토큰 기존 코드 구조

## 핵심
- Dify 원본 코드에는 **워크스페이스 관리**, **앱별 통계**, **토큰 사용량 추적** 기능이 이미 존재
- **커뮤니티 버전은 다중 워크스페이스 생성 불가** — `ALLOW_CREATE_WORKSPACE = False`가 기본값이며, 워크스페이스 생성 API는 `@enterprise_inner_api_only`로 엔터프라이즈 전용. LICENSE 파일에도 "one tenant = one workspace" 기준 다중 테넌트 운영은 서면 허가 필요로 명시
- 따라서 **단일 워크스페이스 내에서** 앱/워크플로우/데이터셋을 부서·프로젝트별로 구분 관리하는 구조가 됨
- 통계는 모두 **앱 단위**로만 집계됨 → 부서/프로젝트별 합산 집계 레이어를 새로 만들어야 함

## 상세

### 1. 워크스페이스 (Tenant) 모델

**위치**: `api/models/account.py`

| 모델 | 역할 | 주요 필드 |
|------|------|-----------|
| `Tenant` | 워크스페이스 단위 | id, name, plan, status(NORMAL/ARCHIVE), custom_config, created_at |
| `TenantAccountJoin` | 사용자-워크스페이스 매핑 | tenant_id, account_id, role, current, invited_by |
| `Account` | 사용자 계정 | id, name, email, status, last_login_at, last_active_at |

**역할 체계** (`TenantAccountRole`):
- `OWNER` — 전체 권한, 소유권 이전 가능
- `ADMIN` — 관리 권한 (멤버 초대/삭제, 설정 변경)
- `EDITOR` — 앱/워크플로우 생성·편집
- `NORMAL` — 대화 조회 등 제한적 접근
- `DATASET_OPERATOR` — 데이터셋 관리 전용

### 2. 워크스페이스 API 엔드포인트

**위치**: `api/controllers/console/workspace/`

| 엔드포인트 | 메서드 | 기능 |
|-----------|--------|------|
| `/console/api/workspaces` | GET | 내 워크스페이스 목록 |
| `/console/api/all-workspaces` | GET | **전체 워크스페이스 목록** (어드민 전용) |
| `/console/api/workspaces/current` | POST | 현재 워크스페이스 정보 + 역할 |
| `/console/api/workspaces/switch` | POST | 워크스페이스 전환 |
| `/console/api/workspaces/info` | POST | 워크스페이스 이름 수정 |
| `/console/api/workspaces/current/members` | GET | 현재 워크스페이스 멤버 목록 |
| `/console/api/workspaces/current/members/invite-email` | POST | 멤버 초대 |

> [!tip] `/all-workspaces`가 이미 존재
> `@admin_required` 데코레이터로 보호되며, 페이지네이션을 지원한다. 중앙 관리 대시보드의 워크스페이스 목록 기반으로 활용 가능.

### 3. 앱 통계 API (앱별 일일 집계)

**위치**: `api/controllers/console/app/statistic.py`

모든 엔드포인트는 `start`, `end` 파라미터(YYYY-MM-DD HH:MM)로 기간 필터링 가능.

| 엔드포인트 | 반환 데이터 |
|-----------|------------|
| `/apps/{id}/statistics/daily-messages` | 일별 메시지 수 |
| `/apps/{id}/statistics/daily-conversations` | 일별 대화 수 |
| `/apps/{id}/statistics/daily-end-users` | 일별 고유 사용자 수 |
| `/apps/{id}/statistics/token-costs` | 일별 토큰 수 + 비용(USD) |
| `/apps/{id}/statistics/average-session-interactions` | 대화당 평균 메시지 수 |
| `/apps/{id}/statistics/user-satisfaction-rate` | 사용자 만족도 (좋아요 비율) |
| `/apps/{id}/statistics/average-response-time` | 평균 응답 시간(ms) |
| `/apps/{id}/statistics/tokens-per-second` | 토큰 생성 처리량(TPS) |

### 4. 워크플로우 통계 API

**위치**: `api/controllers/console/app/workflow_statistic.py`

| 엔드포인트 | 반환 데이터 |
|-----------|------------|
| `/apps/{id}/workflow/statistics/daily-conversations` | 일별 워크플로우 실행 수 |
| `/apps/{id}/workflow/statistics/daily-terminals` | 일별 고유 사용자 수 |
| `/apps/{id}/workflow/statistics/token-costs` | 일별 토큰 사용량 |
| `/apps/{id}/workflow/statistics/average-app-interactions` | 평균 실행 단계 수 |

### 5. 토큰 사용량 추적 구조

**메시지 레벨** (`api/models/model.py` — `Message` 모델):
```
message_tokens    → 입력(프롬프트) 토큰 수
answer_tokens     → 출력(응답) 토큰 수
message_unit_price / answer_unit_price → 토큰당 단가
total_price       → 총 비용
currency          → 통화 (기본 USD)
```

**워크플로우 레벨** (`api/models/workflow.py` — `WorkflowRun` 모델):
```
total_tokens      → 워크플로우 실행 전체 토큰 수
elapsed_time      → 소요 시간
total_steps       → 실행된 노드 수
```

**워크플로우 노드 레벨** (`WorkflowNodeExecutionModel`):
```
execution_metadata (JSON) → total_tokens, total_price, currency
```

**일일 집계 쿼리 패턴** (statistic.py 내부):
```sql
SELECT date, SUM(message_tokens + answer_tokens) AS token_count,
       SUM(total_price) AS total_price
FROM messages
WHERE app_id = :app_id AND invoke_from != 'debugger'
GROUP BY date
```

> [!important] 앱 단위로만 집계됨
> 현재 통계는 모두 개별 앱 ID 기준. 워크스페이스 전체나 조직 전체로 묶어 집계하는 API는 없다.

### 6. 빌링/크레딧 시스템

**위치**: `api/services/billing_service.py`, `feature_service.py`, `credit_pool_service.py`

| 서비스 | 주요 메서드 | 반환 |
|--------|-----------|------|
| `BillingService.get_info()` | 워크스페이스별 빌링 정보 | plan, members/apps 수·제한, 벡터 공간 사용량 |
| `BillingService.get_plan_bulk()` | 여러 워크스페이스 플랜 일괄 조회 | {tenant_id: plan} 딕셔너리 |
| `FeatureService.get_features()` | 기능 플래그 + 사용량 | 멤버·앱 제한, API rate limit, 크레딧 |
| `CreditPoolService.get_pool()` | 크레딧 잔액 조회 | quota_limit, quota_used, remaining |

**크레딧 풀 모델** (`TenantCreditPool`):
- `pool_type`: trial / paid
- `quota_limit`: 전체 한도
- `quota_used`: 사용량
- `remaining_credits`: 잔액 (계산 프로퍼티)

## 관련 노트
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
- [[2. 회의록/0414 AAI 데일리 스크럼.md]]
- [[4. 지식노트/Dify - 워크플로우 도구 버전 동기화.md]]
- [[4. 지식노트/Dify - DSL YAML 작성 및 관리.md]]
