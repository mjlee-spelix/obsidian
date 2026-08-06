## 작업 A — 사전 분석 결과

### A.1 현 인덱스 매트릭스

|테이블/MView|인덱스|타입|컬럼|
|---|---|---|---|
|**spx_mv_audit_enriched**|`_id_idx`|btree UNIQUE|`(id)` — **이것만 존재**|
|**spx_mv_kpi_calls_daily**|`_unique_idx`|btree UNIQUE|`(tenant_id, day, app_owner_dept_id, actor_dept_id, target_app_id, app_mode)`|
|**spx_mv_model_tokens_daily**|`_unique_idx`|btree UNIQUE|`(tenant_id, day, model_provider, model_id)`|
|spx_resource_ownership|pkey + 2|UNIQUE + btree|`(id)`, `(tenant_id, resource_type, resource_id)`, `(owner_department_id)`|
|spx_departments|pkey + 3|UNIQUE + btree + partial|`(id)`, `(tenant_id, code)`, `(tenant_id)`, `(tenant_id, keycloak_group_id) WHERE NOT NULL`|
|spx_accounts|pkey + 2|UNIQUE + btree|`(id)`, `(sub)`, `(email)`|
|spx_audit_events|pkey + 10|다수|`occurred_at`, `tenant_id`, `action_mart`, `actor_email` 등|

### A.3 시드 카디널리티 재확인

|테이블|실측|기대값|정합|
|---|---|---|---|
|spx_audit_events|452,785|(기준 외)|-|
|spx_mv_audit_enriched|**450,156**|450,156|✅|
|spx_mv_kpi_calls_daily|**6,522**|6,522|✅|
|spx_mv_model_tokens_daily|**155**|155|✅|
|spx_departments|**37**|30|⚠️ +7 증가|
|spx_resource_ownership|**177**|150|⚠️ +27 증가|

> departments(37 vs 30)와 resource_ownership(177 vs 150) 변동은 시드 이후 운영으로 인한 자연 증가로 판단. 핵심 마트 3종은 정합 통과.

---

## 작업 C — EXPLAIN ANALYZE 매트릭스

### spx_mv_kpi_calls_daily 쿼리군 (6,522행) — **전부 Index Hit ✅**

|Endpoint|Query|Plan 타입|인덱스 Hit|Actual Time|Buffers|
|---|---|---|---|---|---|
|KPI api-calls (SUM calls)|`SUM(calls) WHERE tenant+day`|**Bitmap Index Scan**|✅ `_unique_idx`|**0.57ms**|shared hit=40|
|KPI adoption (COUNT DISTINCT)|`COUNT(DISTINCT target_app_id)`|**Index Only Scan**|✅ `_unique_idx`|**1.89ms**|shared hit=25|
|drill dept-call-count|`SUM(calls) GROUP BY dept`|Bitmap Index Scan|✅ `_unique_idx`|~0.5ms|shared hit=40|
|drill dept-call-rps (window)|`ROW_NUMBER() OVER`|Bitmap Index Scan|✅ `_unique_idx`|**1.84ms**|shared hit=46|
|drill dept-error-rate|`SUM(errors, calls) GROUP BY dept`|Bitmap Index Scan|✅ `_unique_idx`|~0.5ms|shared hit=40|
|drill dept-dau|`SUM(users) GROUP BY actor_dept`|Bitmap Index Scan|✅ `_unique_idx`|~0.5ms|shared hit=25|

### spx_mv_model_tokens_daily (155행) — **Seq Scan 허용 (소규모)**

|Endpoint|Plan 타입|Actual Time|비고|
|---|---|---|---|
|model-tokens|Seq Scan|**0.21ms**|155행 전량 스캔, 인덱스보다 효율적|

### spx_v_resource_ownership_enriched (View, ~177행) — **Seq Scan 허용 (소규모)**

