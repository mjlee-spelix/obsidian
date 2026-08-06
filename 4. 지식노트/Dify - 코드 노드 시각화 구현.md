---
tags: [지식, dify, 시각화, 차트]
date: 2026-03-25
---
# Dify - 코드 노드 시각화 구현

## 핵심
- 코드 노드(JS/Python)는 샌드박스에서 실행되어 `Chart.js`, `matplotlib` 등 렌더링 라이브러리 직접 사용 불가
- 차트 설정값(JSON)을 생성하거나 외부 API를 호출하는 방식으로 우회해야 함
- 코드 노드 output에 `Files` 타입 없음 → 이미지를 직접 반환 불가

## 상세

### 방법 1. QuickChart API (가장 권장)
- 코드 노드에서 차트 설정을 JSON으로 만든 후 `encodeURIComponent`로 URL 생성
- Answer 노드에서 마크다운으로 출력
  ```
  ![그래프]({{url}})
  ```

### 방법 2. Dify Marketplace 플러그인
- `Digitforce Data Analysis`: 자연어 질문으로 차트 자동 생성, 내부적으로 JSON → 인터랙티브 차트 렌더링
- `json2chart`: JSON 데이터를 차트 이미지로 변환
- `base64_codec`: 이미지 데이터를 실제 파일 객체로 변환하여 대화창에 첨부 가능

### 방법 3. Base64 문자열 방식
- 코드 노드에서 이미지를 Base64 문자열로 변환 후 String 타입으로 반환
- Answer 노드에서 아래처럼 출력
  ```
  ![차트 이미지]({{코드노드_결과_변수}})
  ```
- **단점**: Base64 문자열이 길어지면 LLM context를 많이 소모함 (차트 해상도 조절 필요)

### 유용한 플러그인 목록
| 플러그인 | 설명 | 링크 |
|---|---|---|
| sql_polyglot | 쿼리 최적화/오류 검사 | https://marketplace.dify.ai/plugin/abesticode/sql_polyglot |
| markdown2html | MD → HTML 변환 | https://marketplace.dify.ai/plugin/migege/markdown2html |
| base64_codec | Base64 → 이미지 파일 변환 | https://marketplace.dify.ai/plugin/bowenliang123/base64_codec |
| email_html_pro | HTML 이메일 전송 | https://marketplace.dify.ai/plugin/abesticode/email_html_pro |
| json2chart | JSON → 차트 이미지 | https://marketplace.dify.ai/plugin/lfenghx/json2chart |
| data_analysis | 데이터 분석 시각화 | https://marketplace.dify.ai/plugin/digitforce/data_analysis |

## 관련 노트
- [[Dify - Sandbox 개념 및 matplotlib 설치]]
- [[Base64 인코딩]]
