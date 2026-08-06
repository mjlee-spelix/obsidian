# 과제 1 Dify 일배치 워크플로우 설정

## 사용 파일

- Dify 가져오기: [[인사 정보 기반 권한 자동 맵핑 - DB직접연동.yml]]
- DB 스키마: [[01_schema.sql]]
- 테스트 데이터: [[02_seed.sql]]

이 워크플로우는 별도 API 서버 없이 `db-client-node` 플러그인이 PostgreSQL에 직접 접속한다.
실행 모드는 `apply_test`로 고정되며 실제 권한 테이블 `user_permissions`는 변경하지 않는다.

## DB 연결 정보

| 항목 | 값 |
|---|---|
| DB 종류 | PostgreSQL |
| Host | Dify 서버에서 접근 가능한 테스트 PC의 IP |
| Port | `55432` |
| Database | `hr_permission_poc` |
| Username | `hr_user` |
| Password | `hr_pass` |

`localhost`는 Dify 서버 자신을 의미하므로 사용할 수 없다. 사내 Dify 서버에서 테스트 PC의
`55432` 포트에 접근할 수 있어야 한다. 방화벽과 PostgreSQL 포트 공개 여부도 확인한다.

2026-06-24 현재 테스트 PC에서 확인된 사설 IPv4는 `192.168.3.92`이다. 따라서 플러그인
Credential의 Host에는 우선 이 주소를 사용한다. 네트워크가 바뀌면 `ipconfig`로 다시 확인한다.

## 일배치 스케줄

- 트리거: Dify `Schedule Trigger`
- 주기: 매일 오전 1시
- 시간대: `Asia/Seoul`
- 최대 처리 건수: 100건

스케줄 실행 시 `sys.timestamp`를 기준으로 다음 값이 자동 생성된다.

| 변수 | 예시 | 설명 |
|---|---|---|
| `batch_id` | `HR-BATCH-20260624-010000` | 실행 시각 기반 배치 식별자 |
| `batch_date` | `2026-06-24` | 한국 시간 기준 처리일 |
| `limit` | `100` | 한 번에 처리할 최대 이벤트 수 |

## 전체 노드 동작

1. **일배치 스케줄: 매일 오전 1시**
   - Dify가 `Asia/Seoul` 기준 매일 오전 1시에 워크플로우를 시작한다.
2. **Code: 배치 실행값 생성**
   - `sys.timestamp`를 한국 시간으로 변환한다.
   - 배치 ID, 배치 기준일, 최대 처리 건수를 생성한다.
3. **DB 조회: 대상 인사이동**
   - `hr_movement_events`에서 `pending`이고 기준일 이하인 이벤트를 조회한다.
4. **DB 조회: 직원 정보**
   - 대상 직원의 현재 부서, 직급, 재직 상태를 조회한다.
5. **DB 조회: 현재 권한**
   - 운영 권한이 아닌 `user_permissions_test`의 현재 권한을 조회한다.
6. **DB 조회: 권한 정책**
   - 공통·부서·직급별 활성 권한 정책을 조회한다.
7. **DB 조회: 부서 마스터**
   - 유효한 부서 코드를 조회한다.
8. **Code: 배치 권한 diff 계산**
   - 정상 이동은 목표 권한과 현재 권한의 추가·삭제·유지 목록을 계산한다.
   - 퇴직은 테스트 권한 전체를 삭제 대상으로 계산한다.
   - 직원, 부서, 정책이 없으면 `review_required`로 분류하고 권한 변경을 막는다.
9. **DB 반영: 배치 적용/로그/검토큐**
   - 정상 건만 `user_permissions_test`에 반영한다.
   - 모든 결과를 `permission_change_logs`에 기록한다.
   - 검토 필요 건은 `permission_review_queue`에 넣는다.
   - 이벤트 상태를 `processed` 또는 `review_required`로 변경한다.
10. **완료**
   - 전체 건수, 성공 건수, 검토 필요 건수와 각 직원별 결과를 반환한다.

## 안전장치

- 실제 권한 테이블 `user_permissions`는 읽거나 수정하지 않는다.
- 알 수 없는 부서나 정책 누락은 기존 테스트 권한을 유지한다.
- 처리된 이벤트는 다시 조회하지 않는다.
- `request_id`는 유일하게 관리하여 동일 인사이동의 중복 적재를 막는다.
- 권한 추가는 기본키 충돌 시 무시하여 중복 실행에 대비한다.

## 게시 및 활성화

1. YML을 Dify 1.13.3 Workflow로 가져온다.
2. Schedule Trigger에서 시간대와 실행 시각을 확인한다.
3. 테스트 시 Schedule Trigger 노드의 **이 단계 실행**을 누르면 예약 시각을 기다리지 않고 실행된다.
4. 워크플로우를 게시한다.
5. Quick Settings에서 게시된 Schedule Trigger를 활성화한다.

외부 n8n 또는 Cron 호출은 필요하지 않다.

## 테스트용 예상 결과

[[02_seed.sql]]에는 다음 네 건이 포함된다.

- E1001: HR에서 FIN으로 전배 → FIN 권한 추가, HR 권한 삭제
- E1002: manager에서 executive로 승진 → 직급 권한 변경
- E9001: 퇴직 → 테스트 권한 전체 삭제
- E1003: 존재하지 않는 LEGAL 부서로 전배 → 자동 변경 차단, 검토 큐 등록
