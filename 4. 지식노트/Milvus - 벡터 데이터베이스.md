---
tags: [AI, RAG, 개발, CS]
---

## 개요

[Milvus](https://github.com/milvus-io/milvus)는 **Zilliz**가 주도하는 오픈소스 벡터 데이터베이스(Apache 2.0)로, LF AI & Data Foundation 소속 프로젝트다.

RAG, 이미지/텍스트 검색, 추천 시스템 등 **AI 애플리케이션의 벡터 저장 및 검색** 용도로 널리 쓰인다.

---

## 핵심 개념: 벡터 데이터베이스란?

텍스트·이미지 등을 임베딩 모델로 변환한 **고차원 벡터**를 저장하고, 의미적으로 유사한 벡터를 빠르게 찾아주는 데이터베이스.

RAG 파이프라인에서는 문서를 임베딩해서 Milvus에 저장하고, 질문이 들어오면 질문의 임베딩과 가장 가까운 문서 조각을 꺼내서 LLM에 넘긴다.

---

## 주요 특징

### 검색 능력
- **ANN(Approximate Nearest Neighbor) 검색**: 수십억 개 벡터에서도 빠른 검색
- Dense / Sparse 벡터 모두 지원
- 메타데이터 필터링 + 벡터 검색 **하이브리드 검색** 가능
- Multi-vector 검색 지원

### 인덱스 타입
| 인덱스 | 설명 |
|--------|------|
| HNSW | 그래프 기반, 정확도·속도 균형 우수 |
| IVF | 역파일 인덱스, 대규모 데이터에 적합 |
| PQ / SQ | 벡터 양자화로 메모리 절감 |
| CAGRA (GPU) | NVIDIA CUDA 기반 GPU 가속 인덱스 |

### 아키텍처
- Go + C++ 구현, **컴퓨팅과 스토리지 분리** 설계
- 읽기 부하 → Query Node, 쓰기 부하 → Data Node 개별 수평 확장 가능

### 배포 옵션
| 모드 | 용도 |
|------|------|
| Milvus Lite | 로컬 프로토타이핑 (파이썬 pip 설치) |
| Standalone | 소규모 프로덕션 |
| Distributed | 대규모 프로덕션 (K8s) |

---

## RAG 생태계 연동

LangChain, LlamaIndex, OpenAI, HuggingFace 등과 **공식 통합** 지원 → RAG 파이프라인에 바로 꽂아 쓸 수 있다.

---

## 경쟁 제품 비교

| 제품 | 특징 |
|------|------|
| **Milvus** | 오픈소스, 대규모, GPU 가속, 분산 |
| Qdrant | 오픈소스, Rust 기반, 빠름 |
| Weaviate | 오픈소스, 멀티모달 강점 |
| Pinecone | 완전 관리형 SaaS |
| Chroma | 로컬 개발용, 초경량 |

---

## 관련 노트

- [[Cognita - 모듈형 RAG 프레임워크]]
