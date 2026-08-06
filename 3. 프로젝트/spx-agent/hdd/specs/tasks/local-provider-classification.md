---
tags: [프로젝트, dify, AI-Agent, HDD]
type: spec/tasks
screen: 로컬 모델 "(로컬)" 라벨 namespaced 버그 fix (LOCAL_PROVIDERS 정규화)
harness: [H-DASH-09]
date: 2026-06-10
last_updated: 2026-06-10
---
# 로컬 모델 판별 (LOCAL_PROVIDERS 정규화) — Tasks

> **목적**: 모델 차원 차트 "(로컬)" 라벨이 운영에서 0건 붙는 버그를 잡는다. 원인 = `model_provider`가 플러그인 경로 형식(`langgenius/ollama/ollama`)인데 `LOCAL_PROVIDERS`는 bare 이름(`ollama`)이라 미스매치.
> **근거 조사**: [[3. 프로젝트/spx-agent/references/local-provider-classification.md]] (신호 비교·DB 실증·권고안 전문)
> **결함**: H-DASH-09 ([[3. 프로젝트/spx-agent/hdd/defect-catalog.md]])
> **범위 원칙**: 이름 정규화 + frozenset 확장 + 공유 상수화 + 테스트 시드 교체. endpoint 복호화·DB allowlist는 범위 밖(필요 시 후속).

## 시작 전 컨텍스트 (신규 세션 필독)

- **코드 repo 루트**: `C:\Users\Administrator\Projects\spx-agent` (베이스 `dev`). 이 문서의 모든 상대경로(`api/...`)는 이 루트 기준. 볼트(`3. 프로젝트/spx-agent/`)에는 하네스 문서만 있음.
- **브랜치**: `KAN-29-admin-dashboard`에서 작업 (모델 차트 트랙과 동일 브랜치).
- **별도 트랙**: 이 fix는 B안(워크플로우 모델 분류, `194 복구 대기`)과 **무관하게 지금 칠 수 있음**. 159 운영 Dify로 형식 검증 완료, 194 복구 안 기다려도 됨.
- **착수 전 PM 사인 필수**: 아래 § PM 결정 4건. 사인 전 코드 변경 금지.

## ⚠️ 핵심 사실 (재조사 불필요 — 2026-06-10 확정)

1. **mart `model_provider` = 플러그인 경로 형식** (`langgenius/ollama/ollama`, `yangyaofei/vllm/vllm`). 코드 체인이 변환 없는 pass-through라 Dify `messages.model_provider` 형식 그대로 내려옴 (reference §4.6, 체인 근거 H-DASH-09 노트).
2. **DB에 로컬 직접 플래그 없음** — 이름 기반 판별이 사실상 유일한 즉시 가능 신호.
3. **사용처 2곳, 중복 정의**: `api/services/admin/dashboard_model_tokens_service.py:19` `LOCAL_PROVIDERS` (`:58` `get_model_tokens()`) / `api/services/admin/dashboard_drill_calls_service.py:49` `_LOCAL_PROVIDERS` (`:82` `get_model_call_share()`).
4. **현재 시드/테스트가 bare** → green이지만 운영 버그 못 잡는 맹점. 시드 교체가 fix의 필수 일부.

## 채택안 — 방법 A (세그먼트 추출 + frozenset 매칭)

```python
LOCAL_PROVIDERS = frozenset({"ollama", "xinference", "localai", "vllm"})  # vllm 추가

def _is_local_provider(provider_path: str) -> bool:
    """플러그인 경로(author/plugin/provider) 마지막 세그먼트 추출 후 매칭.
    bare 이름이면 원본 그대로 반환되어 두 형식 모두 흡수."""
    short_name = provider_path.rsplit("/", 1)[-1] if provider_path else ""
    return short_name in LOCAL_PROVIDERS
```

> **방법 A 채택 사유**: `rsplit("/",1)[-1]`은 경로면 마지막 세그먼트를, bare면 원본을 반환 → mart 형식이 경로/bare 무엇이든 동작(형식 불확실성에 강건). 방법 B(전체 경로 allowlist)는 서드파티 author 변경 시 깨짐.

## 구현 체크리스트

### 1단계 — 공유 상수 + 정규화 헬퍼 신설
- [ ] [PM 사인 후] 공유 모듈에 `LOCAL_PROVIDERS` frozenset + `_is_local_provider()` 헬퍼 1곳 정의 (위치: `api/services/admin/` 공통 모듈 또는 기존 상수 모듈 — architecture.md 확인). `vllm` 포함.
- [ ] frozenset 최종 범위는 PM 결정 #1 반영 (vllm 확정, openllm/TGI 등은 결정 따름).

### 2단계 — 사용처 2곳 repoint
- [ ] `dashboard_model_tokens_service.py:19` — 로컬 정의 제거, 공유 헬퍼 import. `:58` 판별부를 `_is_local_provider(provider)`로 교체.
- [ ] `dashboard_drill_calls_service.py:49` — `_LOCAL_PROVIDERS` 로컬 정의 제거, 공유 헬퍼 import. `:82` 판별부 교체.
- [ ] grep 게이트: `grep -rn "LOCAL_PROVIDERS\s*=" api/services/admin/` → 정의 **1곳만** 남는지 확인(중복 재발 방지).

### 3단계 — 테스트 시드 namespaced 교체 (맹점 제거 — 필수)
- [ ] 모델 라벨 관련 시드/픽스처의 provider 값을 **운영과 동일한 namespaced 형식**으로 교체 (`ollama` → `langgenius/ollama/ollama`, vllm은 `yangyaofei/vllm/vllm`).
- [ ] 회귀 테스트 추가: namespaced provider에 "(로컬)" 라벨이 붙는지 (bare로 통과하던 기존 케이스가 namespaced에서도 green인지).
- [ ] `openai_api_compatible`은 "(로컬)" 미표시로 남는지 명시 케이스 1개 (방침 = 미표시 감수).

### 4단계 — 검증
- [ ] pytest 해당 서비스 테스트 통과.
- [ ] (선택) mart 1행 SELECT로 실제 `model_provider` 형식 눈 확인 (159 또는 194 복구 후) — PM 결정 #3 최종 닫기.

## PM 결정 필요 4건 (권고안 §9 — 사인받을 것)

1. **`LOCAL_PROVIDERS` 추가 범위** — vllm 확정 여부 / openllm·TGI 등 추가 여부.
2. **매칭 방식** — 방법 A(세그먼트 추출, 권장) vs 방법 B(전체 경로 allowlist).
3. **mart `model_provider` 형식 확인** — 코드 체인상 플러그인 경로로 확정되나, 운영/194 실데이터 1행으로 최종 확인.
4. **openai_api_compatible 처리** — 미표시 감수(권장) vs DB/env allowlist 도입.

## 범위 밖 (후속 트랙)
- endpoint URL 복호화(사설 IP 판별) — 정확하나 RSA 복호화 + 캐시 비용 큼. 비채택.
- DB/env 기반 allowlist — openai_api_compatible이 실제 문제될 때 승격 검토.
- SESSION_HISTORY 결정 박제 — PM 사인 + 구현 머지 후.
