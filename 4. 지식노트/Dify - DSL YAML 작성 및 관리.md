---
tags: [지식, dify, dsl, yaml, workflow, chatflow]
date: 2026-04-06
---
# Dify - DSL YAML 작성 및 관리

## 핵심
- Dify 앱은 `.yml` 파일로 export/import가 가능한 DSL 구조를 가진다
- 직접 YAML을 작성하는 것보다 **UI에서 빈 앱을 export한 결과를 템플릿으로 삼는 것**이 가장 빠르고 안전하다
- DSL 버전(`version` 필드)은 Dify 인스턴스 버전에 종속적이므로 현재 사용 중인 Dify의 export 결과를 기준으로 작성해야 한다
- 노드 ID는 문자열 형태의 타임스탬프(밀리초), 위치값(`position`, `width`, `height`)은 UI에서 자동 조정되므로 대충 두어도 된다

## 상세

### 최상위 구조

```yaml
app:
  name: Sales Analysis Agent
  description: 매출 분석 전문 에이전트
  icon: 🤖
  icon_background: '#FFEAD5'
  mode: advanced-chat          # chat / advanced-chat / workflow / agent-chat
  use_icon_as_answer_icon: false
kind: app
version: 0.1.5                 # Dify 버전에 따라 다름
workflow:
  environment_variables: []
  features:
    file_upload: { enabled: false }
    opening_statement: ''
    suggested_questions: []
  graph:
    nodes: [...]
    edges: [...]
    viewport: { x: 0, y: 0, zoom: 1 }
  conversation_variables: []
```

### mode 종류

| mode | 설명 |
|------|------|
| `chat` | 기본 챗봇 (프롬프트 오케스트레이션) |
| `advanced-chat` | 챗플로우 (그래프 기반 대화형) |
| `workflow` | 워크플로우 (단발성 실행) |
| `agent-chat` | 에이전트 (도구 사용 대화형) |

### 노드 공통 구조

```yaml
- id: '1712000000000'          # 문자열 형태의 타임스탬프 (unique)
  type: custom                 # 거의 항상 custom
  position: { x: 80, y: 282 }
  positionAbsolute: { x: 80, y: 282 }
  width: 244
  height: 54
  selected: false
  sourcePosition: right
  targetPosition: left
  data:
    type: start                # ★ 여기가 실제 노드 타입
    title: 시작
    desc: ''
    # ... 노드 타입별 필드
```

### 주요 노드 타입별 예시

**Start 노드** (입력 변수 정의):
```yaml
data:
  type: start
  title: 시작
  variables:
    - variable: query
      label: 질문
      type: text-input
      required: true
      max_length: 500
```

**LLM 노드**:
```yaml
data:
  type: llm
  title: Analysis Agent
  model:
    provider: anthropic
    name: claude-sonnet-4-5
    mode: chat
    completion_params:
      temperature: 0.3
      max_tokens: 4096
  prompt_template:
    - role: system
      text: |
        너는 매출 분석 전문가다...
    - role: user
      text: '{{#sys.query#}}'
  context:
    enabled: false
    variable_selector: []
  vision: { enabled: false }
  structured_output_enabled: false
```

**Code 노드** (Python):
```yaml
data:
  type: code
  title: JSON 파싱
  code_language: python3
  code: |
    def main(text: str) -> dict:
        import json
        data = json.loads(text)
        return {
            'chart': data.get('chart', False),
            'chartType': data.get('chartType', ''),
            'chartData': json.dumps(data.get('chartData', {})),
        }
  variables:
    - variable: text
      value_selector: ['1712000000000', 'text']   # [node_id, output_key]
  outputs:
    chart: { type: boolean, children: null }
    chartType: { type: string, children: null }
    chartData: { type: string, children: null }
```

**Tool 노드** (서브 워크플로우 호출):
```yaml
data:
  type: tool
  title: NL2SQL
  provider_id: workflow
  provider_type: workflow
  provider_name: nl2sql-tool
  tool_name: nl2sql
  tool_label: NL2SQL
  tool_configurations: {}
  tool_parameters:
    question:
      type: mixed
      value: '{{#sys.query#}}'
```

