---
tags: [지식, dify, workflow, tool, publishing]
date: 2026-03-26
---
# Dify - 워크플로우 도구 버전 동기화

## 핵심
- 자식 워크플로우를 게시(publish)하면 새 버전이 생성됨
- 부모 챗플로우에서 이 워크플로우를 도구로 사용하는 경우, **자식의 "도구로서의 워크플로우" 설정을 다시 저장**해야 최신 버전에 연결됨
- 입력 파라미터가 변경되지 않았더라도 버전 동기화를 위해 재저장 필요
- 재저장하지 않으면 이전 게시 버전의 워크플로우가 호출될 수 있음

## 상세

### 도구 설정이 저장하는 것
"도구로서의 워크플로우" 저장 시 DB(`tool_workflow_providers` 테이블)에 기록되는 항목:

| 필드 | 설명 |
|------|------|
| `name`, `label`, `description` | 도구 이름, 라벨, 설명 |
| `parameter_configuration` | 입력 파라미터 스키마 (JSON) |
| `icon` | 도구 아이콘 |
| **`version`** | **저장 시점의 워크플로우 게시 버전 (타임스탬프 기반)** |

### 버전 핀닝 메커니즘
Dify 소스 코드(`workflow_tools_manage_service.py`)에서 확인:

```python
# 도구 생성 시
version=workflow.version

# 도구 업데이트 시
workflow_tool_provider.version = workflow.version

# 동기화 상태 확인
"synced": workflow.version == db_tool.version
```

### 업데이트 순서
자식 워크플로우를 수정한 후 부모에 반영하려면:

1. 자식 워크플로우 수정
2. 자식 워크플로우 **게시** → 새 `version` 생성
3. 자식의 "도구로서의 워크플로우" 설정 **저장** → `version` 갱신
4. 부모 챗플로우 **게시** (필요한 경우)

### 주의사항
- 3번을 빠뜨리면 도구가 이전 버전을 참조하여 구버전 워크플로우가 실행될 수 있음
- 입력/출력 스키마가 동일해도 버전 불일치가 발생함
- Dify UI에서 `synced` 플래그로 동기화 상태를 확인할 수 있음

## 관련 노트
- [[Dify - 이메일 노드 Convert to HTML 설정]]
- [[Dify - SSE vs Blocking 응답 모드]]
