---
tags: [AI, RAG, AI-Agent, 개발]
---

## 개요

[Cognita](https://github.com/truefoundry/cognita)는 **TrueFoundry**가 만든 오픈소스 RAG 프레임워크로, 프로덕션 환경에서 바로 쓸 수 있는 **모듈형 RAG 애플리케이션**을 빠르게 구축할 수 있도록 설계되었다.

LangChain과 LlamaIndex 위에 올라가는 구조로, 각 컴포넌트를 독립적으로 교체·확장할 수 있다.

---

## 주요 특징

### 청킹 & 임베딩
- 문서 처리 자동화 (스케줄 업데이트 / 이벤트 트리거 방식 모두 지원)
- 배치 처리로 연산 부하 절감
- 이미 인덱싱된 문서는 추적해서 중복 처리 방지 (Incremental Indexing)

### 검색(Retrieval) 설정
- 유사도 검색 (Similarity Search)
- 쿼리 분해 (Query Decomposition) — 복잡한 질문을 서브쿼리로 분리
- 문서 재순위화 (Reranking)
- 청킹 방식, 검색 방식 모두 설정 가능 → 실험/튜닝이 쉽다

### 모듈형 아키텍처
| 컴포넌트 | 설명 |
|----------|------|
| Parsers | PDF, HTML 등 다양한 형식 파싱 |
| Data Loaders | 다양한 소스에서 데이터 수집 |
| Embedders | 사전학습 임베딩 모델 지원 |
| Vector DBs | Chroma, Qdrant, Weaviate, SingleStore 등 |

### UI & 프로덕션
- No-code UI 제공 → 코드 없이 RAG 설정 실험 가능
- 로컬 개발 환경과 프로덕션 환경 모두 지원
- GPT-4 기반 멀티모달 비전 파서 지원 (이미지 포함 문서 처리)

---

## Dify와의 비교

| 항목 | Cognita | Dify |
|------|---------|------|
| 목적 | RAG 파이프라인 전문 프레임워크 | 올인원 LLM 앱 빌더 |
| 청킹/검색 설정 | 세밀하게 커스터마이징 가능 | 기본 설정 제공 |
| 러닝커브 | 중간 (개발자 친화적) | 낮음 (노코드 우선) |
| 벡터DB | Qdrant, Weaviate 등 외부 연동 | 내장 지식베이스 |

---

## 관련 노트

- [[Milvus - 벡터 데이터베이스]]
- [[Dify - 멀티 에이전트 설계 패턴]]
