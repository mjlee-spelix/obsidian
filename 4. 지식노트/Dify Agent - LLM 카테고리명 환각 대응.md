---
tags: [지식, dify, AI-Agent, LLM, AI]
date: 2026-03-26
---
# Dify Agent - LLM 카테고리명 환각 대응

## 핵심
- 사용자가 "자전거"라고 입력하면 LLM이 DB의 실제 값 `Bikes` 대신 `Bicycle` 등 임의의 값을 추측해 쿼리 생성
- 세 가지 방법으로 대응: DISTINCT 조회, 용어 사전 제공, ILIKE 부분 일치 검색

## 상세

### 문제 상황
- DB 실제 값: `Bikes`
- 사용자 입력: `"자전거"`
- LLM 생성 쿼리: `WHERE category = 'Bicycle'` → 결과 없음

### 해결 방법

#### 방법 1. DISTINCT 사전 조회 (현재 적용)
쿼리 실행 전 실제 카테고리 목록을 먼저 조회하도록 프롬프트 지침 추가
```
SQL 실행 전 반드시 해당 컬럼의 DISTINCT 값을 먼저 조회하고,
사용자 입력과 가장 유사한 실제 값을 사용해 쿼리를 생성하세요.
```
→ 토큰 및 쿼리 횟수 증가하는 단점 있음

#### 방법 2. 용어 사전 직접 삽입
프롬프트에 한국어-DB 영문명 매핑 테이블 제공
```
[카테고리 매핑]
자전거 = Bikes
부품 = Components
의류 = Clothing
```
→ 값이 고정적이고 많지 않을 때 효과적

#### 방법 3. ILIKE 부분 일치 강제
```sql
WHERE category ILIKE '%%자전거%%'
```
→ 언어 차이나 오타에 대응 가능, 단 노이즈 결과 포함 가능성

### 추천 조합
고정 카테고리 + DISTINCT 조회 + ILIKE 세 가지를 프롬프트에 함께 적용

## 관련 노트
- [[SQL - % 와일드카드 이스케이프]]
- [[Dify Agent - null값 KeyError 처리]]
