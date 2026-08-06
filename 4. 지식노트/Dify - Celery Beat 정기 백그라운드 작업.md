---
tags: [dify, 개발, AI-Agent]
date: 2026-04-22
---
# Dify - Celery Beat 정기 백그라운드 작업

## 핵심
- `worker_beat` 컨테이너는 Celery Beat(스케줄러)을 실행하며, 정해진 주기마다 태스크를 자동으로 Redis 큐에 등록한다
- 실제 실행은 `worker` 컨테이너가 담당 — worker_beat은 "언제 할지"만 정하고, "실제로 하는 건" worker
- 주로 **데이터 정리(retention)**, **상태 체크**, **예약 실행** 등 사용자 요청과 무관한 관리 작업을 처리
- 설정 위치: `api/extensions/ext_celery.py` (feature flag로 각 작업 개별 활성화/비활성화)

## 상세

### worker_beat의 위치

```
api 컨테이너      → 사용자 요청 처리, 비동기 태스크 생성
worker 컨테이너   → 큐에서 태스크 꺼내서 실행
worker_beat 컨테이너 → 정해진 시간마다 태스크를 큐에 자동 등록 (스케줄러)
```

리눅스 crontab과 같은 역할이지만, Celery 안에서 동작하며 Redis를 통해 worker에게 작업을 전달한다.

### 등록된 정기 작업 목록

#### 데이터 정리 (retention)

| 작업 | 주기 | 하는 일 |
|------|------|---------|
| `clean_messages` | 매일 새벽 4시 | 보존 기간 지난 오래된 대화 메시지 삭제 |
| `clean_workflow_runlogs` | 매일 새벽 2시 | 만료된 워크플로우 실행 로그·노드 실행·트리거 로그 삭제 |
| `clean_embedding_cache` | 매일 새벽 2시 | 오래된 임베딩 캐시 삭제 |
| `clean_unused_datasets` | 매일 새벽 3시 | 안 쓰는 데이터셋 인덱스 정리 |
| `clean_workflow_runs_task` | 매일 자정 | Sandbox 워크플로우 실행 정리 (SaaS 전용) |

#### 상태 체크/모니터링

| 작업 | 주기 | 하는 일 |
|------|------|---------|
| `check_upgradable_plugin` | 15분마다 | 업그레이드 가능한 플러그인 확인 |
| `datasets-queue-monitor` | 30분마다 | dataset 큐 길이 모니터링, 임계치 초과 시 알림 |
| `batch_update_api_token_last_used` | 설정 주기 | API 토큰 마지막 사용 시각을 Redis → DB로 일괄 반영 |

#### 예약 실행

| 작업 | 주기 | 하는 일 |
|------|------|---------|
| `workflow_schedule_task` | 설정 주기 | 예약된 워크플로우 실행 시간이 되면 실행 |
| `human_input_form_timeout` | 설정 주기 | HITL 입력 폼 타임아웃 처리 → 워크플로우 재개/중단 |
| `trigger_provider_refresh` | 설정 주기 | 만료된 트리거 인증 정보 갱신 |

#### 기타

| 작업 | 주기 | 하는 일 |
|------|------|---------|
| `mail_clean_document_notify` | 매주 월요일 10시 | 자동 비활성화된 데이터셋 소유자에게 이메일 알림 |
| `create_tidb_serverless` | 매시간 | TiDB 서버리스 클러스터 생성 (TiDB 사용 시) |
| `update_tidb_serverless_status` | 10분마다 | TiDB 클러스터 상태 확인 |

### 동시 실행 방지

여러 worker가 같은 태스크를 중복 실행하지 않도록 Redis 락을 사용:
- `clean_messages` → 락 키: `retention:clean_messages`
- `clean_workflow_runs_task` → 락 키: `retention:clean_workflow_runs_task`

## 관련 노트
- [[4. 지식노트/Dify - 토큰 데이터 저장 흐름 (동기·비동기).md]]
- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]]
