---
tags: [프로젝트, dify, 차트, matplotlib]
date: 2026-03-27
---

## 현재 워크플로우 분석

### 매출 분석 (chatflow)

```
[시작] → [에이전트] → [코드: JSON 파싱] → [IF/ELSE] ─ TRUE ──→ [이메일 발송 도구] → [답변 2]
                                              │
                                              └─ FALSE ─→ [답변]
```

- 에이전트가 SQL 실행 후 JSON 응답 (chartType, chartData 포함)
- 코드 노드가 에이전트 텍스트에서 JSON 파싱 → `chart`, `chartType`, `chartData`, `email` 등 추출
- IF/ELSE 조건: `chart != true AND email == true` → 이메일 발송
- **현재 문제: chart=true여도 차트 이미지를 생성하는 단계가 없음**

### 이메일 발송 (workflow tool)

```
[시작] → [코드: HTML 생성] → [IF has_chart_image]
                                 ├─ TRUE → [Base64→Image] → [send email (첨부)]
                                 └─ FALSE → [send email (첨부 없음)]
```

- 입력: `email_to`, `email_subject`, `analysis_result`, `insights`, `further_analysis`, `chart_image`
- **chart_image (base64) 입력은 이미 준비되어 있음** → 전달만 하면 됨

### 데이터 흐름에서 빠진 조각

```
에이전트 → chartType + chartData (JSON 텍스트) → ??? → chart_base64 (PNG 이미지)
```

에이전트는 chartData를 JSON으로 잘 뽑아주지만, 이걸 **이미지로 변환하는 단계**가 없다.

---

## 설계: 차트 생성 노드 삽입

### 핵심 아이디어

**매출 분석 워크플로우의 JSON 파싱 코드 노드 바로 뒤에**, 차트 생성 코드 노드를 하나 추가한다.
이 노드는 `chart=true`일 때만 matplotlib로 PNG를 생성하고, `chart=false`면 빈 문자열을 통과시킨다.
별도 워크플로우나 에이전트 도구로 분리할 필요 없음 — 차트 생성은 기계적 변환이지 LLM 판단이 아니므로.

### 변경 후 매출 분석 워크플로우

```
[시작] → [에이전트] → [코드①: JSON 파싱] → [코드②: 차트 생성] → [IF/ELSE] ─ ...
                                                                     (기존과 동일)
```

**추가되는 노드: 코드②(차트 생성) 1개뿐.** 나머지 구조는 거의 그대로 유지.

### 코드② 차트 생성 노드 상세

**입력 변수:**

| 변수 | 소스 | 타입 |
|------|------|------|
| `chart` | 코드①.chart | boolean |
| `chartType` | 코드①.chartType | string |
| `chartData` | 코드①.chartData | string (JSON) |
| `json` | 코드①.json | string (전체 JSON) |

**로직:**
```
if chart == false:
    return { chart_base64: "", json: json }  # 그대로 통과

chart_base64 = matplotlib로 PNG 생성 → base64 인코딩
json에 chart_base64 필드 추가
return { chart_base64, json }
```

**출력 변수:**

| 변수 | 타입 | 설명 |
|------|------|------|
| `chart_base64` | string | `data:image/png;base64,...` 또는 빈 문자열 |
| `json` | string | chart_base64가 포함된 전체 JSON |

### IF/ELSE 조건 변경

현재 조건: `chart != true AND email == true`

변경 필요:

| 조건 | 분기 |
|------|------|
| `email == true` | → 이메일 발송 도구 → 답변 2 |
| ELSE | → 답변 |

**chart 조건 제거.** 차트 생성은 이미 코드②에서 처리 완료. IF/ELSE는 이메일 발송 여부만 판단.

### 이메일 발송 도구 호출 변경

현재 `chart_image` 파라미터에 빈 문자열을 전달하고 있음.

변경: `chart_image`에 **코드②의 chart_base64**를 전달.