**IF/ELSE 노드**:
```yaml
data:
  type: if-else
  title: 이메일 분기
  logical_operator: and
  conditions:
    - id: 'cond-1'
      variable_selector: ['code_node_id', 'email']
      comparison_operator: is
      value: 'true'
```

**Answer 노드** (챗플로우 전용):
```yaml
data:
  type: answer
  title: 답변
  answer: '{{#llm_node_id.text#}}'
```

**Agent 노드** (도구 사용 에이전트):
```yaml
data:
  type: agent
  title: Sales Agent
  agent_strategy_provider_name: langgenius/agent/agent
  agent_strategy_name: function_calling
  agent_parameters:
    model:
      value:
        provider: anthropic
        model: claude-sonnet-4-5
        mode: chat
        completion_params: { temperature: 0.3 }
    tools:
      value:
        - provider_name: workflow/nl2sql-tool
          tool_name: nl2sql
        - provider_name: workflow/chart-tool
          tool_name: chart
    instruction:
      value: |
        너는 매출 분석 에이전트다...
    query:
      value: '{{#sys.query#}}'
    maximum_iterations:
      value: 5
```

### 엣지 구조

```yaml
edges:
  - id: 'edge-1'
    source: '1712000000000'    # 시작 노드 ID
    target: '1712000000001'    # 대상 노드 ID
    sourceHandle: source       # IF/ELSE는 'true'/'false'
    targetHandle: target
    type: custom
    data:
      sourceType: start
      targetType: llm
      isInIteration: false
```

## 작업 팁

### 1. 템플릿 먼저 확보 (가장 현실적)
- Dify UI에서 빈 챗플로우 하나 만들고 우상단 `...` → `DSL 내보내기`
- 받은 YAML을 프로젝트에 두고 수정
- 노드 ID, `version` 번호 등이 현재 Dify 인스턴스와 호환되는 값임을 보장할 수 있음

### 2. 편집 시 주로 건드리는 부분
Dify DSL에서 사람이 자주 수정하는 부분은 제한적이다:
- **프롬프트 텍스트** (`prompt_template` 안의 `text`)
- **code 노드의 Python 코드**
- **모델 파라미터** (`model`, `completion_params`)
- **노드 연결** (`edges`)

위치값은 Dify UI에서 자동 조정되므로 손댈 필요 없음.

### 3. ID 관리 규칙
- 노드 ID: **문자열 형태의 타임스탬프(밀리초)** 사용 — `'1712345678901'`
- 엣지 ID: unique하면 뭐든 OK (`'edge-1'`, `'llm2tool'` 등)
- 같은 워크플로우 안에서 중복만 피하면 됨

### 4. 버전 호환성 주의
- `version` 필드는 Dify 버전에 따라 다름 (0.1.x / 0.3.x / 0.4.x ...)
- 다른 버전에서 만든 DSL을 그대로 import하면 에러 발생 가능
- 현재 사용 중인 Dify 버전의 export 결과를 기준으로 작성하는 것이 안전

### 5. 권장 프로젝트 구조

```
project-root/
  dify/
    sales-analysis-json.yml    # JSON 모드 챗플로우
    sales-analysis-md.yml      # MD 모드 챗플로우
    nl2sql-tool.yml            # NL2SQL 서브 워크플로우
    chart-tool.yml             # Chart 서브 워크플로우
```

- 로컬에서 편집 → Dify UI에서 import(덮어쓰기 또는 신규 생성)
- git으로 버전 관리하면 워크플로우 이력 추적 가능

### 6. 배포 경로
| 방법 | 특징 |
|------|------|
| Dify UI 수동 import | 가장 안전, 테스트·반영에 적합 |
| API 호출 (`/console/api/apps/import`) | CI/CD나 자동 배포 시 유용 |

### 7. 서브 워크플로우 도구 동기화 주의
- YAML로 서브 워크플로우를 import해도, 부모 챗플로우가 그걸 도구로 참조하려면 **"도구로서의 워크플로우" 설정을 다시 저장**해야 버전이 동기화됨
- 관련: [[Dify - 워크플로우 도구 버전 동기화]]

## 관련 노트
- [[Dify - 워크플로우 도구 버전 동기화]]
- [[Dify - 멀티 에이전트 설계 패턴]]
- [[Dify - 코드 노드 시각화 구현]]
- [[Dify - SSE vs Blocking 응답 모드]]
