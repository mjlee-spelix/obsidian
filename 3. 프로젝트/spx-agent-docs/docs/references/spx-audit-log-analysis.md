---
title: 감사 로그 신규 챕터 — dify-audit 분석
phase: Phase 4 / 클러스터 B / B1
status: 완료 (2026-06-04)
audience: spx-agent 사용자 매뉴얼 작성자 (Phase 4 챕터 집필자)
purpose: |
  spx-agent의 감사 로그 기능(dify-audit 별도 앱)을 사용자 매뉴얼 관점으로 정리.
  무엇이 기록되는가(이벤트 카탈로그) / 어디서 보는가(진입점·탭·필터) /
  어떻게 내보내는가(Export) / 누가 볼 수 있는가(권한)를 박제한다.
  클러스터 B의 첫 산출물 — B2(Monitor Analysis)가 본 문서를 3축 비교의 한 축으로 인용.
source_root: C:\Users\Administrator\Projects\spx-agent\dify-audit
embedded_in: web/app/components/header/account-setting/audit-log-page/index.tsx (iframe)
code_verified_at: 2026-06-04
---

# 1. 본 문서의 위치

> ⚠️ **본 분석본은 이벤트 카탈로그·UI·권한 구조 박제용(구조 나열)입니다. 본문은 이 구조를 그대로 옮기지 마시기 바랍니다.** 각 로그 항목(시각·행위자·액션·대상 등 컬럼)·필터·내보내기마다 "①무엇을 뜻하는가(의미) → ②어떤 질문에 답하나/왜 중요한가 → ③업무에 어떻게 활용하나(감사 추적·보안 점검 시나리오)" 3단으로 서술합니다. → [[conventions#데이터 조회·시각화 챕터 서술 원칙 (대시보드·감사로그·모니터링)]]

본 문서는 **사용자 매뉴얼 집필을 위한 분석 노트**다. 감사 로그는 spx-agent 본체(Dify fork)가 아니라 **`dify-audit`라는 별도 Next.js 16 앱**으로 구현되어 있고, Dify 설정 화면 안에 **iframe으로 임베드**된다. 본 문서는 다음을 정전화한다:

1. **아키텍처** — 왜 별도 앱인가, 데이터가 어디서 어떻게 모이는가
2. **이벤트 카탈로그** — 무엇이 기록되는가(action·category·source 전수)
3. **화면·필터·내보내기** — 사용자가 실제로 보는 UI
4. **권한·격리** — 누가, 어느 워크스페이스 범위까지 볼 수 있는가

> **클러스터 B 진행 순서**: B1(본 문서) → B2 [[references/spx-monitoring-analysis]]. B2는 "워크스페이스 KPI(대시보드) vs 앱 단위(모니터링) vs 이벤트(감사 로그)" **3축 비교표**에서 본 문서의 §3 카탈로그와 §8 영역 구분을 인용한다.

> **인접 분석**: 워크스페이스 KPI는 [[references/spx-dashboard-analysis]], 권한 모델 코어는 [[references/spx-app-permissions-analysis]](owner/admin 판정·테넌트 격리 근거), 부서/멤버 운영은 [[references/spx-departments-management]].

> **배경 티켓**: KAN-28. 챕터 추정 분량 2~3p, 우선순위 P1.

## 1.1 표기 가이드 — KC 추상화 (전역 규칙 #6)

> 2026-06-04 결정 5 박제. 정전 가이드는 A3 [[references/spx-departments-management#0-표기-가이드-—-kc-추상화-전역-규칙-6]]. 본 문서는 이를 따른다.

- **챕터 본문(mdx)**: "Keycloak" 직접 노출 금지. **"외부 시스템(예, Keycloak)"** 또는 **"관리 시스템"**으로 표기(IdP 중립성 — 사용 기업마다 IdP가 다를 수 있음).
- **본 분석본(reference)**: 코드와 직결되는 정전이라 **코드 식별자·메타 정보는 Keycloak 그대로 보존**한다(추상화 예외):
  - DB·컨테이너·파일·컬럼·함수명(`keycloakDb`, `user_entity`, `keycloak` 컨테이너, `verify-token.ts` 등)
  - 인증 메커니즘 식별이 필요한 부분(JWT `sub`·JWKS 검증·SSO 토큰 흐름)
- 단, **사용자에게 보이는 동작을 묘사하는 문장**(§4.1 "사용자" 열·검색 절)은 챕터 인용 시 추상화됨을 본문에 환기한다.

---

# 2. 아키텍처 — 사용자에게 설명할 수준

## 2.1 별도 앱 + iframe 임베드

| 항목 | 내용 |
|------|------|
| 형태 | `dify-audit` — 독립 Next.js 16 앱 (App Router, server component) |
| 노출 위치 | **설정 → 감사 로그** 탭. iframe `src="/audit"`, 같은 origin이라 Dify의 `access_token` 쿠키가 그대로 전달됨 |
| 진입점 코드 | `web/.../account-setting/audit-log-page/index.tsx` (단순 iframe 래퍼) |
| 탭 노출 조건 | **워크스페이스 owner/admin만** (`isCurrentWorkspaceManager`). 일반/편집자에겐 메뉴 자체가 안 보임 |
| 메뉴 라벨 | ko: **"감사 로그"** / en: "Audit Log" (`settings.auditLog`, 아이콘 `i-ri-file-list-3`) |

> 매뉴얼 작성 시: 감사 로그는 "별도 시스템"이 아니라 **설정 화면의 한 탭**으로 보이게 의도됐다. 사용자는 좌측 메뉴 → 감사 로그로 진입하면 됨. 직접 URL(`/audit`) 진입은 차단된다(§7.3).

## 2.2 3개 데이터베이스를 본다

dify-audit은 자기 DB 외에 Dify·Keycloak DB를 **읽기 전용**으로 참조한다.

| DB | 역할 | 접근 |
|-----|------|------|
| **audit** (`spx_audit_events` 등) | 수집된 이벤트 저장소 (자체 소유) | 읽기/쓰기 |
| **dify** | 운영 활동 원천 — 앱·대화·워크플로우·멤버 등 폴링 대상 | 읽기 전용 |
| **keycloak** | 행위자(actor) 이름·이메일 보강(enrichment)용 | 읽기 전용 |

> 핵심: 감사 로그는 Dify가 능동적으로 "찍는" 로그가 아니라, **dify-audit이 Dify DB를 주기적으로 들여다보며 사후 재구성**하는 구조다. 이 차이가 §5(수집 주기·지연)의 사용자 안내로 이어진다.

## 2.3 통합 이벤트 테이블 — `spx_audit_events`

모든 종류의 이벤트를 **단일 테이블**에 저장해 통합 검색/필터를 단순화한다. 사용자 화면의 모든 행이 이 한 테이블의 row다.

| 컬럼 | 의미 | 화면 표기 |
|------|------|----------|
| `occurredAt` | 발생 시각 | "시간" |
| `category` | `admin` / `user` / `security` | "카테고리" 배지 |
| `action` | 이벤트 종류 (§3 카탈로그) | "액션" |
| `actorType` | `account`(관리자) / `end_user`(앱 사용자) / `api` / `system` | 상세 "유형" |
| `actorId` / `actorEmail` | 행위자. 이메일은 대부분 NULL → Keycloak에서 이름 보강 | "사용자" |
| `targetType` / `targetId` / `targetName` | 대상 리소스 | "대상" |
| `ipAddress` / `userAgent` | 컨텍스트 | 상세에서만 |
| `status` | 결과 (success/failed/running 등, §6에서 정규화) | "결과" |
| `details` | 액션별 가변 JSON | "비고"(요약)·상세(전체) |
| `tenantId` | 워크스페이스. **NULL = 글로벌**(보안·시스템 이벤트) | (격리 판정용, §7) |
| `source` | 수집 경로 (§2.4) | 상세 "수집 출처" |

> `details` 외에 대시보드(B2) 집계용 **생성 컬럼**(app_mode_d, model_provider_d, total_tokens_d 등)과 materialized view 3종(`spx_mv_audit_enriched` / `spx_mv_kpi_calls_daily` / `spx_mv_model_tokens_daily`)이 있다. 매뉴얼 대상 아님 — 대시보드 챕터와의 연결고리로만 기억.

## 2.4 수집 경로(source) 5종 — "어떻게 모이는가"

감사 로그는 **5개의 서로 다른 경로**로 채워진다. 사용자에겐 "기록 방식이 여러 가지라 일부는 실시간, 일부는 지연/일배치"라는 점만 전달하면 된다.

| source 값 | 경로 | 방식·주기 | 담당 이벤트 |
|-----------|------|----------|------------|
| `dify_db` | **DB 폴링 수집기 13종** | 5분마다 cron (앱 기동 60초 후 1회 + 이후 `*/5`) | 생성·수정·실행·대화·메시지 등 (§3.1) |
| `pg_trigger` | **PostgreSQL 트리거** | Dify DB의 DELETE/역할변경 시 **즉시** | 삭제·역할 변경 6종 (§3.2) — 폴링으론 흔적이 안 남아 트리거가 유일 |
| `nginx_log` | **nginx 액세스 로그 와처** | 파일 변경 실시간 감시(chokidar) | 인증 실패·rate limit·API 호출 3종 (§3.3) |
| `dify_audit_app` | **self-audit** | 감사 로그 화면 사용 시 즉시 | 로그인·조회·내보내기 등 6종 (§3.4) |
| (별도 테이블) | **시스템 로그 일배치** | 매일 새벽 1시, 어제치 컨테이너 stdout | "시스템 로그" 탭 (§4.2) — `spx_audit_events`가 아니라 `spx_system_logs` |

---

# 3. 이벤트 카탈로그 — 무엇이 기록되는가

action·라벨은 코드의 `src/lib/audit-meta.ts`(한국어 라벨 사전)에서 그대로 가져왔다. **사용자 화면에 노출되는 한국어 라벨이 곧 매뉴얼 표기**다.

## 3.1 DB 폴링 수집기 13종 (source=`dify_db`)

각 수집기는 Dify의 특정 테이블을 cursor(시간) 기준으로 폴링한다. 행위자 이메일은 Dify가 Keycloak SSO를 쓰므로 대부분 NULL이고, 화면 표시 시 Keycloak에서 이름을 채운다.

| # | 수집기 | 원천 테이블 | action (라벨) | category | 비고 |
|---|--------|------------|--------------|----------|------|
| 1 | prompt_changes | `app_model_configs` | `prompt_update` (프롬프트 수정) | admin | 프롬프트 길이·미리보기 200자 |
| 2 | api_tokens | `api_tokens` | `api_token_create` (API 토큰 발급) | admin | actorId 없음(생성자 미기록) |
| 3 | dataset_changes | `datasets` | `dataset_create` (데이터셋 생성) | admin | 인덱싱 방식 |
| 4 | document_changes | `documents` | `document_upload` (문서 업로드) | admin | 인덱싱 실패 시 status=failed |
| 5 | conversations | `conversations` | `conversation_start` (대화 시작) | **user** | actor가 앱 사용자면 end_user |
| 6 | workflow_runs | `workflow_runs` | `workflow_execute` (워크플로우 실행) | **user** | 소요시간·토큰·에러 |
| 7 | messages | `messages` | `message_send` (메시지 송수신) | **user** | 질문·답변·모델·토큰·비용·지연 |
| 8 | app_changes | `apps` | `app_create` (앱 생성) / `app_update` (앱 수정) | admin | create/update 분리(§3.6) |
| 9 | workflow_publishes | `workflows` | `workflow_publish` (발행) / `workflow_draft` (초안 저장) | admin | version='draft' 여부로 분기 |
| 10 | members | `tenant_account_joins` | `member_join` (멤버 가입) / `member_change` (멤버 변경) | admin | create/update 분리 |
| 11 | workflow_nodes | `workflow_node_executions` | `workflow_node_execute` (노드 실행) | **user** | 노드 단위 — 볼륨 큼(§3.6) |
| 12 | message_feedbacks | `message_feedbacks` | `message_feedback` (답변 피드백) | **user** | 👍/👎 + 내용, status=rating |
| 13 | provider_changes | `provider_models` | `provider_model_add` (모델 추가) / `provider_model_update` (모델 변경) | admin | actorType=system |

## 3.2 PostgreSQL 트리거 (source=`pg_trigger`) — 삭제·역할 변경 6종

폴링은 INSERT/UPDATE만 잡는다. **DELETE는 row가 사라져 흔적이 안 남으므로** Dify 테이블에 AFTER 트리거를 걸어 즉시 기록한다. 모두 category=admin, actorType=account.

| 트리거 테이블 | action (라벨) | 트리거 조건 | details |
|--------------|--------------|------------|---------|
| `documents` | `document_delete` (문서 삭제) | DELETE | 문서명·datasetId·토큰 |
| `datasets` | `dataset_delete` (데이터셋 삭제) | DELETE | 이름·설명·인덱싱 방식 |
| `apps` | `app_delete` (앱 삭제) | DELETE | 앱명·모드·설명 |
| `api_tokens` | `api_token_delete` (API 토큰 삭제) | DELETE | 토큰타입·appId |
| `tenant_account_joins` | `member_remove` (멤버 제거) | DELETE | 역할·테넌트 |
| `tenant_account_joins` | `member_role_change` (멤버 역할 변경) | UPDATE 시 role 변경 | oldRole→newRole |

> 트리거는 `SECURITY DEFINER` + 예외 swallow로 설계 — **감사 기록 실패가 Dify 본 트랜잭션(삭제 자체)을 막지 않는다.** 즉 "삭제는 됐는데 로그가 빠질 수 있다"가 아니라, 정상 동작 시 항상 남고 장애 시에만 누락. 매뉴얼엔 "삭제 행위도 추적된다"로 충분.

## 3.3 nginx 로그 와처 (source=`nginx_log`) — 3종

nginx 액세스 로그를 실시간 감시해 HTTP 상태/경로로 분류한다. `/_next/`·`/static/`·favicon은 무시.

| 조건 | action (라벨) | category |
|------|--------------|----------|
| HTTP 401 / 403 | `auth_failed` (인증 실패) | **security** |
| HTTP 429 | `rate_limit_exceeded` (Rate Limit 초과) | **security** |
| `/v1/chat-messages`·`/v1/completion-messages`·`/v1/workflows/run` | `api_call` (API 호출) | **user** |

> actorType=api, 행위자는 IP·User-Agent로만 식별(사용자 매핑 없음). `NGINX_LOG_PATH` 미설정이면 이 경로는 동작 안 함 → 보안 이벤트가 안 쌓이는 환경이 있을 수 있음(배포 의존). 매뉴얼엔 단정적 약속을 피하고 "nginx 연동 시" 톤 권장.

## 3.4 self-audit (source=`dify_audit_app`) — 감사 로그 자체 사용 6종

감사 로그를 **누가 열람·내보냈는지**도 기록한다(감사자의 감사). 모두 category=admin, actorType=account.

| action | 라벨 | 발생 시점 |
|--------|------|----------|
| `audit_login` | audit 로그인 | 감사 앱 진입 인증 성공 |
| `audit_logout` | audit 로그아웃 | 로그아웃 |
| `audit_login_denied` | audit 접근 거부 | 권한 없는 접근 시도 |
| `audit_list_view` | audit 목록 조회 | 이벤트 목록 페이지 열람(필터·결과수 details) |
| `audit_detail_view` | audit 상세 조회 | 개별 이벤트 상세 열람 |
| `audit_export` | audit 내보내기 | CSV/JSON 다운로드(포맷·건수 details) |

> fire-and-forget(비동기)으로 기록 — 사용자 응답을 막지 않고, 실패해도 swallow. "감사 로그 열람·내보내기 행위 자체가 남는다"는 점은 내부 통제 관점에서 매뉴얼에 적어둘 가치 있음.

## 3.5 카테고리 3종 — 화면 배지

| category | 라벨 | 색상 | 포함 액션 성격 |
|----------|------|------|---------------|
| `admin` | 관리자 | 파랑 | 리소스 생성·수정·삭제, 멤버·모델 관리, 감사 열람 |
| `user` | 사용자 | 초록(emerald) | 대화·메시지·워크플로우 실행·피드백·API 호출 |
| `security` | 보안 | 빨강(rose) | 인증 실패·rate limit |

## 3.6 작성 시 주의할 수집 동작 2가지

- **app/member/provider create vs update 분리**: 같은 row의 `created_at`/`updated_at`을 비교해 두 이벤트로 쪼갠다. 단 **`updated_at - created_at ≥ 5초`일 때만 update 이벤트 생성** — 생성 직후 즉시 수정(노이즈)을 거른다. 매뉴얼엔 불필요한 세부지만, "생성과 수정이 별도 줄로 보이는" 이유 설명에 활용 가능.
- **워크플로우는 실행(run)과 노드(node) 둘 다 기록**: 한 번 실행에 노드 수만큼 `workflow_node_execute`가 추가로 쌓여 **볼륨이 크다**. 필터에서 카테고리 "사용자" 선택 시 노드 실행이 다수 섞임을 안내하면 좋음.

---

# 4. 화면 구성 — 사용자가 보는 것

설정 → 감사 로그 진입 시 상단에 **2개 탭**(`NavTabs`)이 있다: **이벤트** / **시스템 로그**.

## 4.1 이벤트 탭 (`/`) — 감사 이벤트 목록

페이지 제목 "**감사로그**", 설명 "시스템 사용자의 행동 기록입니다."

### 목록 테이블 (`EventTable`)

| 열 | 내용 |
|----|------|
| 시간 | `yyyy-MM-dd HH:mm` |
| 카테고리 | 색상 배지(관리자/사용자/보안) |
| 액션 | 한국어 라벨 |
| 사용자 | 이름(외부 시스템에서 보강) + 이메일. 없으면 유형 + ID 앞 8자 |
| 대상 | targetName |
| 비고 | 액션별 한 줄 요약(`getEventSummary`) — 예: 메시지는 `Q:"..." · 모델 · N토큰 · N초`, 워크플로우는 `이름(버전) · 소요초 · 토큰` |
| 결과 | 성공(초록)/실패(빨강)/진행중(주황) 등 |

- 행 클릭 → **상세 페이지** `/[id]`
- 빈 결과: "조건에 맞는 이벤트가 없습니다."
- 페이지당 기본 50건(10~100), `Pagination`은 "총 N건 / P 페이지"

### 필터 바 (`FilterBar`)

| 필터 | 옵션 |
|------|------|
| 카테고리 | 모든 카테고리 / 관리자 / 사용자 / 보안 |
| 액션 | 모든 액션 — **카테고리 선택 시 해당 카테고리 액션만** 동적 노출(`ACTIONS_BY_CATEGORY`) |
| 기간 | 시작/끝 `datetime-local` (from/to) |
| 검색 | 이메일·대상·액션 텍스트. **외부 시스템(예, Keycloak)의 사용자도 조회**해 이름/이메일로 매칭되는 actorId까지 검색(최대 100명). ※ 챕터 본문에서는 "사용자 이름·이메일로도 검색됩니다" 수준으로 추상화 |
| 초기화 | 필터 있을 때만 노출 |

- 모든 필터는 URL 쿼리(`?category=&action=&from=&to=&search=&page=`)로 관리 → 공유·북마크 가능
- 카테고리 변경 시 action 필터 자동 해제

### 상세 페이지 (`/[id]`)

4개 섹션으로 구성: **행위자(Actor)** / **대상(Target)** / **결과**(상태·수집 출처·수집 시점) / **상세 정보**(details를 한국어 필드 라벨로 렌더 + "원본 JSON 보기" 토글). 필드 라벨은 `DETAIL_FIELD_LABELS`(액션별 매핑)를 따른다.

## 4.2 시스템 로그 탭 (`/logs`) — 컨테이너 원시 로그

제목 "**시스템 로그**", 설명 "컨테이너의 stdout 로그를 일일 배치로 적재한 원시 로그입니다." → `spx_system_logs` 테이블 (이벤트와 **다른 데이터**).

| 항목 | 내용 |
|------|------|
| 출처 | 매일 새벽 1시 `docker logs --timestamps`로 어제치 수집(컨테이너 9종 + `audit_event` 비정규화 스냅샷) |
| 필터 | 컨테이너(source) / 레벨(INFO·WARN·ERROR·DEBUG·TRACE·레벨없음) / 기간 / 메시지 검색 |
| 테이블(`LogTable`) | 시각(ms 단위)·컨테이너·레벨 배지·메시지. 행 클릭 시 **원본 라인 펼침** |
| 페이지 | 기본 50건(20~200) |

> 대상 컨테이너 기본 9종: postgres, api, worker, worker_beat, nginx, keycloak, plugin_daemon, redis, sandbox. 레벨은 메시지 정규식 추론(WARNING→WARN, FATAL/CRITICAL→ERROR 등).

> **매뉴얼 판단 필요**: 시스템 로그 탭은 사용자(워크스페이스 관리자)보다 **인프라 운영자** 대상에 가깝다. 감사 로그 챕터에서 비중을 줄이고 "원시 컨테이너 로그를 함께 조회 가능" 수준으로 짧게 다룰지, 별도 절로 뺄지 집필 시 결정. (B2 모니터링 챕터와의 경계도 함께 고려)

---

# 5. 수집 주기·보존 — 사용자 안내가 필요한 시점

| 항목 | 값 | 근거 |
|------|-----|------|
| DB 폴링 주기 | **5분** (기동 60초 후 1회 + `*/5 * * * *`) | `instrumentation.ts` |
| 최초 backfill | 수집기별 1시간 또는 24시간 소급 | 각 collector `since` 기본값 |
| 트리거·nginx·self-audit | 실시간(즉시) | §2.4 |
| 시스템 로그 배치 | 매일 새벽 1시(어제치) | cron `0 1 * * *` |
| 정리(보존) | 매일 새벽 3시, **기본 90일 경과분 삭제** (`AUDIT_RETENTION_DAYS`) | `cleanup.ts` |

> 사용자 관점 핵심 2가지: ① **폴링 항목은 최대 5분 지연될 수 있다**(방금 한 작업이 바로 안 보일 수 있음). ② **기본 90일 보존** — 그 이전 이벤트는 삭제됨. 단 일배치가 `audit_event`를 시스템 로그(`audit_event` source)로 텍스트 스냅샷 적재하므로, 원본이 정리돼도 일일 스냅샷 흔적은 시스템 로그 탭에 남는다.

---

# 6. 결과(status) 정규화

Dify 원본 status가 제각각(succeeded/success/completed…)이라 화면 표시용으로 정규화한다(`getStatusInfo`).

| 정규화 표기 | 색 | 원본 값 |
|------------|----|---------|
| 성공 | 초록 | success, succeeded, completed, normal |
| 실패 | 빨강 | failed, error, stopped |
| 진행중/부분/대기 | 주황 | running, processing, partial-succeeded, pending, waiting |
| (원본 그대로) | 회색 | 매핑 없는 값 |

---

# 7. 권한·격리 — 누가 어디까지 보는가

## 7.1 접근 권한: owner/admin만

- iframe 탭 자체가 `isCurrentWorkspaceManager`(owner/admin)에게만 노출 → 일반/편집자는 메뉴를 못 봄
- 서버측에서도 이중 확인: `adminTenantIds.length === 0`이면 forbidden 페이지("접근 권한이 없습니다 — owner/admin만")
- 근거 권한 모델은 [[references/spx-app-permissions-analysis]](워크스페이스 역할 owner/admin) 인용

## 7.2 워크스페이스 격리 (테넌트)

목록·상세·내보내기 모두 **`tenantId ∈ 내 admin/owner 워크스페이스 OR tenantId = NULL(글로벌)`** 조건으로만 조회된다. 즉:

- 자신이 관리하는 워크스페이스의 이벤트만 본다(다중 워크스페이스 관리자는 합집합)
- 보안·시스템 이벤트(tenantId=NULL)는 모든 관리자에게 보임
- 상세 페이지 직접 접근 시에도 tenant 불일치면 forbidden 리다이렉트

> 식별자 매핑: 쿠키의 Keycloak JWT `sub` → `spx_accounts.sub`로 Dify 계정 id 해석 → `tenant_account_joins`에서 role IN (owner, admin) AND current=true인 tenant 목록(`adminTenantIds`)을 만든다(`audit-session.ts`). JIT provisioning 전(Dify 로그인 이력 없음)이면 미인증 처리.

## 7.3 직접 URL 접근 차단

`/audit` 직접 진입(주소창)은 `sec-fetch-dest=document`로 감지해 Dify 메인으로 리다이렉트한다. **iframe 안에서의 navigation만 허용**(proxy.ts + layout.tsx 이중 체크). 매뉴얼엔 "감사 로그는 설정 화면 안에서만 열린다"로 표현.

---

# 8. 영역 구분 — 대시보드 vs 모니터링 vs 감사 로그 (B2 인용용)

클러스터 B의 핵심 혼동 지점. 한국어판에서 셋을 명확히 분리해야 한다.

| 구분 | 단위 | 무엇을 보여주나 | 위치 | 산출물 |
|------|------|----------------|------|--------|
| **대시보드** | 워크스페이스 | KPI 집계(부서별 리소스·모델 토큰·드릴다운) | `/dashboard` (관리자) | [[references/spx-dashboard-analysis]] |
| **모니터링** | 앱 단위 | 개별 앱의 로그·통계(원본 Dify "Analysis") | 앱 상세 탭 | [[references/spx-monitoring-analysis]] (B2) |
| **감사 로그** | 이벤트 | 누가·언제·무엇을 했는가(행위 추적) | 설정 → 감사 로그 (iframe) | 본 문서 |

> 세 영역은 데이터 원천이 일부 겹친다(대시보드 일부 지표는 `spx_audit_events` mart에서 나옴, §2.3). 하지만 **목적이 다르다**: 대시보드=현황 집계, 모니터링=앱 디버깅, 감사 로그=통제·추적. B2에서 본 표를 3축 비교표로 확장.

---

# 9. 내보내기(Export)

- "내보내기" 버튼 → 다이얼로그 "감사로그 내보내기"
- **현재 필터 조건 그대로** 적용해 다운로드(category/action/search/from/to)
- 포맷 2종: **CSV**("Excel에서 열기", UTF-8 BOM 포함 → 한글 안 깨짐) / **JSON**("프로그램으로 처리")
- CSV 컬럼은 occurred_at·category·action·actor_*·target_*·ip_address·status·source·details(JSON 문자열)
- 내보내기 행위는 self-audit `audit_export`로 기록됨(포맷·건수 포함)

> ⚠️ **상한 불일치 (코드 검증 발견, 2026-06-04)**: 다이얼로그 문구는 "최대 **50,000건**까지 내보낼 수 있습니다"(`ExportButton.tsx` 하드코딩)지만, **버튼이 실제로는 `limit` 파라미터를 전달하지 않는다**(`buildUrl`은 현재 필터 + format만 복사). export 라우트의 limit 기본값이 **10,000**(`Math.min(50000, limit||10000)`)이라 **버튼 다운로드는 사실상 10,000건에서 잘린다**. 50,000은 URL에 `?limit=50000`을 수동으로 붙여야만 도달. → **매뉴얼에는 "최대 50,000"을 단정하지 말고**, 건수 많으면 기간을 나눠 내보내도록 안내(챕터 §내보내기 반영). 잘림 시 사용자 경고도 없음. (개발팀 전달 후보: 버튼에 limit 전달 또는 문구 정정)

---

# 10. 한국어 매핑 박제 (conventions 글로서리 반영 대상)

본 챕터에서 확정한 표기. [[conventions]] 용어집에 일괄 반영 필요(기존 등재: Audit Log=감사 로그, Collector=수집기, dify-audit).

## 10.1 카테고리·행위자 유형

| 내부 값 | 한국어 |
|---------|--------|
| admin / user / security | 관리자 / 사용자 / 보안 |
| account / end_user / api / system | 관리자 / 앱 사용자 / API / 시스템 |

## 10.2 액션 라벨 (전수 — `audit-meta.ts` 기준)

| action | 라벨 | action | 라벨 |
|--------|------|--------|------|
| prompt_update | 프롬프트 수정 | app_create / app_update | 앱 생성 / 앱 수정 |
| api_token_create | API 토큰 발급 | workflow_publish / workflow_draft | 워크플로우 발행 / 초안 저장 |
| dataset_create | 데이터셋 생성 | member_join / member_change | 멤버 가입 / 멤버 변경 |
| document_upload | 문서 업로드 | workflow_node_execute | 워크플로우 노드 실행 |
| conversation_start | 대화 시작 | message_feedback | 답변 피드백 |
| message_send | 메시지 송수신 | provider_model_add / _update | 모델 추가 / 모델 변경 |
| workflow_execute | 워크플로우 실행 | document_delete / dataset_delete | 문서 삭제 / 데이터셋 삭제 |
| api_call | API 호출 | app_delete / api_token_delete | 앱 삭제 / API 토큰 삭제 |
| auth_failed | 인증 실패 | member_remove / member_role_change | 멤버 제거 / 멤버 역할 변경 |
| rate_limit_exceeded | Rate Limit 초과 | audit_login / _logout / _login_denied | audit 로그인 / 로그아웃 / 접근 거부 |
| | | audit_list_view / _detail_view / _export | audit 목록 조회 / 상세 조회 / 내보내기 |

> **용어 주의**: 코드 라벨은 "데이터셋"이지만 [[conventions]]는 knowledge base를 **"지식"**으로 통일했다. 챕터 본문에선 "지식(데이터셋)" 또는 화면 라벨 존중 여부를 집필 시 결정 — A2 [[references/spx-knowledge-permissions]]의 "지식" 표기와 충돌하지 않게 통일 필요. (후속 deferred)

---

# 11. 챕터 작성 시 결정/후속 사항 (deferred to 본격 작성)

- [x] **"데이터셋" vs "지식" 표기 통일** — 옵션 (가) 채택 완료(2026-06-04). 글로서리 "지식" 유지, B1 챕터 본문에 "감사 로그에 '데이터셋'으로 표시되는 항목은 지식 베이스를 의미합니다" 한 줄 안내. progress L201 박제
- [x] **conventions 글로서리 일괄 갱신** — §10 표기 반영 완료(2026-06-04). progress L218에서 권한 모델·로그인 옵션·감사 로그 3 sub-section 흡수
- [ ] **시스템 로그 탭 비중** — 워크스페이스 관리자용 매뉴얼에 어디까지 넣을지(§4.2). 인프라 운영자 대상이면 짧게.
- [ ] **nginx/보안 이벤트 톤** — `NGINX_LOG_PATH` 배포 의존이라 "환경에 따라" 톤(§3.3).
- [x] **KC 추상화 적용** (전역 규칙 #6) — 2026-06-04 완료. §1.1 표기 가이드 신설(보존 영역 명시), §4.1 사용자 열·검색 절 추상화 표기 + 챕터 인용 환기. 코드/메타 식별자(DB·JWT·컨테이너)는 §0.3에 따라 보존. 챕터 본문(mdx)은 완전 추상화 적용
- [ ] **인터리브 챕터 1p 작성** — `ko/use-spx-agent/analytics-audit/audit-log/readme.mdx` (8번 작업에서 슬롯 위치 확정, sidebars.js "통계·감사" 카테고리 주석으로 표시됨)
- [ ] **B2 3축 비교표** — §8을 B2 모니터링 챕터로 확장 인용.
- [ ] **스크린샷** — Phase 5에서 감사 로그 목록·필터·상세·내보내기 다이얼로그 캡처(현재 텍스트 우선).
