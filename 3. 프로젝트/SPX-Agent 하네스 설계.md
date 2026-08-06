---
tags: [프로젝트, dify, AI-Agent, HDD, 하네스]
status: 진행중
date: 2026-04-29
---
# SPX-Agent 하네스 설계

> 이 노트는 **사람용 계획/설계 근거** 문서.
> `.claude/`에는 들어가지 않음 — 에이전트는 이 문서를 읽을 필요 없음.

## 목표

[[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md|SPX-Agent 대시보드]] 프로젝트에서 Claude Code가 **자율적으로 판단하고 구현**할 수 있도록, `.claude/` 폴더에 하네스 문서 체계를 구축한다.

| 수준 | 사람의 역할 | 에이전트의 역할 |
|------|-----------|---------------|
| 하네스 없이 | "대시보드 API 만들어줘" → 맥락 없는 범용 코드 | 지시받은 것만 |
| 기존 HDD | "이 Spec 읽고, H-DASH-01 방어해서 구현해줘" | 지시받은 대로 구현 |
| **하네스 엔지니어링** | "대시보드 KPI 화면 구현해줘" | 스스로 spec 찾고, harness 확인하고, 컨벤션 맞춰서 구현하고, 자기 평가 |

## 왜 필요한가

### 프로젝트 특수 상황

1. **신입 개발자가 대규모 오픈소스(Dify)를 수정하는 프로젝트** — 설계 먼저 확정해야 함
2. **다른 팀원(김이사님) 작업에 의존** — RBAC 스키마 바뀌면 쿼리 전부 영향
3. **도메인 함정이 이미 여러 개 발견됨** — AI에게 알려주지 않으면 매번 같은 실수 반복

### 기대 효과

| 지표 | 하네스 없이 | 하네스 적용 |
|------|-----------|-----------|
| AI 코드 첫 생성 채택률 | 30~40% | 70~80% |
| 도메인 버그 발견 시점 | 테스트 단계 | 구현 시점에서 차단 |
| 코드 재작업률 | 30~40% | 5~10% |
| 지식 보존 | 머릿속 | 문서화 |

> **사람은 판단한다** ("이 프로젝트에서 무엇이 함정인가")
> **AI는 실행한다** ("그 함정을 코드로 막는 방법")

---

## 핵심 원칙 (자료 조사에서 추출)

6개 자료를 조사하여 추출한 원칙. 자료 목록은 하단 "참고 자료" 참조.

**1. "맵을 주되, 매뉴얼은 주지 마라" (OpenAI)**
- 진입점(CLAUDE.md)은 ~100줄 목차. 깊은 내용은 포인터로 분산.
- 컨텍스트는 희소 자원 — 너무 많으면 핵심을 놓침.

**2. "불변식 강제, 구현은 자율적으로" (OpenAI)**
- 넘으면 안 되는 선(아키텍처 규칙)을 명확히 하되, 방법은 에이전트에게 자유를.

**3. Guide(피드포워드) + Sensor(피드백) (Martin Fowler)**
- Guide: 행동 전에 방향 잡기 — CLAUDE.md, Spec, Defect Catalog
- Sensor: 행동 후에 검증 — 테스트, 린터, 자기 평가

**4. Generator-Evaluator 분리 (Anthropic)**
- 에이전트가 자기 작업을 평가하면 항상 "잘했다"고 함 → 평가 기준을 별도 문서로.

**5. "리포지터리가 유일한 진실의 원천" (OpenAI)**
- 머릿속 지식, Slack → 에이전트가 접근 불가 = 존재하지 않음.

**6. Steering Loop (Martin Fowler)**
- 새 함정 발견 → defect-catalog에 추가 → 다음 구현에서 자동 방어.

---

## `.claude/` 디렉토리 구조

> `3. 프로젝트/spx-agent/` 폴더 전체가 실제 프로젝트 레포의 `.claude/`에 그대로 복사·동기화됨.
> 이 노트(`SPX-Agent 하네스 설계.md`)는 `3. 프로젝트/` 루트에 있어 **`.claude/`에 들어가지 않음** — 사람용 계획 문서.

