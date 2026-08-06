---
tags: [dify, 개발, CS, AI-Agent]
date: 2026-05-08
---
# Dify - 데이터 소스 3종 비교 (OLTP vs 로그 파일 vs 내부 로그 테이블)

## 핵심
- Dify에서 운영 데이터를 얻을 수 있는 소스는 3종: **OLTP 테이블 / 로그 파일 / 내부 로그 테이블**. 각각 기록 시점·메커니즘·정체가 다름
- audit 스키마(`audit.audit_events` 등)는 **로그 파일을 ETL해서 만든 정규 테이블** — 새 4번째 영역이 아니라 ②의 가공물
- 대시보드/모니터링 차트 설계 시 "어느 소스에서 가져올까"는 **차트가 묻는 질문의 성격**으로 결정 — 비즈니스 사실 / 인프라 부수 / 실행 펼침

## 상세

### 3종 소스 정의

| 구분 | ① OLTP 테이블 | ② 로그 파일 | ③ 내부 로그 테이블 |
|---|---|---|---|
| 대표 예 | `messages`, `workflow_runs`, `apps`, `datasets`, `accounts` | api.log, nginx access.log, celery worker stdout, plugin daemon log | `workflow_app_logs`, `workflow_node_executions` |
| 저장 위치 | Postgres `public` 스키마 | 컨테이너 파일/stdout (docker logs) | Postgres `public` 스키마 |
| 누가 씀 | Dify 비즈니스 service 코드 | 프레임워크/인프라 (Flask middleware, gunicorn, nginx, Celery) | GraphEngine 이벤트 핸들러 + service |
| 데이터 성격 | 정규화된 비즈니스 엔티티 | 반정형 텍스트 (한 줄 = 한 이벤트) | 정규화된 실행/이벤트 row |
| 보존 | 영구 | 일자 rotation, 며칠~몇 주 | 영구 |
| 정체 | "**무엇이 존재하는가**" | "**프레임워크/인프라 레벨에서 무슨 일이 있었나**" | "**그게 어떻게 실행됐는가** (OLTP의 한 row를 펼침)" |

### 기록 시점 — 트리거 비교

| | OLTP | 로그 파일 | 내부 로그 테이블 |
|---|---|---|---|
| 트리거 | 사용자 비즈니스 액션 완료 | 매 HTTP 요청 / 매 예외 / 매 태스크 | 워크플로우 노드 이벤트 / 실행 종료 |
| 동기/비동기 | 대부분 동기, `workflow_runs`는 비동기 | 동기 | 대부분 비동기 (Celery) |
| 빈도 | 1요청 = 1~소수 row | 1요청 = 여러 줄 | 1요청 = 노드 수만큼 row |
| 데이터 출처 | 요청 본문 + LLM 응답 + 인증 컨텍스트 | nginx/Flask/Celery 자체 + 코드의 logger 호출 | GraphEngine 내부 상태 |
| 지연 | 즉시 또는 수초 (Celery) | 즉시 | 수초~수십초 (Celery 큐) |

### 시퀀스 예시 — ADVANCED_CHAT 1번 호출

```
사용자: "오늘 날씨 알려줘" (POST /chat-messages)
   │
   ▼
[nginx]  ─────► access.log 1줄                         ② 로그파일
   │
[gunicorn → Flask]
   │  before_request 미들웨어 ─► api.log              ② 로그파일
   │
[controller → service → AdvancedChatAppGenerator]
   ├─► conversations INSERT (없으면)                  ① OLTP (동기)
   │
[GraphEngine 실행]
   ├─► 노드별 NodeRunStarted/Succeeded
   │     └─► WorkflowPersistenceLayer.on_event()
   │           └─► Celery 큐 → workflow_node_executions  ③ 내부테이블 (비동기 N개)
   │
   └─► 워크플로우 종료 시
         ├─► WorkflowRunSucceededEvent
         │     └─► Celery → workflow_runs INSERT      ① OLTP (비동기)
         │
         └─► _save_message() → messages INSERT        ① OLTP (동기)
   │
[service finally]
   └─► workflow_app_logs INSERT                        ③ 내부테이블 (동기)
   │
[after_request] ──► api.log                            ② 로그파일
[gunicorn → nginx → 사용자]
```

**한 호출에서 발생하는 기록 합계**:
- ① OLTP: messages 1, workflow_runs 1, conversations 0~1
- ② 로그 파일: nginx 1줄, api.log 여러 줄, celery 로그 여러 줄
- ③ 내부 로그 테이블: workflow_node_executions N (노드 수), workflow_app_logs 1

### audit 스키마와의 관계

| | 출처 | 저장 형태 |
|---|---|---|
| `audit.audit_events` | ② 로그 파일 + ① OLTP 일부 | 정규 테이블로 ETL |
| `audit.system_logs` | ② 로그 파일 (컨테이너 stdout 통째) | 일일 배치로 텍스트 적재 |

→ audit은 새 4번째 영역이 아니라 **②를 SQL로 질의 가능하게 가공한 결과**. nginx access.log를 직접 grep하는 대신 `SELECT FROM audit.audit_events WHERE source='nginx_log'`로 조회.