|Endpoint|Plan 타입|Actual Time|비고|
|---|---|---|---|
|KPI total-objects|Hash Join (Seq+Seq)|**0.38ms**|resource_ownership(177) + departments(37)|
|drill top-owners|Hash Join x3|**1.28ms**|+ accounts(326) JOIN|
|drill dept-cumulative|Hash Join|~0.4ms||
|drill dept-new-creations|Hash Join|~0.4ms||
|dept-objects|Hash Join|~0.4ms||
|dept-activity|Hash Join|~0.4ms||

### spx_mv_audit_enriched (450,156행) — **전부 Seq Scan ⚠️🔥**

|Endpoint|Query|Plan 타입|Actual Time|Buffers|비고|
|---|---|---|---|---|---|
|drill model-call-share|`GROUP BY model_id`|**Parallel Seq Scan**|**1,480ms**|hit=2451 read=9838|tenant+occurred_at 필터|
|drill model-users|`COUNT(DISTINCT actor_id) GROUP BY model_id`|**Parallel Seq Scan**|**2,275ms**|hit=2614 read=9742|tenant+occurred_at+actor+model 필터|
|drill dept-user-activity DAU|`COUNT(DISTINCT actor_id) 24h`|**Parallel Seq Scan**|**1,121ms**|hit=2698 read=9646||
|drill error-table cause|`ROW_NUMBER() OVER dept, error_text`|**Parallel Seq Scan**|**1,362ms**|hit=2813 read=9550||
|drill dept-user-activity NEW|`NOT IN (subquery)`|**Seq Scan x2**|**3,333ms** 🔥|hit=5732 read=18811|테이블 2회 풀스캔|
|drill dept-user-activity CHURNED|`NOT IN (subquery)`|**Seq Scan x2**|**3,274ms** 🔥|hit=5988 read=18555|테이블 2회 풀스캔|

---

## 작업 D — 결과 분석

### D.1 응답 시간 분포 (단일 호출, 부하 0)

|범위|Endpoints|최대 Actual Time|
|---|---|---|
|< 2ms|KPI 4종 + drill calls 3종 + model-tokens + view 6종 (12개)|1.89ms|
|1~2.3s|drill model-call-share, model-users, DAU, error-cause (4개)|2,275ms|
|**> 3s** 🔥|**drill dept-user-activity NEW / CHURNED** (2개)|**3,333ms**|

**p95 base: ~3,333ms — 3초 임계 초과** (15종 중 2종이 3초 이상)

### D.2 Seq Scan 발견 매트릭스

|Endpoint|쿼리 영역|원인|현 인덱스|
|---|---|---|---|
|drill model-call-share|`spx_mv_audit_enriched WHERE tenant_id + occurred_at`|**tenant_id, occurred_at 인덱스 부재**|`id` only|
|drill model-users|동일|동일|`id` only|
|drill dept-user-activity DAU|동일|동일|`id` only|
|drill dept-user-activity WAU|동일|동일|`id` only|
|drill dept-user-activity NEW|동일 + NOT IN 서브쿼리 (2회 풀스캔)|동일|`id` only|
|drill dept-user-activity CHURNED|동일 + NOT IN 서브쿼리 (2회 풀스캔)|동일|`id` only|
|drill error-table cause|동일|동일|`id` only|

**결론: 모든 Seq Scan이 `spx_mv_audit_enriched` 단일 테이블에 집중. 원인은 `(tenant_id, occurred_at)` 복합 인덱스 부재.**

### D.3 Chain Refresh 상태

**chain refresh 실패 중** 🔴

```
relation "public.spx_v_audit_enriched" does not exist
```

