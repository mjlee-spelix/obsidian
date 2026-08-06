# Quality Score — 서비스별 구현 품질 등급

## 등급 기준

| 등급 | 의미 | 조건 |
|------|------|------|
| A | 완성 | 전 레이어 구현 + 테스트 커버리지 80%+ + OpenAPI 문서화 |
| B | 기능 완료 | 전 레이어 구현 + 기본 테스트 존재 |
| C | 부분 구현 | 일부 레이어만 존재, 테스트 미흡 |
| D | 미착수 | Prisma 스키마만 있거나 미구현 |

## 현재 상태

> D-14 사전구축 단계. 모든 서비스 미착수 (2026-04-16 기준)

| 서비스 | Repository | Service | Controller | Zod Schema | Test | 등급 |
|--------|-----------|---------|------------|-----------|------|------|
| MaterialService | - | - | - | - | - | **D** |
| CuttingService | - | - | - | - | - | **D** |
| ProductionService | - | - | - | - | - | **D** |
| QualityService | - | - | - | - | - | **D** |
| FinishingService | - | - | - | - | - | **D** |
| ShipmentService | - | - | - | - | - | **D** |
| OrderRepository | - | - | - | - | - | **D** |
| OptimizationService | - | - | - | - | - | **D** |
| ERPSyncService | - | - | - | - | - | **D** |
| PermissionService | - | - | - | - | - | **D** |
| BackupService | - | - | - | - | - | **D** |
| UserLayoutService | - | - | - | - | - | **D** |

## 서비스 구현 체크리스트

각 서비스 완성 조건:

- [ ] `packages/db/prisma/schema.prisma` — Prisma 모델 정의
- [ ] `bunx prisma migrate dev` — 마이그레이션 생성 및 적용
- [ ] `apps/api/src/repositories/{service}.repository.ts` — IRepository 구현
- [ ] `apps/api/src/services/{service}.service.ts` — 비즈니스 로직, `Result<T, DomainError>` 반환
- [ ] `packages/openapi/src/schemas/{service}.ts` — Zod 입출력 스키마
- [ ] `apps/api/src/controllers/{service}.controller.ts` — Express 라우터 + Zod 검증
- [ ] `apps/api/src/app.ts` — 라우터 등록
- [ ] `apps/api/src/tests/{service}.test.ts` — Testcontainers 통합 테스트
- [ ] `packages/openapi/src/openapi.yaml` — OpenAPI 스펙 업데이트
- [ ] domain-validate.sh 패턴 검사 통과 (CRITICAL=0)

## Layer C 테이블 보호 현황

| 테이블 | Layer | 삭제 방지 트리거 | 상태 |
|--------|-------|--------------|------|
| garment_lots | C | PERMANENT | - (마이그레이션 미생성) |
| line_outputs | C | PERMANENT | - |
| line_daily_summaries | C | PERMANENT | - |
| qc_inspections | C | PERMANENT | - |
| mfz_records | C | PERMANENT | - |

## Gate 체크리스트

### Gate 1 (W4): 도메인 골격
- [ ] 9개 서비스 Repository 인터페이스 정의
- [ ] Layer C 5개 테이블 트리거 마이그레이션
- [ ] domain-validate.sh CI 연동

### Gate 2 (W12): 핵심 기능
- [ ] MaterialService ~ ShipmentService 등급 B 이상
- [ ] k6 p95 < 500ms
- [ ] Dredd 계약 테스트 100% 통과

### Gate 3 (W17): 통합
- [ ] ERP Mock 연동 완료
- [ ] 33개 화면 + Dashboard + MyPage 기능 완성
- [ ] fast-check 6개 속성 테스트 통과
- [ ] 권한 관리(Admin-P) 화면 및 PermissionService 완성
- [ ] DB 백업 Job 3계층 동작 확인

### Gate 4 (W20): 출시 준비
- [ ] 전 서비스 등급 B 이상
- [ ] 보안 감사 통과 (bun audit 0 critical)
- [ ] PWA 2시간 오프라인 동작 검증
