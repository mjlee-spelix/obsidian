# 과제 1 POC - 인사 정보 기반 권한 자동 맵핑

인사이동 이벤트를 일괄 조회하고 권한 변경까지 수행하는 Dify 1.13.3용 POC이다.

## 주요 산출물

- [[인사 정보 기반 권한 자동 맵핑 - DB직접연동.yml]]
  - 권장 워크플로우
  - `db-client-node`로 PostgreSQL에 직접 연결
  - Dify Schedule Trigger로 매일 오전 1시 자동 실행
  - 일배치 인사이동 조회부터 테스트 권한 반영까지 한 번에 수행
- [[인사 정보 기반 권한 자동 맵핑 - API서버연동.yml]]
  - 별도 Python API 서버를 사용하는 이전 방식
- [[인사 정보 기반 권한 자동 맵핑 - 플러그인노드샘플.yml]]
  - DB 플러그인 원본 설정을 보존한 참고 파일
- [[01_schema.sql]]
  - 테이블과 인덱스 생성
- [[02_seed.sql]]
  - 직원, 정책, 현재 권한, 인사이동 테스트 데이터 생성
- [[dify_workflow_setup.md]]
  - Dify 설정, 노드 흐름, 일배치 실행 방법

## 테스트 PostgreSQL

| 항목 | 값 |
|---|---|
| 컨테이너 | `seoyon-hr-postgres` |
| Host port | `55432` |
| Container port | `5432` |
| Database | `hr_permission_poc` |
| Username | `hr_user` |
| Password | `hr_pass` |

기존 Dify PostgreSQL과 충돌하지 않도록 별도 컨테이너와 `55432` 포트를 사용한다.

## 안전 범위

- 실제 권한 원본: `user_permissions`
- POC 변경 대상: `user_permissions_test`
- 변경 이력: `permission_change_logs`
- 수동 검토 대상: `permission_review_queue`
- 인사이동 수신 큐: `hr_movement_events`

워크플로우의 실행 모드는 `apply_test`로 고정되어 실제 권한 원본은 변경하지 않는다.
직원·부서·권한 정책이 확인되지 않으면 자동 변경하지 않고 검토 큐로 보낸다.

## 문자 인코딩

SQL, Markdown, YML 파일은 UTF-8로 저장한다. Windows PowerShell에서 파일을 읽거나 쓸 때도
`-Encoding UTF8`을 지정한다.
