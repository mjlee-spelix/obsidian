---
tags: [템플릿, HDD]
---
# 화면별 Spec 템플릿 (3파일 구조)

> Spec은 **화면 단위로 3개 파일**로 분리합니다.
> 각 파일은 `hdd/specs/` 하위 폴더에 동일한 파일명으로 생성합니다.

## 폴더 구조

```
hdd/specs/
├── requirements/{{screen_name}}.md   ← 무엇을 만들지 (비즈니스 규칙 + Harness)
├── design/{{screen_name}}.md         ← 어떻게 만들지 (API + 쿼리 + 컴포넌트)
└── tasks/{{screen_name}}.md          ← 어떤 순서로 만들지 (체크리스트)
```

## 각 파일 역할

| 파일 | 에이전트가 할 수 있게 되는 것 | 사람이 판단하는 것 |
|------|--------------------------|-----------------|
| **requirements** | 비즈니스 규칙 이해, 어떤 Harness를 방어할지 파악 | 이 화면에서 무엇이 함정인가 |
| **design** | 파일 위치, 쿼리 작성, 타입 정의 | API 구조, 쿼리 전략 |
| **tasks** | 구현 순서, 완료 체크 | 우선순위, 의존 순서 |

---

## 1. requirements/{{screen_name}}.md

```markdown
---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/requirements
screen: {{screen_name}}
harness: []
date: {{date}}
---
# {{screen_name}} — Requirements

> 관련: [[specs/design/{{screen_name}}.md|Design]] · [[specs/tasks/{{screen_name}}.md|Tasks]]

## 화면 요구사항

(이 화면이 무엇을 보여주는지, 레이아웃/구성요소를 서술)

| 구성요소 | 주 수치 | 부가 정보 |
|---------|--------|----------|
| | | |

## 기간/필터

- (이 화면에서 사용하는 파라미터: 기간, 부서, 필터 등)

## 비즈니스 규칙

**구성요소 1:**
- 데이터 출처: (테이블명, 컬럼)
- 집계 방식: (COUNT, SUM, DISTINCT 등)
- 필터 조건: (필수 WHERE 조건)

**공통 규칙:**
- (모든 구성요소에 적용되는 규칙)

## 방어할 Harness

| ID | 결함 | 이 화면에서의 방어 |
|----|------|---------------|
| H-XXX-XX | (결함 요약) | (방어 방법) |

## 관련 노트

- [[defect-catalog.md|Defect Catalog]]
```

---

## 2. design/{{screen_name}}.md

```markdown
---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/design
screen: {{screen_name}}
harness: []
date: {{date}}
---
# {{screen_name}} — Design

> 관련: [[specs/requirements/{{screen_name}}.md|Requirements]] · [[specs/tasks/{{screen_name}}.md|Tasks]]

## API 엔드포인트

\```
GET /console/api/admin/...
Query Parameters:
  - param1: type (설명)
Authorization: admin 권한 필요
\```

## Response 타입

\```typescript
interface XxxResponse {
  // 응답 구조 정의
}
\```

## 쿼리 설계

**구성요소 1:**
\```sql
SELECT ...
FROM ...
WHERE ...;
\```

## 서비스 레이어

\```python
def some_logic():
    pass
\```

## 프론트엔드 컴포넌트

\```
feature/
├── page.tsx
├── components/
│   └── xxx.tsx
└── hooks/
    └── use-xxx.ts
\```

## 관련 노트

- [[design.md|HDD 상세 설계]]
- (관련 reference 링크)
```

---

## 3. tasks/{{screen_name}}.md

```markdown
---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: {{screen_name}}
harness: []
date: {{date}}
---
# {{screen_name}} — Tasks

> 관련: [[specs/requirements/{{screen_name}}.md|Requirements]] · [[specs/design/{{screen_name}}.md|Design]]

## 백엔드

- [ ] 1. (서비스 클래스 생성)
- [ ] 2. (집계 메서드 구현)
- [ ] 3. (API 엔드포인트 등록)

## 프론트엔드

- [ ] 4. (TanStack Query 훅 작성)
- [ ] 5. (컴포넌트 구현)
- [ ] 6. (로딩/에러 상태 처리)

## 테스트

- [ ] 7. H-XXX-XX: (테스트 설명)
- [ ] 8. H-XXX-XX: (테스트 설명)

## 관련 노트

- [[specs/requirements/{{screen_name}}.md|Requirements]]
- [[specs/design/{{screen_name}}.md|Design]]
```