```yaml
# 변경 전
chart_image:
  type: mixed
  value: ''

# 변경 후
chart_image:
  type: mixed
  value: '{{#코드②.chart_base64#}}'
```

이렇게 하면 이메일 발송 워크플로우는 **수정 없이** 기존 로직으로 차트 이미지를 첨부하게 됨.

### 답변 노드 변경

현재: `{{#코드①.json#}}`
변경: `{{#코드②.json#}}` (chart_base64가 포함된 JSON)

web_v2 프론트엔드는 JSON 안의 chart_base64를 꺼내서 `<img src>` 렌더링하거나,
기존처럼 chartData로 Chart.js 클라이언트 렌더링도 가능 (선택).

---

## 4가지 시나리오 흐름

### 1. chart=false, email=false (일반 질문)

```
에이전트 → 코드①(JSON) → 코드②(통과, chart_base64="") → IF/ELSE → FALSE → 답변
```

### 2. chart=true, email=false (차트 요청)

```
에이전트 → 코드①(JSON) → 코드②(matplotlib → base64) → IF/ELSE → FALSE → 답변(차트 포함 JSON)
```

### 3. chart=false, email=true (이메일 발송)

```
에이전트 → 코드①(JSON) → 코드②(통과) → IF/ELSE → TRUE → 이메일 발송(차트 없음) → 답변 2
```

### 4. chart=true, email=true (차트 + 이메일)

```
에이전트 → 코드①(JSON) → 코드②(matplotlib → base64) → IF/ELSE → TRUE → 이메일 발송(차트 첨부) → 답변 2
```

---

## 이메일 발송 워크플로우

**수정 불필요.** 이미 chart_image 입력과 Base64→Image 디코딩 로직이 구현되어 있음.
매출 분석 쪽에서 chart_base64를 chart_image 파라미터로 넘겨주기만 하면 됨.

---

## Dify 웹앱 (네이티브 채팅)에서의 표시

현재 답변은 raw JSON 출력 → web_v2 프론트가 파싱해서 렌더링하는 구조.
Dify 네이티브 채팅에서 차트를 직접 보여주려면 답변을 마크다운으로 바꿔야 함:

```markdown
{{분석 결과 텍스트}}

![차트](data:image/png;base64,...)

**인사이트:** ...
```

하지만 이건 web_v2와 답변 포맷이 충돌함.
→ **당장은 web_v2 JSON 포맷 유지**, Dify 웹앱에서는 JSON 안의 chart_base64로 표시하는 건 프론트 영역에서 처리.
→ 나중에 Dify 네이티브용과 web_v2용 답변을 분리하려면 별도 chatflow를 만들거나, 답변 노드를 2개 두는 방식 검토.

---

## 작업 체크리스트

### Dify 샌드박스 준비
- [ ] `python-requirements.txt`에 `matplotlib` 추가
- [ ] sandbox 컨테이너 재시작
- [ ] 코드 노드에서 `import matplotlib` 테스트

### 매출 분석 워크플로우 수정
- [ ] 코드②(차트 생성) 노드 추가 — 코드①과 IF/ELSE 사이에 삽입
- [ ] matplotlib 코드 작성 (plan-dify-chart.md 참고)
- [ ] IF/ELSE 조건 변경: `email == true` 단일 조건으로 수정
- [ ] 이메일 발송 도구의 `chart_image` 파라미터에 `코드②.chart_base64` 연결
- [ ] 답변 노드의 출력을 `코드②.json`으로 변경

### 테스트
- [ ] "국가별 매출을 차트로 보여줘" → chart_base64 포함 JSON 확인
- [ ] "국가별 매출 알려줘" → chart_base64 빈 문자열 확인
- [ ] "국가별 매출을 차트로 보여주고 이메일로 보내줘" → 이메일에 차트 첨부 확인

## 관련 노트

- [[n8n workflow 정리]]
- [[3. 프로젝트/web-ui/리뉴얼 설계]]
- 차트 코드 상세: `agent-sales-analysis/plan-dify-chart.md`
