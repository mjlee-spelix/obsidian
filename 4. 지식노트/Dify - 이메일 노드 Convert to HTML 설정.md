---
tags: [지식, dify, email, plugin]
date: 2026-03-26
---
# Dify - 이메일 노드 Convert to HTML 설정

## 핵심
- Dify 이메일 전송 플러그인(`langgenius/email`)의 `convert_to_html` 설정은 **반드시 True**로 켜야 함
- 이 설정은 두 가지를 동시에 제어함:
  1. 마크다운 → HTML 변환 여부
  2. 이메일 Content-Type 헤더 (`text/html` vs `text/plain`)
- False로 끄면 `Content-Type: text/plain`이 되어 HTML 태그가 그대로 텍스트로 표시됨
- 코드 노드에서 직접 HTML을 생성해도 `convert_to_html: true`로 해야 `text/html`로 발송됨

## 상세

### convert_to_html 동작

| 설정 | Content-Type | 변환 | 결과 |
|------|-------------|------|------|
| `true` | `text/html` | 마크다운 → HTML 변환 적용 | HTML 메일로 정상 렌더링 |
| `false` | `text/plain` | 변환 없음 | HTML 태그가 텍스트로 노출 |

### "raw HTML + text/html" 조합은 없음
- 코드 노드에서 HTML을 직접 만들어도 `convert_to_html: true`로 설정해야 함
- 마크다운 변환기가 이미 HTML인 내용을 다시 처리하지만, 단순 HTML은 대부분 통과됨
- `false`로 하면 HTML이 plain text로 보내져서 태그가 그대로 보임

### 삽질 기록
1. `convert_to_html: true` + 코드 노드 HTML → 정상 작동
2. `convert_to_html: false` + 코드 노드 HTML → 실패 (태그가 텍스트로 표시)
3. 독립 테스트에서는 잘 되는데 도구로 호출하면 안 되는 경우 → [[Dify - 워크플로우 도구 버전 동기화]] 참고

### 이메일 노드 내부 버그 주의
- 파일 첨부(attachments) 필드를 한 번이라도 설정하면, UI에서 비워도 내부에 `attachments.type: variable` 설정이 남을 수 있음
- Pydantic 검증 에러 발생: `Value error, value must be a list`
- 해결: 노드를 삭제하고 다시 생성

## 관련 노트
- [[Dify - 워크플로우 도구 버전 동기화]]
- [[Dify - SSE vs Blocking 응답 모드]]