```
[옵시디언 볼트]                                         [실제 프로젝트 .claude/]
3. 프로젝트/                                             spx-agent/.claude/
├── SPX-Agent 중앙 관리 대시보드.md   (사람용 메인 노트)    
├── SPX-Agent 하네스 설계.md         (사람용, 이 파일)    
└── spx-agent/                       ←─── 통째 복사 ───→ .claude/
    ├── CLAUDE.md                                        ├── CLAUDE.md
    ├── architecture.md                                  ├── architecture.md
    ├── conventions.md                                   ├── conventions.md
    ├── hdd/
    │   ├── defect-catalog.md                            │   ├── defect-catalog.md
    │   ├── design.md                                    │   ├── design.md
    │   ├── quality-criteria.md                          │   ├── quality-criteria.md
    │   └── specs/
    │       ├── requirements/{kpi-cards, dept-objects, model-tokens, dept-activity}.md
    │       ├── design/{...동일...}.md
    │       └── tasks/{...동일...}.md
    └── references/
        ├── dify-db-schema.md
        ├── dify-app-modes.md
        └── rbac-schema.md
```

> **분리 이유**: 사람용 계획 문서(이 파일)는 에이전트가 읽을 필요 없음 → 컨텍스트 절약.
> 이제 `spx-agent/` 폴더 전체를 통째 복사/심볼릭 링크할 수 있어 동기화가 단순해짐.

### 문서별 역할