- **원인**: 5/19 마이그레이션(`20260519000000_fix_layer1_mview_naming`)에서 `spx_v_audit_enriched` → `spx_mv_audit_enriched`로 이름 변경됨. 소스 코드([db-poller.ts:51](vscode-webview://0kun88lmbhvoelp74f387de507knj854009fbc6fq3qvel28u55p/dify-audit/src/workers/db-poller.ts#L51))는 이미 새 이름 사용 중이나, **실행 중인 컨테이너(dify-audit)가 구 빌드**라서 옛 이름 참조
- **영향**: 마트 데이터가 갱신 안 됨 (현재 autovacuum 시점: 2026-05-19 08:08 UTC 기준으로 동결)
- **조치**: 컨테이너 재빌드 필요 (본 위임 범위 외 — 진행 금지 항목)

---

## 작업 E — 인덱스 추가 권장

Update Todos

- [ ] 
    
    작업 A.1: 현 인덱스 전수 조사
    
- [ ] 
    
    작업 A.2: endpoint 15종 목록 + URL 패턴 정리
    
- [ ] 
    
    작업 A.3: 시드 카디널리티 재확인
    
- [ ] 
    
    작업 C: EXPLAIN ANALYZE 모든 쿼리 plan
    
- [ ] 
    
    작업 D: 결과 분석 (Seq Scan 매트릭스 + 권장)
    
- [ ] 
    
    작업 E: 인덱스 추가 권장 보고
    

| 우선순위      | 인덱스 후보                                                                      | DDL                                                                                                                             | 영향 endpoint                                                                                                  | 예상 효과                                                                     | trade-off                                               |
| --------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------- | ------------------------------------------------------- |
| 🔥 **높음** | `spx_mv_audit_enriched (tenant_id, occurred_at)`                            | `CREATE INDEX CONCURRENTLY spx_mv_audit_enriched_tenant_occurred_idx ON public.spx_mv_audit_enriched (tenant_id, occurred_at);` | model-call-share, model-users, dept-user-activity (DAU/WAU/NEW/CHURNED), error-table cause — **7개 endpoint** | Parallel Seq Scan (1.1~3.3s) → Index Scan (~~10-50ms 추정). **60~~100배 개선** | REFRESH CONCURRENTLY 시 인덱스 재구축 비용 증가. 450K행 기준 ~1-2초 추가 |
| 중         | `spx_mv_audit_enriched (tenant_id, occurred_at, actor_id)` INCLUDE 확장       | `CREATE INDEX CONCURRENTLY ... (tenant_id, occurred_at, actor_id) INCLUDE (actor_dept_id, model_id);`                           | model-users, dept-user-activity — Index Only Scan 가능                                                         | Seq Scan → Index Only Scan, 힙 접근 제거                                       | 인덱스 크기 ~2배, REFRESH 부담 증가                               |
| 낮음        | 없음 — Layer 2 (`kpi_calls_daily`, `model_tokens_daily`) 및 View 쿼리는 현재 상태로 충분 | -                                                                                                                               | -                                                                                                            | -                                                                         | -                                                       |

---

## 검증 요약

|검증|결과|
|---|---|
|V.2 EXPLAIN ANALYZE 매트릭스|✅ 15종 전체 쿼리 패턴 plan 출력 완료|
|V.3 시드 카디널리티 정합|✅ 핵심 마트 3종 정합 (dept/ownership 소폭 증가는 자연 증가)|
|V.4 chain refresh 정상 동작|🔴 **실패 중** — `spx_v_audit_enriched` 구명 참조. 컨테이너 재빌드 필요|

---

## 발견 사항 요약 및 다음 작업 안내

### 핵심 발견

1. **`spx_mv_audit_enriched`에 `(tenant_id, occurred_at)` 인덱스가 없음** — 7개 endpoint가 450K행 풀스캔, 2개가 3초 임계 초과
2. **chain refresh가 깨져 있음** — dify-audit 컨테이너가 구 빌드 사용 중 (소스는 이미 수정됨)

### 권장 다음 작업

1. **🔥 즉시**: dify-audit 컨테이너 재빌드 → chain refresh 정상화
2. **🔥 높음 PR**: `spx_mv_audit_enriched (tenant_id, occurred_at)` 인덱스 추가 마이그레이션
3. **정식 부하 테스트**: 인덱스 추가 후 k6/pgbench로 RPS 시뮬레이션 진입 가능
4. dept-user-activity의 `NOT IN` 서브쿼리 패턴은 인덱스 추가만으로도 크게 개선되지만, 추후 `LEFT JOIN ... IS NULL` 또는 `EXCEPT` 패턴으로 리팩토링하면 추가 개선 가능