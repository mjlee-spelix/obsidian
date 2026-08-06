---
tags:
  - 서연이화
  - 과제5
  - dify
  - 테스트
created: 2026-06-26
target: 3. 프로젝트/서연이화/과제5/대책서 작성 도우미 v0.0.3.yml
---

# Dify 테스트 케이스 — 대책서 작성 도우미 v0.0.3

> [!info] 사용법
> Dify 워크플로우 실행 시 `inputs.payload` (JSON object)에 아래 JSON을 붙여넣는다.
> 출력은 `data.outputs.result` 확인. 더미 풀이 도어트림/사출/조립 테마라 예시도 거기에 맞춤.
> 대상: [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 v0.0.3.yml]] · 결정 근거: [[3. 프로젝트/서연이화/과제5/RAG 독립 항목 확정 체크리스트.md]]

---

## 기대결과 요약표

| # | 케이스 | 핵심 확인 | 기대 결과 |
|---|---|---|---|
| T1 | 원인 기본 | 기본 5/5 | ok:true, items 10, meta 5/5 |
| T2 | 원인 개수변경 | internal3/external2 | items 5 (내3+외2) |
| T3 | 원인 재추천 | exclude 누적(A8) | 제외 2건 미출현 |
| T4 | 내부만(A9) | external=0 | 내부5, externalAdded=0 |
| T5 | 외부만 | internal=0 | 외부4, internalAdded=0 |
| T6 | 합 10 초과 | 외부 축소 | 내8+외2, requestedExternal=2 |
| T7 | exclude 정규화(A8) | 공백·중복·빈값 | 1건만 제외 처리 |
| T8 | 대책 기본 | 원인 1개 | 이 원인 대책 10 |
| T9 | 대책 재추천 | exclude 누적 | 제외 1건 미출현 |
| T10 | 에러: 현상 없음 | A3 | ok:false, MISSING_PHENOMENON |
| T11 | 에러: 원인 없음 | A3 | ok:false, MISSING_CAUSE |
| T12 | 에러: step 오류 | A3/A7 | ok:false, INVALID_STEP |
| T13 | 에러: 개수 0/0 | A3 | ok:false, INVALID_COUNT |

> 공통 확인: **에러 케이스에서 워크플로우가 죽지 않고 정상 종료**(끝·에러 노드 도달)되는지 = A3 핵심.

---

## ✅ 정상 — 원인 (step: cause)

### T1. 기본 5/5
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량(좌측 0.8mm 초과)" }
```

### T2. 개수 변경 (내부3/외부2)
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량", "internalCount": 3, "externalCount": 2 }
```

### T3. 재추천 — exclude 누적
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량", "exclude": ["설비 정렬(alignment) 편차로 인한 치수 산포", "작업 표준서 미준수 및 작업자 숙련도 편차"] }
```

---

## 🔸 경계 케이스

### T4. 내부만 (external=0) — A9
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량", "internalCount": 5, "externalCount": 0 }
```

### T5. 외부만 (internal=0)
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량", "internalCount": 0, "externalCount": 4 }
```

### T6. 합 10 초과 → 외부 축소
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량", "internalCount": 8, "externalCount": 5 }
```

### T7. exclude 정규화 (공백·중복·빈값) — A8
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량", "exclude": ["  금형/치공구 마모에 따른 형상 불량  ", "금형/치공구 마모에 따른 형상 불량", ""] }
```

---

## ✅ 정상 — 대책 (step: countermeasure)

### T8. 기본 (원인 1개)
```json
{ "step": "countermeasure", "phenomenon": "도어트림 상단 단차 불량", "cause": "금형/치공구 마모에 따른 형상 불량" }
```

### T9. 재추천 — exclude
```json
{ "step": "countermeasure", "phenomenon": "도어트림 상단 단차 불량", "cause": "금형/치공구 마모에 따른 형상 불량", "exclude": ["금형/치공구 마모 한계 관리 및 교체 주기 단축"] }
```

---

## ❌ 에러 — result.ok=false + errorCode (A3)

### T10. 현상 없음 → MISSING_PHENOMENON
```json
{ "step": "cause", "phenomenon": "" }
```

### T11. 대책인데 원인 없음 → MISSING_CAUSE
```json
{ "step": "countermeasure", "phenomenon": "도어트림 상단 단차 불량", "cause": "" }
```

### T12. 잘못된 step → INVALID_STEP
```json
{ "step": "solution", "phenomenon": "도어트림 상단 단차 불량" }
```

### T13. 개수 둘 다 0 → INVALID_COUNT
```json
{ "step": "cause", "phenomenon": "도어트림 상단 단차 불량", "internalCount": 0, "externalCount": 0 }
```

---

## 출력 형태 (참고)

**성공**
```json
{ "ok": true,
  "items": [ { "text": "...", "score": 88, "source": "internal", "ragSource": "[출처]..." } ],
  "meta": { "internalAdded": 5, "externalAdded": 5, "requestedInternal": 5, "requestedExternal": 5, "model": "gpt-5-nano", "partial": false } }
```
**에러**
```json
{ "ok": false, "errorCode": "MISSING_PHENOMENON", "message": "현상이 입력되어야 원인을 추천할 수 있습니다.", "items": [] }
```

> [!note] 한계(임포트 후 확인)
> - 모델 등록 필요: OpenAI 공급자 인증 + `gpt-5-nano` 토글 ON.
> - A9 완전 skip(비용0)·A10 노드 자체 실패 무시는 후속(버전기록 v0.0.3 참고).
> - `meta.partial=true`는 외부 요청(>0)인데 LLM이 빈 응답일 때만 — 정상 호출로는 재현 어려움.

---

## 관련 노트
- [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 v0.0.3.yml]]
- [[3. 프로젝트/서연이화/과제5/대책서 작성 도우미 버전 기록.md]]
- [[3. 프로젝트/서연이화/과제5/RAG 독립 항목 확정 체크리스트.md]]
