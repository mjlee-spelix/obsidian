---
tags: [지식, n8n, 시각화, 워크플로우]
date: 2026-03-25
---
# n8n - 코드 노드 환경 제약 및 시각화

## 핵심
- 코드 노드 반환값은 반드시 **객체의 배열** `[{ "json": { ... } }]` 형태여야 함
- Self-hosted 환경에서 Python 노드 사용 시 Python 3가 컨테이너에 설치되어 있어야 함
- `matplotlib`, `Chart.js` 모두 사용 불가 (Python runner 없음)
- 차트 결과물은 주로 이미지 파일 또는 HTML 리포트 형태

## 상세

### 환경 제약

| 문제 | 원인 | 해결 |
|---|---|---|
| `Code doesn't return items properly` | 단일 객체 반환 | `[{ "json": {...} }]` 배열 형태로 반환 |
| Python 노드 에러 | 컨테이너에 Python 3 미설치 | Docker 이미지에 Python 3 추가 |
| 외부 모듈 import 불가 | CommonJS 제약 또는 환경 변수 미설정 | `NODE_FUNCTION_ALLOW_EXTERNAL` 환경 변수 설정 또는 `require` 사용 |

### 시각화 구현 방법

#### 데이터 가공
- 여러 행의 데이터를 하나의 차트로 묶으려면 `items`를 루프 돌며 `labels`, `data` 배열로 재조립 필요

#### 이미지 처리 흐름
1. `HTTP Request` 노드로 외부 차트 엔진(QuickChart 등)에서 이미지를 바이너리로 가져옴
2. 이메일 첨부 / Slack 전송 / Google Drive 저장 등 다음 자동화 단계로 연결

#### 제약 사항
- n8n UI 자체는 차트 렌더링 도구가 아님
- 인터랙티브 차트 불가 → 이미지 파일 또는 HTML 리포트로 출력

## 관련 노트
- [[Dify - 코드 노드 시각화 구현]]
