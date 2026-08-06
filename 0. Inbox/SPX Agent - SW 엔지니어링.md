### SW 엔지니어링 — 프로젝트에 필요한 키워드

**1. 레이어드 아키텍처 (Layered Architecture)**

- Dify가 `Controller → Service → Model` 3계층으로 분리돼 있음. 새 대시보드 API 만들 때 비즈니스 로직을 어디에 넣을지 판단하려면 이 구조를 이해해야 함
- 예: `statistic.py`(컨트롤러)는 직접 쿼리를 짜고, `app_service.py`(서비스)는 로직을 분리 — 어느 쪽 패턴을 따를지

**2. REST API 설계 원칙**

- 엔드포인트 네이밍, HTTP 메서드 선택, 상태 코드, 쿼리 파라미터 설계. 새 통계 API(`/dashboard/statistics/*` 같은)를 설계할 때 직접 필요
- 페이지네이션, 필터링, 정렬 패턴도 포함 (`GET /apps?tag_ids=...&page=1` 같은 기존 패턴 참고)

**3. 디자인 패턴 (프로젝트에서 만난 것들)**

- **데코레이터 패턴** — `@login_required`, `@admin_required` 같은 인증/권한 체크가 이 패턴. 새 API에도 동일하게 적용해야 함
- **팩토리 패턴** — 프론트엔드 `createBizChartComponent()`가 이것. 새 차트 추가할 때 이 패턴을 따라야 함
- **리포지토리 패턴** — `celery_workflow_execution_repository.py`가 DB 접근을 추상화하는 패턴. 서비스 계층과 DB 사이를 분리

**4. 메시지 큐 / 비동기 처리 패턴**

- Producer-Consumer 모델, Task Queue 개념. Celery가 정확히 이 구조이고, 워크플로우 토큰이 비동기로 저장되는 이유. 대시보드에서 "데이터 지연"이 왜 생기는지 설명할 수 있어야 함

**5. DB 설계 기초**

- **정규화** — 테이블 간 관계(1:N, M:N) 이해. `App↔Message`(1:N), `App↔Tag`(M:N via TagBinding)
- **인덱싱** — 집계 쿼리(`GROUP BY date`, `SUM(tokens)`) 성능에 직결. 부서별 통계 쿼리가 느려지지 않으려면
- **집계 쿼리 설계** — 기존 앱 단위 → 태그/부서 단위로 확장할 때 JOIN + GROUP BY 조합이 복잡해짐

**6. 인증(AuthN) vs 인가(AuthZ)**

- 인증 = "누구인지 확인" (JWT), 인가 = "뭘 할 수 있는지 확인" (RBAC). Dify의 `OWNER > ADMIN > EDITOR > NORMAL > DATASET_OPERATOR` 역할 체계가 인가. 권대리님 RBAC 작업과 연결되는 개념

**7. 컴포넌트 설계 / 상태 관리 패턴**

- **컴포넌트 합성** — React에서 작은 컴포넌트를 조합해 큰 UI를 만드는 원리. 차트 + 필터 + 테이블을 조합해서 대시보드를 구성
- **클라이언트 상태 vs 서버 상태** — Zustand(클라이언트)와 TanStack Query(서버 캐시)를 왜 나눠 쓰는지

**8. 컨테이너화 & 서비스 분리**

- api / worker / worker_beat / redis / postgres가 왜 별도 컨테이너인지. 하나가 죽어도 나머지가 살아있는 구조 = **관심사 분리(Separation of Concerns)**의 인프라 버전