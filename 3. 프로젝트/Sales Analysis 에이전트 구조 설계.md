---
tags: [프로젝트, dify, agent, sales-analysis, nl2sql]
date: 2026-04-06
status: 설계중
---

## 배경

현재 Sales Analysis 에이전트 하나에 SQL 생성, 실행, 분석 서술, 차트 데이터 구성, 이메일 처리, JSON 조립까지 모든 역할이 몰려 있다.
프롬프트가 무거워지고, 후속 요청("차트로 그려줘", "메일 보내줘") 처리 시에도 불필요한 로직이 섞여 있다.

이 에이전트는 향후 **Master Agent의 서브 도구 중 하나**로 쓰일 예정이므로, 역할을 분리하고 재사용 가능한 구조로 재설계한다.

## 목표

- 역할을 명확히 분리해 프롬프트 경량화
- 후속 요청(차트/메일) 시 불필요한 SQL 재실행 방지 (메모리 활용)
- JSON 모드(Next.js UI용)와 MD 모드(Dify 웹앱용) 두 가지 출력 모드 지원
- Master Agent의 도구로 통합 가능한 구조

## 전체 위치

```
Master Agent
├── Sales Analysis Agent  ← 설계 대상
├── Agent 1 (다른 도메인)
└── Agent 2 (다른 도메인)
```

## Sales Analysis Agent 구조

```
Sales Analysis Agent
│
├── 역할
│   ├── 사용자 의도 분류 (신규 분석 / 차트 요청 / 메일 요청 / 복합)
│   ├── 메모리 관리 (이전 분석 결과 저장/조회 → 후속 요청 처리)
│   ├── 서브 도구 호출 순서 판단
│   └── 최종 응답 작성 (JSON 또는 MD)
│
├── 서브 도구
│   │
│   ├── [1] NL2SQL Tool (서브 워크플로우)
│   │   ├── 프롬프트: DB 스키마 + SQL 규칙 + 테이블 선택 가이드
│   │   ├── 도구: SQL Executor
│   │   ├── 입력: 사용자 질문 (자연어)
│   │   └── 출력: { sql, columns, raw_data, row_count }
│   │
│   ├── [2] Chart Tool (서브 워크플로우)
│   │   ├── LLM: raw_data → matplotlib 코드용 데이터 가공
│   │   ├── 코드 노드: matplotlib 실행 → base64 변환
│   │   ├── 입력: { raw_data, chart_type, title }
│   │   └── 출력: { base64_image }
│   │
│   └── [3] Email Tool (기존 서브 워크플로우 재사용)
│       ├── 텍스트 본문만 발송 (파일 첨부 미지원)
│       ├── 입력: { to, subject, body }
│       └── 출력: { success }
│
├── 출력 모드
│   │
│   ├── [JSON 모드] Next.js UI 연동용
│   │   ├── 분석 → analysis_result, insights, further_analysis
│   │   ├── 차트 → chartData JSON 반환 (UI가 렌더링)
│   │   │         Chart Tool 호출 안 함
│   │   ├── 메일 → Email Tool 직접 호출하여 발송
│   │   └── 출력 포맷:
│   │       {
│   │         user_request, analysis_result,
│   │         chart, chartType, chartData,
│   │         insights, further_analysis,
│   │         request_example
│   │       }
│   │
│   └── [MD 모드] Dify 웹앱용
│       ├── 분석 → 마크다운 문서로 직접 출력
│       ├── 차트 → Chart Tool 호출 → base64 이미지 출력
│       ├── 메일 → Email Tool 직접 호출하여 발송
│       └── 출력 포맷: 마크다운 텍스트 (+ 이미지)
│
└── 메모리
    ├── 저장 시점: NL2SQL 실행 후
    ├── 저장 내용: raw_data, analysis_result, insights
    └── 사용 시점: "차트로 그려줘", "메일 보내줘" 등 후속 요청
```

## 요청 유형별 호출 흐름

### ① 신규 분석

```
"2024년 매출 분석해줘"
  Sales Agent → NL2SQL → 결과 해석 → 응답
                                    → 메모리 저장
```

### ② 차트 후속 요청

```
"차트로 보여줘"
  Sales Agent → 메모리 로드 → [JSON] chartData 생성하여 응답
                             → [MD]  Chart Tool → 이미지 응답
```

### ③ 메일 후속 요청

```
"메일로 보내줘 abc@test.com"
  Sales Agent → 메모리 로드 → Email Tool → "발송 완료" 응답
```

### ④ 분석 + 차트 복합

```
"분석하고 차트도 그려줘"
  Sales Agent → NL2SQL → 결과 해석
                        → [JSON] chartData 포함 응답
                        → [MD]  Chart Tool → 텍스트 + 이미지 응답
                        → 메모리 저장
```

### ⑤ 분석 + 메일 복합

```
"분석하고 메일로 보내줘 abc@test.com"
  Sales Agent → NL2SQL → 결과 해석 → Email Tool → 응답
                                               → 메모리 저장
```

## JSON 출력 필드 변경점

