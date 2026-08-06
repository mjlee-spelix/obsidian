---
tags: [개발, 학습, 기술스택]
date: 2026-04-23
---

# 기술 스택 학습 가이드

## 학습 순서 (이 프로젝트 기준)

| 순서 | 기술 | 이유 |
|------|------|------|
| 1 | PostgreSQL + SQL | statistic.py 쿼리 읽으려면 필수 |
| 2 | SQLAlchemy | 기존 코드 패턴 따라가려면 |
| 3 | Flask + Flask-RESTx | 새 API 엔드포인트 추가 |
| 4 | Docker Compose | 로컬 Dify 띄우고 테스트 |
| 5 | Next.js + React | 백엔드 다 되면 그때 |
| 6 | Celery | 필요할 때 |

---

## 1. 백엔드 API — Flask + Flask-RESTx

### 개념
- **Flask**: Python으로 웹 서버 만드는 도구. 누군가 `/api/statistics` 같은 주소로 요청을 보내면 Python 함수가 실행돼서 데이터를 돌려주는 구조
- **Flask-RESTx**: Flask에 REST API 만들기 편하게 해주는 확장판. Swagger 문서도 자동 생성해줌

### 이 프로젝트에서 핵심적으로 알아야 할 것
- **Blueprint / Namespace**: Dify는 `console`, `service_api`, `inner_api`로 API를 분리함. 새 대시보드 API를 어디에 추가할지 결정하려면 이 구조를 이해해야 함
- **데코레이터**: `@setup_required`, `@login_required`, `@admin_required` 같은 것들이 엔드포인트마다 붙어 있음 → 인증/권한 체크를 어떻게 거는지
- **요청 파라미터 파싱**: Pydantic 모델로 쿼리 파라미터를 검증하는 패턴 (예: `AppListQuery.model_validate()`)
- **응답 직렬화**: Pydantic 모델로 응답 구조를 정의하는 패턴 (예: `AppPartial`, `AppPagination`)
- 📂 참고 코드: `api/controllers/console/app/statistic.py` (통계 API), `api/controllers/console/tag/tags.py` (태그 API)

### 링크
- 공식 문서: https://flask.palletsprojects.com
- Flask-RESTx 공식: https://flask-restx.readthedocs.io
- 실습 (공식 튜토리얼 — 미니 블로그 만들기): https://tutorial.flask.palletsprojects.com

---

## 2. ORM / DB — SQLAlchemy + PostgreSQL

### 개념
- **PostgreSQL**: 데이터베이스. 엑셀처럼 데이터를 표 형태로 저장하고, SQL로 조회/추가/수정
- **SQLAlchemy**: SQL을 Python 코드로 쓸 수 있게 해주는 도구
  - SQL: `SELECT * FROM messages WHERE app_id = '123'`
  - SQLAlchemy: `Message.query.filter_by(app_id='123').all()`

### 이 프로젝트에서 핵심적으로 알아야 할 것
- **집계 쿼리 (GROUP BY, SUM, COUNT)**: 기존 `statistic.py`가 `SELECT date, SUM(message_tokens + answer_tokens) ... GROUP BY date` 패턴으로 일별 토큰을 집계함. 부서별 통계를 만들려면 이 쿼리를 확장해야 함
- **JOIN**: `App`과 `TagBinding`을 조인해서 "이 태그의 앱들"을 찾고, 그 앱들의 `messages`를 합산하는 식으로 연결됨
- **Mapped 타입 모델 정의**: Dify 모델은 `Mapped[str]`, `Mapped[int]` 같은 SQLAlchemy 2.0 스타일을 씀. 새 모델 추가할 때 이 패턴을 따라야 함
- **Numeric 타입**: 토큰 비용에 `Numeric(10,4)`, `Numeric(10,7)` 사용 → 부동소수점 오차 방지용, 금액 계산에 중요
- **db.paginate()**: 앱 목록 같은 곳에서 페이지네이션 처리하는 방법
- 📂 참고 코드: `api/models/model.py` (Message 모델), `api/controllers/console/app/statistic.py` (집계 쿼리)

### 링크
- PostgreSQL 공식: https://www.postgresql.org/docs
- SQLAlchemy 공식: https://www.sqlalchemy.org
- 실습 (SQL 먼저 — PostgreSQL 특화): https://pgexercises.com
- 실습 (SQLAlchemy 공식 튜토리얼): https://docs.sqlalchemy.org/tutorial

---

## 3. 비동기 작업 — Celery + Redis

### 개념
- **Redis**: 메모리에 데이터를 저장하는 초고속 저장소. 여기서는 "할 일 목록" 역할
- **Celery**: 그 할 일 목록을 보면서 백그라운드에서 작업을 처리하는 일꾼. 토큰 계산처럼 무거운 작업을 API 응답 기다리지 않고 나중에 처리할 때 사용

> [!tip] 지금 당장 깊게 안 파도 됨. 개념만 잡으면 충분