| 문서 | 유형 | 에이전트가 자율적으로 할 수 있게 되는 것 |
|------|------|--------------------------------------|
| **CLAUDE.md** | Guide | 어떤 문서를 읽어야 하는지 스스로 판단 |
| **architecture.md** | Guide | 파일을 어디에 만들지, 어디를 수정할지 판단 |
| **conventions.md** | Guide | Dify 기존 스타일에 맞는 코드 작성 |
| **defect-catalog.md** | Guide | 구현 시 자동으로 harness 방어 적용 |
| **design.md** | Guide | 수정 시 영향 범위 자동 파악 |
| **quality-criteria.md** | Sensor | 구현 완료 후 스스로 품질 검증 |
| **specs/requirements/*.md** | Guide | 무엇을 만들지, 비즈니스 규칙, Harness 방어 |
| **specs/design/*.md** | Guide | API + 쿼리 + 컴포넌트 설계 |
| **specs/tasks/*.md** | Guide | 구현 순서 체크리스트, 테스트 항목 |
| **references/*.md** | 참조 | DB 스키마, AppMode 규칙 등 필요할 때 참조 |

### 에이전트 워크플로우

```
사람: "대시보드 KPI 화면 구현해줘"

에이전트:
  1. CLAUDE.md → "specs/requirements/kpi-cards.md를 봐야겠구나"
  2. specs/requirements/ → 요구사항, "H-DASH-01,03,04,05,07,08 방어"
  2a. specs/design/ → API 설계, 쿼리, 컴포넌트
  2b. specs/tasks/ → 구현 순서 체크리스트
  3. defect-catalog.md → 각 harness 구체적 방어 방법
  4. architecture.md → 파일 위치, Blueprint 등록 방법
  5. conventions.md → 네이밍, import 패턴
  6. references/ → DB 스키마, AppMode 분기 규칙
  7. 구현
  8. quality-criteria.md → 자기 평가
```

---

## 작업 현황

### Phase A: 하네스 문서 구축

| 작업                     | 상태    | 비고                                                                                                                                                                                                     |
| ---------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Defect Catalog 초안 + 검토 | ✅ 완료  | [[3. 프로젝트/spx-agent/hdd/defect-catalog.md]] 15개 패턴. ⚠️ 기획 변경 시 재검토                                                                                                                                     |
| 상세 설계 (design.md)      | ✅ 초안  | [[3. 프로젝트/spx-agent/hdd/design.md]] 변경 규칙 + 의존 관계                                                                                                                                                      |
| KPI 카드 Spec            | ✅ 완료  | specs 3파일: [[3. 프로젝트/spx-agent/hdd/specs/requirements/kpi-cards.md]] · [[3. 프로젝트/spx-agent/hdd/specs/design/kpi-cards.md]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/kpi-cards.md]]                        |
| CLAUDE.md 진입점          | ✅ 완료  | [[3. 프로젝트/spx-agent/CLAUDE.md]]                                                                                                                                                                        |
| references/ 3개         | ✅ 완료  | dify-db-schema, dify-app-modes, rbac-schema                                                                                                                                                            |
| 파일명 영어 통일 + 구조 정리      | ✅ 완료  | 적용계획→plan, 상세설계→design, specs/ 폴더                                                                                                                                                                      |
| architecture.md        | ✅ 완료  | [[3. 프로젝트/spx-agent/architecture.md]] 파일 위치 + 등록 방법                                                                                                                                                    |
| conventions.md         | ✅ 완료  | [[3. 프로젝트/spx-agent/conventions.md]] 코드 스타일 + 테스트 패턴                                                                                                                                                   |
| quality-criteria.md    | ✅ 완료  | [[3. 프로젝트/spx-agent/hdd/quality-criteria.md]] 컴포넌트별 등급 + 체크리스트                                                                                                                                         |
| 부서별 오브젝트 Spec          | ✅ 완료  | specs 3파일: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-objects.md]] · [[3. 프로젝트/spx-agent/hdd/specs/design/dept-objects.md]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dept-objects.md]]               |
| 모델별 토큰 Spec            | ✅ 완료  | specs 3파일: [[3. 프로젝트/spx-agent/hdd/specs/requirements/model-tokens.md]] · [[3. 프로젝트/spx-agent/hdd/specs/design/model-tokens.md]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/model-tokens.md]]. H-DASH-02 핵심 |
| 부서별 활동 Spec            | ✅ 완료  | specs 3파일: [[3. 프로젝트/spx-agent/hdd/specs/requirements/dept-activity.md]] · [[3. 프로젝트/spx-agent/hdd/specs/design/dept-activity.md]] · [[3. 프로젝트/spx-agent/hdd/specs/tasks/dept-activity.md]]. 가장 복잡     |

### Phase B: 구현 (하네스 문서 기반)

| 작업 | 상태 | 비고 |
|------|------|------|
| 화면별 Spec + Harness ID로 AI 구현 지시 | ❌ | Phase A 완료 후 |

### Phase C: Gate Review

| 작업                | 상태  | 비고                 |
| ----------------- | --- | ------------------ |
| Harness 테스트 결과 확인 | ❌   | 도메인 규칙 일치 여부 직접 판단 |
| RBAC 테이블 실제 연동 검증 | ❌   | 김이사님 테이블 완성 후      |
| 통합 테스트            | ❌   |                    |

### 문서 작성 순서 (의존성)

```
[즉시 가능] ← 완료
  ✅ CLAUDE.md, references/ 3개, 파일명 통일

[Dify 소스 분석 필요] ← 완료
  ✅ architecture.md — 파일 위치 + 등록 방법
  ✅ conventions.md — 코드 스타일 + 테스트 패턴

[설계 판단 필요] ← 완료
  ✅ quality-criteria.md — 컴포넌트별 등급 + 체크리스트
  ✅ 나머지 Specs 3개 — 4개 컴포넌트 + 설정 모달

[구현 단계에서]
  ⑧ Harness 단위 테스트 작성
  ⑨ Gate Review (사람 검증)
```

---

## 템플릿

| 템플릿 | 용도 |
|--------|------|
| [[5. 템플릿/하네스 - CLAUDE.md.md]] | 프로젝트 진입점 지도 |
| [[하네스 - 화면별 Spec]] | 화면별 구현 지시서 (requirements + design + tasks) |
| [[5. 템플릿/하네스 - Defect Catalog 엔트리.md]] | 새 결함 패턴 추가 시 |

---

## 다른 프로젝트 비교 (Garment OEM MES)

> 같은 하네스 엔지니어링 철학으로 다른 분이 만든 시스템과 비교.
> Phase B/C 진행 시 자동화 도입 결정 자료.

### 핵심 차이

| 차원 | Garment OEM MES | SPX-Agent (우리) |
|------|----------------|-----------------|
| 프로젝트 성격 | 그린필드 (모노레포 신규) | 브라운필드 (Dify 위에 추가) |
| 에이전트 구조 | 멀티 에이전트 7개 (planner/orchestrator/dev-deployer 등 분담) | 싱글 에이전트 1개 |
| 규칙 자동 참조 | `globs` 프론트매터로 파일 수정 시 자동 로드 | CLAUDE.md 포인터 → 의식적으로 읽어야 함 |
| 검증 자동화 | `domain-validate.sh` + CI 머지 자동 차단 | quality-criteria.md 자기 평가 |
| 정량 임계값 | k6 p95 < 800ms, 커버리지 ≥80%, Dredd 100% | 느슨한 정성 기준 ("응답 2초 이내" 등) |
| 마일스톤 | W4/W12/W17/W20 주차별 Gate | Phase A/B/C 단계별 |
| 슬래시 명령 | `/session-start`, `/daily-check` 등 자동화 | 없음 |
| ADR | `notes/ADR-001~013` 결정 기록 | 데일리 노트 + 이 문서로 대체 |

### 같은 점

- 하네스 엔지니어링 철학: Guide(피드포워드) + Sensor(피드백) 분리
- 불변식 명시: 그쪽은 Forbidden Patterns(C-1~C-8), 우리는 H-DASH-01~15
- 컨벤션 문서화: 역할별/레이어별 코딩 규칙 분리
- 품질 등급 A~D 체계 (우리가 차용)

### 빌릴 만한 것

| 아이디어 | 적용 가능성 | 비고 |
|---------|-----------|------|
| `globs` 자동 참조 | ⭐ 높음 | `.claude/` 옮길 때 frontmatter 추가만 하면 됨 |
| Forbidden Patterns 스크립트 | ⭐ 높음 | H-DASH-01,03 같은 SQL 패턴은 grep으로 검출 가능 |
| 슬래시 명령 | △ 나중에 | `/implement-component KPI` 같은 거 |
| 멀티 에이전트 | ❌ 오버엔지니어링 | 우리 스케일엔 불필요 |
| k6/Dredd 정량 게이트 | △ 부분만 | 부서별 활동 2초 검증 정도는 추가 가능 |
| ADR 노트 | △ 선택 | 큰 결정은 이미 데일리 노트에 있음 |

### 본질적 차이

- **그쪽**: "0 → 1" 프로젝트 → 모든 레이어를 자기들이 정의 가능 (13개 규칙, 7개 에이전트, ISA-95 표준 등 풀스택 통제)
- **우리**: "Dify + α" → 기존 코드 컨벤션을 따라야 하고 신규 영역만 통제 → architecture.md / conventions.md가 "새 코드를 어디에, 어떻게 기존 스타일로 끼워넣나"에 집중

다른 게 아니라 **적합한 수준이 다름**. 지금 우리는 Phase A "문서 토대" 단계, 그쪽은 Phase B-C까지 진행해서 자동화·검증까지 붙인 상태. 우리도 구현 진행하면서 점진적으로 그 수준으로 올라갈 수 있음.

---

## Phase B 적용 — 실전 검증 결과 (2026-04-30)

> Phase A 문서 토대 완성 후 실전 1일차 (5개 컴포넌트 1단계 프론트 목업 완료) 회고에서 추출.
> 일반화 가능 패턴만 정리 — 프로젝트 개별 결정사항은 [[3. 프로젝트/spx-agent/SESSION_HISTORY.md|SESSION_HISTORY]] / 데일리 노트에.

### 1. spec 결함 5종 분류 (검증 체크리스트)

실전 검증에서 발견된 결함을 일반화. 신규 spec 작성 시 이 5종 모두 점검.

| # | 결함 종류 | 증상 | 발견 시점 | 방어책 |
|---|---------|------|---------|------|
| 1 | **구조적 누락** | 페이지 레벨 공통 영역(헤더, 레이아웃)이 어느 컴포넌트 spec에도 안 속함 | 1차 구현 후 화면 비교 | 화면 설계 이미지를 1:1 매핑 — 모든 영역이 어느 spec에 속하는지 |
| 2 | **디테일 누락** | 기능/데이터 구조는 있는데 색상/포맷/라벨 같은 비주얼 디테일 빠짐 | 1차 구현 시각 검증 | spec frontmatter에 `design_image:` 필수, 구현 시 이미지 1:1 대조 |
| 3 | **환경 불일치 (가정)** | spec이 "이상적 환경"(독립 페이지) 가정했는데 실제 환경(모달)이 다름 | 클로드 계획 단계 자발 발견 | spec frontmatter에 `mount_environment:` 필수, 코드 수준 검증 단계 도입 |
| 4 | **환경 불일치 (충돌)** | 마운트 환경의 자체 UI 요소(모달 헤더의 자동 제목 등)와 본 컴포넌트 요소가 충돌 | 1차 구현 시각 검증 | quality-criteria 공통 체크리스트에 "마운트 환경 자체 UI 충돌 없음" 추가 |
| 5 | **이름 모델링 오류** | 컴포넌트 이름이 잘못된 책임을 유도 (예: "header"라 제목까지 만듦) | 시각 검증 + 책임 회고 | 이름 = 책임 범위 매칭. 모호하면 분리 (header → controls) |

> **관찰**: 이번 사이클에 발견된 결함 5종 중 **3종(60%)이 시각 검증으로만 발견**. spec 텍스트만으론 못 잡음.

### 2. 검증 자동화 vs 사람 영역 분리 원칙

> "답이 명확한 것은 자동화, 주관적 판단이 필요한 것은 사람."

| 단계 | 자동화? | 이유 |
|------|------|------|
| 코드 작성 | ✅ 자동 | 클로드 본업 |
| 타입 체크 (`tsc --noEmit`) | ✅ 자동 | 결과 이분법, 비용 작음 |
| 도커 빌드 | ⚠️ 조건부 | 1~3분 비용, 캐시 이슈, 무한 루프 위험 — 백엔드 검증 후 결정 |
| 컨테이너 헬스 체크 | ⚠️ 조건부 | 환경 의존성 큼 |
| **시각 검증 (화면이 spec과 일치)** | ❌ **사람 영역** | 색상/레이아웃/UX는 LLM 판단 어려움 |
| 1:1 이미지 대조 (구조적) | 🤔 가능은 함 | 클로드가 이미지 읽고 비교 가능 — 신뢰도는 사람 < |

**자동화 단계 진화 추천**:
1. 타입 체크 (이미 됨 — quality-criteria의 "테스트 통과" 항목이 자발 행동 유도)
2. 도커 빌드 (검토 — 백엔드 작업 후 결정)
3. 시각 검증 (영원히 사용자 — 의도적 미자동화)

### 3. 프롬프트 진화 곡선 (하네스 성숙도 메트릭)

**하네스가 잘 작동하는지 측정하는 객관 지표 = 프롬프트가 점점 짧아지는가.**

이번 프로젝트의 진화:

| 사이클 | 프롬프트 | 라인 수 | 결과 |
|------|--------|------|-----|
| KPI 1차 (Day 1) | CLAUDE→SESSION→spec 순서 명시 | 5줄 | ✅ |
| KPI 1-B/1-C (Day 2 오전) | spec 내용 일부 박음 | 30줄 | ✅ |
| 1-C 모달 통합 재구현 | 보강된 spec 위치 명시 | 20줄 | ✅ |
| **부서별 오브젝트 차트 (Day 2 오후)** | **`"부서별 오브젝트 차트 1단계 시작. SESSION_HISTORY부터 읽어."`** | **1줄** | ✅ |
| 모델별 토큰 + 부서별 활동 (Day 2 오후) | 두 컴포넌트 동시 + 자기평가 분담 명시 | 3줄 | ✅ |

**원칙**:
- 프롬프트가 점점 짧아지지 않으면 = 하네스 결함
- 매번 같은 정보를 반복 주입하면 그 정보가 spec/CLAUDE.md에 박혀있지 않다는 증거
- 진정한 하네스의 목표는 **"진입점 한 줄"**

### 4. Generator-Evaluator 분리 실증

원칙(Anthropic): 에이전트가 자기 작업을 평가하면 항상 "잘했다"고 함 → 평가 기준을 별도 문서로.

**실증**:
- 자기평가만으론 형식적 통과 가능 — 클로드가 자기 코드를 자기가 평가하는 셈
- quality-criteria.md의 "이미지 1:1 대조" 항목이 이미지 접근 가능해진 후에야 진짜 작동
- **이번 사이클의 결함 5종 중 3종(60%)이 사용자 시각 검증에서만 발견** → 자기평가가 놓친 부분

**개선책**:
- 자기평가는 형식적 검증 (체크리스트, 빌드 통과)
- **사용자 시각 검증을 검증 단계의 정식 일부로 인식** — 누락하면 결함 통과 위험

### 5. 환경 검증의 중요성

> spec은 "이상적 환경"을 가정 → 실제 마운트 환경과 어긋나는 경우 잦음.

**spec 작성 시 필수 frontmatter**:
```yaml
mount_environment: "설정 모달 탭" | "독립 페이지" | "..."
design_image: "images/(화면 설계) ...png"
reference_image: "images/(참조) ...png"     # 선택
```

**코드 수준 검증 단계** — spec의 환경 가정을 실제 코드 환경과 대조:
- 클로드가 작업 계획 단계에서 자발적으로 환경 검증 수행 (이번 사이클에서 URL state 불가 자발 발견)
- spec과 환경이 어긋나면 **계획 단계에서 멈추고 사용자에게 보고**

### 6. 컴포넌트 이름 = 책임 범위

> 이름이 책임을 결정한다. 모호한 이름은 결함을 만든다.

**관찰**: `dashboard-header`라는 이름이 잘못된 책임(제목 포함)을 유도 → 클로드가 자발적으로 `<h1>` 추가 → 모달 자체 헤더와 충돌

**해결**: `dashboard-controls`로 rename — 이름이 정확히 책임 범위(컨트롤만)를 표현

**룰**:
- 컴포넌트 이름이 모호하면 spec 작성 시점에 분리/재명명
- "header" / "section" / "container" 같은 일반 명사는 책임 모호 → "controls" / "toolbar" / "card-grid" 같은 구체적 이름 권장

### 7. spec 분할 전략 — 컴포넌트 단위의 사각지대

**관찰**: 컴포넌트 단위로 spec을 쪼개면 "컴포넌트 위에 있는 페이지 레벨 공통 영역"이 어디에도 안 속함.

**방어**:
- 화면 설계 이미지의 모든 영역을 1:1 매핑 — 어느 spec에 속하는지 체크
- 공통 영역(헤더, 푸터, 레이아웃, 컨텍스트 바)은 별도 spec으로 분리
- design.md In-Scope 표에 모든 영역 명시

### 8. 다음 단계 (Phase B 후속)

이번 사이클에서 식별된 보강 항목:

- [ ] **빌드 자동화 단계 도입** — 백엔드 작업 후 도커 빌드 자동 실행 검토
- [ ] **`mount_environment:` frontmatter 전 spec에 일괄 적용** — 현재는 일부만
- [ ] **이미지 1:1 대조의 정식 평가 절차화** — quality-criteria.md "평가 절차" 섹션에 명시 단계 추가
- [ ] **Phase 2 동적 인터랙션 spec** — KPI 클릭 → 차트 영역 변경, 부서 클릭 → 우측 드로어
- [ ] **컨텍스트 바 spec** — Phase 2 인터랙션과 묶어서 도입
- [ ] **`globs` 자동 참조 도입 검토** — Garment OEM MES 비교에서 식별, Phase B 후반 검토
- [ ] **Forbidden Patterns 스크립트** — H-DASH-01,03 SQL 패턴 grep 검출

---

## 데이터 마트 설계 작업 (2026-05-07)

> 5/7 AAI 주간 보고 회의에서 결정. 대시보드 응답 시간 단축 목표(3초 이내).
> 회의록: [[2. 회의록/0507 AAI 주간 보고]]

### 배경

- 대시보드 쿼리가 OLTP 테이블(`workflow_runs`, `accounts`, `departments`, `resource_ownership` 등)을 직접 조회 → 응답 느림
- 회사 표준 흐름: 대시보드용 별도 마트 스키마 + 배치 갱신 + 인덱싱
- 현재 TanStack staleTime 5분은 클라이언트 캐시 — 서버 응답 자체를 빠르게 하는 건 다른 레이어. 둘 다 필요.

### 작업 범위 (7단계)

#### 1. 사전 분석 (가장 먼저)
- **현재 쿼리 인벤토리**: KPI 4종 + 차트 + 테이블 + drill-through 11종 = 약 20개 SQL 모으기
- **응답 시간 측정**: 어떤 쿼리가 느린지 (느린 것만 마트화 가치 있음)
- **차원/측정값 패턴 추출**: 반복되는 GROUP BY 컬럼들 (department_id, date, model, status…)

#### 2. 스키마 설계
- **팩트 테이블** (`fact_*`): 측정값 모음
  - 예: `fact_workflow_run_daily(date, department_id, app_id, run_count, token_sum, cost_sum, error_count)`
- **디멘션 테이블** (`dim_*`): 대부분 OLTP의 departments/accounts 그대로 JOIN해도 충분 → 별도 dim 안 만들어도 OK
- **시간 입도**: 일별 vs 시간별 vs 분별
  - 대시보드 필터가 "최근 7일/30일"이면 일별로 충분
- **보존 기간**: 6개월 / 1년 (디스크 vs 분석 가치)

#### 3. 인덱스 설계
- 자주 필터되는 컬럼 (date, department_id, status)
- 복합 인덱스: `(date DESC, department_id)` 같은 조합
- 외래키 인덱스 (FK는 기본 인덱스 안 만들어짐)

#### 4. 배치 잡
- **주기**: 5분 / 15분 / 1시간 (실시간성 vs 부하)
- **증분 vs 전체**: incremental(어제 것만 새로) vs full refresh
- **실행 방식**: celery beat / cron / 호스트 cron
- **실패 처리**: 재시도, 알람, 마지막 성공 시각 기록

#### 5. 마이그레이션
- alembic revision: 마트 테이블 + 인덱스 생성
- **초기 백필**: 과거 데이터 한 번에 채우는 SQL (배치로는 시간 오래 걸림)

#### 6. 백엔드 서비스 변경
- 기존 `dashboard_*_service.py`의 SQL을 마트 테이블 참조로 변경
- **Fallback 정책**: 마트 신선도 < 1시간이면 마트, 아니면 OLTP? 아니면 무조건 마트?
- TanStack staleTime 5분과 무관하게 동작

#### 7. 모니터링
- 마트 freshness 표시: `last_updated_at` 컬럼
- 배치 성공/실패 로그 (`mart_refresh_log` 테이블)

### 실제 작업 순서 (권장)

```
1) 쿼리 인벤토리 + 응답 시간 측정 (1일)
   → 어디가 진짜 병목인지 데이터로 결정
2) 스키마 설계 문서 작성 (specs/design/data-mart.md)
   → 리뷰 받고 시작
3) alembic 마이그레이션 + 백필 SQL (반나절)
4) 배치 잡 1개 시범 구현 (1일)
   → fact_workflow_run_daily만 먼저
5) 서비스 1개 마트 참조로 변경 → 응답 시간 비교
6) 검증되면 나머지 일괄 적용
```

### TanStack staleTime과 관계

| 레이어 | 무엇 | 효과 |
|------|------|------|
| TanStack staleTime 5분 | 브라우저에 결과 캐시 | 같은 사용자 재요청 빠름 (서버 응답 X) |
| 데이터 마트 | 서버 응답 자체를 빠르게 | 모든 사용자 첫 요청 빠름 |

→ 둘 다 필요. 마트 도입해도 staleTime 그대로 유지.

### 스켈레톤 UI와 관계

회의에서 "프레임 먼저 / 쿼리 결과 나중" 의견 나옴 (체감 로딩 시간 단축).
**순서**: 데이터 마트 먼저 적용 → 그래도 느리면 스켈레톤 적용.
이미 `<Skeleton />` 컴포넌트는 있어서 큰 작업 아님 (TanStack `isPending` 활용).

### 다음 액션

- [ ] `3. 프로젝트/spx-agent/specs/design/data-mart.md` 빈 문서 골격 만들기
- [ ] 5/8 dev 배포 전, "어떤 쿼리가 느린지" 측정 결과 1장 확보
- [ ] 시범 팩트 테이블 1개(`fact_workflow_run_daily`) 우선 진행

---

## 참고 자료

### 조사한 자료

| 자료                                                                       | 출처                     | 비고                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------------------------------------------------------------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 하네스 엔지니어링: 에이전트 우선 세계에서 Codex 활용하기                                       | OpenAI (Ryan Lopopolo) | [[하네스 엔지니어링 - 에이전트 우선 세계에서 Codex 활용하기.md\|로컬 번역본]]<br>[링크](https://openai.com/ko-KR/index/harness-engineering/)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Harness Engineering for Coding Agent Users                               | Martin Fowler 사이트      | [링크](https://martinfowler.com/articles/harness-engineering.html)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Harness design for long-running application development                  | Anthropic Labs         | [[Harness design for long-running application development.md\|로컬 저장본]]<br>[링크](https://www.anthropic.com/engineering/harness-design-long-running-apps)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Harness Engineering: Building the Operating System for Autonomous Agents | Medium (Plaban Nayak)  | [[Harness Engineering - Building the Operating System for Autonomous Agents.md\|로컬 저장본]]<br>[링크](https://medium.com/the-ai-forum/harness-engineering-building-the-operating-system-for-autonomous-agents-1e20c105f689#id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6IjE5Y2FhZWNkZThmNDg1ZThmNTkzOGY0OGFiYTBjZTdhMzU4MWYwMjciLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJhenAiOiIyMTYyOTYwMzU4MzQtazFrNnFlMDYwczJ0cDJhMmphbTRsamRjbXMwMHN0dGcuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJhdWQiOiIyMTYyOTYwMzU4MzQtazFrNnFlMDYwczJ0cDJhMmphbTRsamRjbXMwMHN0dGcuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJzdWIiOiIxMTQ4NTQzMTM2OTQ3OTgxNjU2MDciLCJlbWFpbCI6Im15bng2eUBnbWFpbC5jb20iLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwibm9uY2UiOiJub3RfcHJvdmlkZWQiLCJuYmYiOjE3Nzc0MjY2NzYsIm5hbWUiOiJNaW5qaSIsInBpY3R1cmUiOiJodHRwczovL2xoMy5nb29nbGV1c2VyY29udGVudC5jb20vYS9BQ2c4b2NJS0JHSmdGeUl5cTFnWUV2Sl9qZ0xTYjNoX1FYUGhUekkxTUprSGMtZ1NBZHpLNnc9czk2LWMiLCJnaXZlbl9uYW1lIjoiTWluamkiLCJpYXQiOjE3Nzc0MjY5NzYsImV4cCI6MTc3NzQzMDU3NiwianRpIjoiMzkzZDhhYmY0YjBhMWE4ODIwNjdiZGZhMDNhZjUzODNkN2RjNjg5NyJ9.T8t5L7umVdfFrh0RvjI1s7WsKMqzk2UVxuptVip1xvedvNMqGRrYMTPVRMQZChRjfMKUZyrSzaiQL2lqsnLS0Y18bMHjW5LOga8ouNFaA0tBc3ZBGGmbg3euVnG3ZArM6YrdGfJpQqGRiCt3Uvl3kXweTzUfUkAnQGcCM3C_rt0A50Jl8KJKKdSLcpszmmgwqqyHrYNvqZwEN9TSuRVVIkj62yOvX9j5uJXP_sNVaQst6qvhv6a7Kptl7hN29GIuHfFmZ6kWnTGMC9DNX_Po4wgz1Zl8ayGVG8Pq6UjGLat9SsmBmUm4nY3oVlSdLj_UIe5bQfsJTvgSHN8b0FMkEg) |
| OpenAI의 harness engineering을 claude skill로 만들기                           | junheedot 블로그          | [링크](https://junheedot.tistory.com/m/entry/Open-AI%EC%9D%98-harness-engineering%EC%9D%84-claude-skill%EB%A1%9C-%EB%A7%8C%EB%93%A4%EA%B8%B0)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Everything Claude Code (ECC)                                             | goddaehee 블로그          | [링크](https://goddaehee.tistory.com/575)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

### 관련 노트

- [[3. 프로젝트/SPX-Agent 중앙 관리 대시보드.md]] — 메인 프로젝트 노트
- [[3. 프로젝트/spx-agent/hdd/defect-catalog.md|Defect Catalog]] — 결함 패턴 15개
- [[3. 프로젝트/spx-agent/hdd/design.md|상세 설계]] — 변경 영향 규칙 + 의존 관계
- [[3. 프로젝트/spx-agent/hdd/specs/kpi-cards.md|KPI 카드 Spec]] — 첫 번째 화면 Spec
- [[3. 프로젝트/spx-agent/CLAUDE.md]] — 에이전트 진입점
