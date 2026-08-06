---
tags: [하네스, 템플릿]
---
# {{project_name}}

> 이 파일은 AI 에이전트의 **진입점 지도**입니다.
> 맵처럼 짧게 유지하고, 깊은 내용은 포인터로 연결하세요.
> 목표: ~100줄. 매뉴얼이 아닌 목차.

## 프로젝트 개요

- **설명**: (한 줄 설명)
- **기술 스택**: (예: Flask, SQLAlchemy, Next.js, PostgreSQL, Redis)
- **베이스 코드**: (예: Dify v0.x.x 기반 커스텀)

## 빠른 시작

```bash
# 백엔드
cd api && poetry install
flask run

# 프론트엔드
cd web && pnpm install
pnpm dev

# 테스트
cd api && pytest tests/
cd web && pnpm test
```

## 디렉토리 구조

```
프로젝트 루트에서 이 기능과 관련된 핵심 경로만 기술.
모든 경로를 나열하지 말 것 — 에이전트가 탐색할 수 있음.

api/
├── controllers/console/   ← 어드민 API 엔드포인트
├── services/              ← 비즈니스 로직
└── models/                ← SQLAlchemy 모델

web/
├── app/(commonLayout)/    ← 페이지 라우트
└── app/components/        ← 공통 컴포넌트
```

## 핵심 컨벤션

- (예: snake_case (Python), camelCase (TypeScript))
- (예: 새 API 엔드포인트 → Blueprint에 등록)
- (예: 새 DB 모델 → models/ 디렉토리에 추가)

## 도메인 지식 (포인터)

> "여기서부터 읽어라" — 필요할 때만 깊은 문서로 진입.

| 문서 | 역할 | 언제 읽나 |
|------|------|----------|
| `hdd/defect-catalog.md` | 도메인 함정 패턴 목록 | 구현 시작 전 **필독** |
| `hdd/design.md` | 변경 영향 규칙 + 의존 관계 | 설계 변경 or 외부 의존 변경 시 |
| `hdd/specs/<화면명>.md` | 화면별 구현 지시서 | 해당 화면 구현 시 |

## 아키텍처 불변식

> 이 규칙은 **기계적으로 강제**됩니다. 에이전트가 이를 위반하면 안 됩니다.

1. (예: AppMode별 쿼리 분기 — ADVANCED_CHAT은 messages만 읽음)
2. (예: 모든 집계 쿼리에 디버깅 필터 필수)
3. (예: RBAC 테이블은 sp_ 접두사 사용)

## 흔한 실수 → 방어

> `hdd/defect-catalog.md`에서 상세 내용 확인.

| 실수 | Harness ID | 한줄 방어 |
|------|-----------|----------|
| (예: 토큰 이중카운트) | H-DASH-01 | ADVANCED_CHAT은 messages만 |
| (예: 디버깅 데이터 혼입) | H-DASH-03 | invoke_from != 'debugger' |

## 구현 시 지시 형식

```
"hdd/specs/kpi-cards.md를 읽고 구현해줘.
 H-DASH-01, H-DASH-03을 반드시 방어하고
 각 Harness에 대한 단위 테스트도 함께 작성해줘."
```