### 이 프로젝트에서 핵심적으로 알아야 할 것
- **Producer/Broker/Consumer 구조**: api 컨테이너가 `.delay()`로 태스크를 Redis에 넣고, worker 컨테이너가 꺼내서 실행하는 흐름. 워크플로우 토큰 저장이 이 방식
- **대시보드에서 신경 쓸 점**: 워크플로우 토큰은 비동기로 저장되니까 실시간 대시보드에서 약간의 지연이 있을 수 있음
- **Celery Beat**: worker_beat 컨테이너가 정기 작업(오래된 데이터 삭제 등)을 스케줄링. `clean_messages`가 통계 원본을 삭제할 수 있어서 보존 기간 설정 확인 필요
- 📂 참고 코드: `api/tasks/workflow_execution_tasks.py` (비동기 저장), `api/extensions/ext_celery.py` (Beat 스케줄)

### 링크
- Celery 공식: https://docs.celeryq.dev
- 개념 이해용 (First Steps): https://docs.celeryq.dev/en/stable/getting-started/first-steps-with-celery.html

---

## 4. 프론트엔드 — Next.js + React + TypeScript

### 개념
- **React**: 버튼, 카드, 차트 같은 UI 컴포넌트를 만드는 JavaScript 라이브러리
- **Next.js**: React 기반 프레임워크. 페이지 라우팅·API 연결 등을 편하게 해줌. Dify 프론트엔드가 이걸로 만들어져 있음
- **TypeScript**: JavaScript에 타입을 추가한 언어. 실수를 줄여줌
  - JS: `let name = "민지"`
  - TS: `let name: string = "민지"`

### 이 프로젝트에서 핵심적으로 알아야 할 것
- **App Router 라우팅**: `(commonLayout)`, `(appDetailLayout)`, `[appId]` 같은 폴더 구조가 곧 URL 경로. 새 대시보드는 `web/app/(commonLayout)/dashboard/`에 추가
- **TanStack Query (React Query)**: `useQuery()` 훅으로 백엔드 API를 호출하고 캐싱하는 패턴. 기존 통계 페이지가 이걸로 데이터를 가져옴
- **ECharts**: 차트 라이브러리. `echarts-for-react` 래퍼로 사용. 기존 차트 설정을 `app-chart-utils.ts`에서 `buildChartOptions()`로 생성
- **Zustand**: 상태 관리 라이브러리. 앱 상세 정보 같은 공유 상태를 관리. 새 대시보드도 별도 스토어 만들면 됨
- **Tailwind CSS**: 클래스명으로 스타일링 (`className="grid grid-cols-2 gap-6"` 같은 식)
- **`'use client'` 지시어**: Next.js에서 클라이언트 컴포넌트를 표시하는 방법. 차트나 상호작용 컴포넌트에 필수
- 📂 참고 코드: `web/app/(commonLayout)/app/(appDetailLayout)/[appId]/overview/` (통계 페이지 전체), `web/service/use-apps.ts` (API 훅)

### 링크
- Next.js 공식 튜토리얼 (제일 잘 돼 있음): https://nextjs.org/learn
- React 기초: https://react.dev/learn
- TypeScript 핸드북: https://www.typescriptlang.org/docs/handbook/intro.html

---

## 5. 컨테이너 — Docker Compose

### 개념
- **Docker**: 앱을 컨테이너라는 독립된 환경에 담아서 실행하는 도구. "내 컴퓨터에서는 됐는데 서버에서 안 돼요" 문제를 없애줌
- **Docker Compose**: 여러 컨테이너(api, db, redis 등)를 한 번에 띄우는 설정 파일. Dify 실행할 때 `docker-compose up` 하는 그것

### 이 프로젝트에서 핵심적으로 알아야 할 것
- **컨테이너 역할 구분**: api(Flask 서버), worker(Celery 비동기 처리), worker_beat(정기 작업 스케줄러), redis(메시지 큐), postgres(DB) — 각각이 왜 분리되어 있는지 이해
- **로그 확인**: `docker logs <컨테이너명>`으로 에러 추적. 컨테이너 내부 로그는 `/app/logs/server.log`
- **환경변수**: `docker/.env`에서 DB 연결, Redis 주소, 기능 플래그 등 설정. `ALLOW_CREATE_WORKSPACE`, `EDITION` 같은 설정이 여기에 있음
- 📂 참고 코드: `docker/docker-compose.yaml`

### 링크
- Docker 공식 Get Started: https://docs.docker.com/get-started
- 브라우저 실습 (설치 없이 바로 실습 가능): https://labs.play-with-docker.com

---

## 6. 인증 — JWT + Keycloak

### 개념
- **JWT**: 로그인한 사용자를 증명하는 토큰. 카드키처럼, API 요청할 때 "나 로그인한 사람이에요"를 증명하는 문자열
- **Keycloak**: 로그인/SSO/권한 관리를 대신 해주는 서버. 승랑님이 담당하는 영역이라 개념만 알면 충분

### 이 프로젝트에서 핵심적으로 알아야 할 것
- **`@login_required` 데코레이터**: 대부분의 API에 붙어 있음. JWT 토큰을 검증해서 `current_user`를 세팅하는 흐름 이해
- **`@admin_required`**: `/all-workspaces` 같은 어드민 API에 사용. 중앙 관리 대시보드 API에도 이런 권한 체크가 필요할 수 있음
- **역할 체계**: OWNER > ADMIN > EDITOR > NORMAL > DATASET_OPERATOR — 현재 Dify의 역할별 접근 권한 이해
- 📂 참고 코드: `api/models/account.py` (TenantAccountRole)

### 링크
- JWT 개념: https://jwt.io/introduction
- Keycloak 공식: https://www.keycloak.org/documentation