### 어느 소스를 쓸지 결정 기준

> 차트가 묻는 질문의 성격으로 결정.

| 질문 성격                                                      | 적합한 소스                 |
| ---------------------------------------------------------- | ---------------------- |
| "**누가 / 무엇을 / 얼마나** 썼나"                                    | ① OLTP                 |
| "프레임워크/인프라 레벨에서 무슨 일?" (HTTP P95, IP, Rate limit, 거버넌스 이력) | ② 로그 파일 / audit_events |
| "어떻게 실행됐나 (노드 단위, trace_id, 디버깅)"                          | ③ 내부 로그 테이블            |

### 수집정보 항목 → 어디서 가져오나 (예시)

| 항목                                   | OLTP                      | 내부 로그 테이블                       | 로그 파일 (audit)       |
| ------------------------------------ | ------------------------- | ------------------------------- | ------------------- |
| 토큰 소비량 / 모델별 비용                      | ✅ messages, workflow_runs | △                               | —                   |
| 노드별 실행 시간                            | ❌                         | ✅ workflow_node_executions      | —                   |
| 호출 경로 (WEB_APP/SERVICE_API/DEBUGGER) | △ messages.from_source    | ✅ workflow_app_logs.from_source | —                   |
| 좋아요/싫어요                              | ✅ message_feedbacks       | —                               | —                   |
| 프롬프트 수정 이력                           | ❌                         | ❌                               | ✅ audit             |
| HTTP P50/P95/P99                     | ❌                         | ❌                               | ✅ audit (nginx_log) |
| 인증 실패, Rate limit                    | ❌                         | ❌                               | ✅ audit             |
| 모델 설정 변경 이력                          | ❌                         | ❌                               | ✅ audit             |

→ 거버넌스/HTTP 인프라 카테고리는 OLTP·내부테이블에 사실상 없음 → audit이 안 오면 그릴 수 없는 그래프들.

### 대시보드 관점에서의 함의

1. **실시간성** 필요한 KPI(에러율 24h)는 비동기 Celery 큐 지연(③/일부 ①) 때문에 수분 늦게 보일 수 있음. 마트 갱신 주기보다 이게 먼저 병목.
2. **HTTP P95, IP, Rate limit** 같은 인프라 항목은 ②에서만 옴 → audit ETL 들어와야 그릴 수 있음.
3. **노드별 실행 시간** 같은 워크플로우 디테일은 ③에서만 옴. ①의 `workflow_runs`엔 합산만 있음.
4. **"방금 실행한 거 안 보임"** 류 이슈는 거의 항상 비동기 Celery 큐 지연(③ 또는 일부 ①의 특성).

### SPX-Agent 현 대시보드 12종 매핑 (2026-05-08 결정)

> 자세한 컴포넌트별 매트릭스는 [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-drill-through.md]] 참조.

| 영역 | 차트 수 | 비고 |
|---|---|---|
| ① OLTP | **11개** | messages + workflow_runs + apps + datasets + accounts (+ RBAC) |
| ② audit | **1개** | top-error-types만 (action 분류가 이미 정규화됨) |
| ③ 내부 로그 테이블 | **0개** | 노드 단위 디테일 차트가 현 설계에 없음 |

### 추가 차트 후보 (audit/system_logs 활용 — 현 설계 밖)

audit이 들어오면 만들 수 있는 차트:
- 인증 실패 / Rate limit 추이 (`source=nginx_log`, `category=security`)
- 엔드포인트별 호출수 + 상태코드 분포
- 거버넌스 이력 (프롬프트 수정, 앱 생성·삭제, API 토큰 발급)
- 멤버 권한 변경 이력
- audit 페이지 메타 감사

system_logs 활용:
- 컨테이너별 ERROR 라인 추이
- 로그 레벨 분포

→ 회의에서 "현 12종 외 추가 후보 있어?" 물으면 위 3가지(보안/HTTP/거버넌스)를 우선 제안.

## 관련 노트
- [[Dify - AppMode별 토큰 저장·통계 쿼리 전체 흐름]] — messages vs workflow_runs AppMode별 분기 (① 영역 상세)
- [[Dify - 통계·토큰 DB 스키마 구조]] — workflow_runs / workflow_node_executions / workflow_trigger_logs 스키마 (① + ③)
- [[Dify - 전체 DB 스키마 관계도]] — 전체 ER 관계
- [[Dify - 토큰 데이터 저장 흐름 (동기·비동기)]] — 비동기 Celery 큐 메커니즘
- [[Dify - workflow_node_executions에서 모델별 토큰 추출]] — ③ 활용 사례
- [[3. 프로젝트/spx-agent/references/audit-schema.md]] — audit 스키마 5개 테이블 명세 (② ETL 결과)
- [[3. 프로젝트/spx-agent/references/dify-db-schema.md]] — ① OLTP 명세
- [[0. Inbox/Phase2 KPI 드릴스루 데이터 소스 매트릭스]] — 12종 컴포넌트별 소스 매핑
- [[0. Inbox/수집정보 및 대시보드 후보_안승랑]] — 수집해야 할 정보 위시리스트