```
기존                          변경 후
─────────────────────         ─────────────────────
user_request          →      user_request
analysis_result       →      analysis_result
chart                 →      chart
chartType             →      chartType
chartData             →      chartData
insights              →      insights
further_analysis      →      further_analysis
request_example       →      request_example
email                 ✕      삭제 (Sales Agent가 직접 발송)
email_to              ✕      삭제
email_subject         ✕      삭제
```

**근거:** Dify 이메일 전송 노드가 파일 첨부를 지원하지 않으므로, 차트 첨부 메일은 애초에 지원 대상이 아님.
따라서 UI에 위임할 이유가 없고, 모든 메일은 Sales Agent가 Email Tool을 직접 호출하여 처리한다.

## 프롬프트 분배

| 항목 | 기존 | 변경 후 |
|------|------|---------|
| DB 스키마 | 메인 프롬프트 | NL2SQL Tool |
| SQL 규칙 | 메인 프롬프트 | NL2SQL Tool |
| 테이블 선택 가이드 | 메인 프롬프트 | NL2SQL Tool |
| 출력 포맷 규칙 | 메인 프롬프트 | Sales Agent |
| 차트 규칙 (JSON) | 메인 프롬프트 | Sales Agent (chartData 구성) |
| 차트 규칙 (MD) | 메인 프롬프트 | Chart Tool (matplotlib) |
| 이메일 규칙 | 메인 프롬프트 | Sales Agent (주소 추출/판단) + Email Tool (발송) |
| 인사이트/분석 규칙 | 메인 프롬프트 | Sales Agent |

## Chatflow vs Workflow 선택

| 컴포넌트 | 타입 | 이유 |
|---------|------|------|
| Master Agent | **Chatflow** | 사용자 대면, 대화 메모리, 멀티턴 |
| Sales Analysis | **Chatflow** | 자체 메모리 필요(후속 요청), Dify 웹앱에서 직접 사용도 가능 |
| NL2SQL Tool | **Workflow** | 순수 함수 (질문→데이터), stateless |
| Chart Tool | **Workflow** | 순수 함수 (데이터→이미지) |
| Email Tool | **Workflow** | 순수 액션 (내용→발송) |

**판단 기준:**
- Chatflow → 대화 메모리/멀티턴/사용자 대면이 필요한 경우
- Workflow → stateless한 단일 입출력, 도구로만 호출되는 경우

**메모리 스코프 주의:**
Chatflow를 도구로 호출하면 각 호출이 독립 대화로 취급될 수 있음.
- Next.js UI 경로(JSON): Master 메모리에 raw_data 저장 → Sales Analysis 호출 시 파라미터로 전달
- Dify 웹앱 경로(MD): Sales Analysis를 직접 호출, 자체 Conversation Variables로 메모리 관리

## 주요 결정사항

| 이슈 | 원인/배경 | 결정 | 관련 노트 |
|------|-----------|------|-----------|
| 에이전트 역할 과중 | SQL+분석+차트+메일+JSON 조립이 한 에이전트에 몰림 | NL2SQL / Chart / Email을 서브 도구로 분리 | |
| Chart/Email 서브에이전트 vs 서브 도구 | 둘 다 가능하나 LLM 판단이 제한적 | 서브 워크플로우(도구)로 분리하되, 호출 판단은 Sales Agent가 수행 | |
| JSON/MD 모드 구현 방식 | 출력 포맷만 다르고 나머지 로직은 동일 | 같은 서브 도구 공유, 메인 프롬프트만 분기 (또는 챗플로우 2개) | [[Dify 차트 시각화 설계]] |
| 차트 첨부 메일 | Dify 메일 노드가 첨부 미지원 | 지원하지 않음. email 플래그 자체를 JSON에서 제거 | |
| 후속 요청 처리 | "차트로 그려줘", "메일 보내줘" 시 SQL 재실행 낭비 | 메모리에 raw_data + analysis_result 저장, 후속 요청 시 로드만 수행 | |
| Master Agent 연동 | 이 에이전트가 Master Agent의 도구로 쓰일 예정 | Sales Analysis는 독립적으로 동작하는 단위 에이전트로 유지 | |
| Chatflow vs Workflow | 각 컴포넌트가 메모리/사용자 대면이 필요한지 여부 | Master·Sales Analysis는 Chatflow, NL2SQL·Chart·Email은 Workflow | [[Dify - DSL YAML 작성 및 관리]] |

## 진행 상황

- [x] 구조 설계
- [ ] NL2SQL Tool 프롬프트 작성
- [ ] Chart Tool 워크플로우 구현 (matplotlib 샌드박스 준비)
- [ ] Email Tool 연결 (기존 워크플로우 재사용)
- [ ] Sales Agent 메인 프롬프트 작성 (JSON 모드)
- [ ] Sales Agent 메인 프롬프트 작성 (MD 모드)
- [ ] 메모리 관리 구현
- [ ] 통합 테스트 (5가지 요청 유형)

## 관련 노트

- [[Dify 차트 시각화 설계]]
- [[Dify - DSL YAML 작성 및 관리]]
- [[Dify - 멀티 에이전트 설계 패턴]]
- [[3. 프로젝트/web-ui/리뉴얼 설계 v2]]
- [[n8n workflow 정리]]
