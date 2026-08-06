---
globs: apps/api/src/**/*.test.ts,apps/web/src/**/*.test.ts,packages/**/*.test.ts
description: 테스트 파일 작성/수정 시 자동 참조되는 규칙
---

# 테스트 규칙

> 적용 대상: `**/*.test.ts`
> 참고: CLAUDE.md §7 Testing Rules — 4개 도구 의무

## 요약
4개 Harness 도구(TC + fast-check + Dredd + k6)를 모두 사용한다.
Gate 1~4 통과 조건에 각 도구 결과가 포함되므로 생략 금지.

## 규칙

### 1. Testcontainers (TC) — 통합 테스트 (H-2)
```typescript
// apps/api/src/services/__tests__/production.service.test.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql'
import { PrismaClient } from '@prisma/client'

describe('ProductionService', () => {
  let container: StartedPostgreSqlContainer
  let prisma: PrismaClient

  beforeAll(async () => {
    container = await new PostgreSqlContainer().start()
    prisma = new PrismaClient({ datasourceUrl: container.getConnectionUri() })
    await prisma.$executeRawUnsafe(fs.readFileSync('migrations/setup.sql', 'utf8'))
  })

  afterAll(async () => {
    await prisma.$disconnect()
    await container.stop()
  })

  it('라인 실적 기록 — 정상', async () => {
    const service = new ProductionService(new LotRepository(prisma), prisma)
    const result = await service.recordLineOutput({ ... })
    expect(result.ok).toBe(true)
    expect(result.value?.dataStatus).toBe('PERMANENT')  // Layer C 확인
  })

  it('MFZ_HOLD LOT 출하 차단', async () => {
    const shipmentService = new ShipmentService(new LotRepository(prisma))
    const result = await shipmentService.confirmShipment({ lotId: mfzHoldLotId })
    expect(result.ok).toBe(false)
    expect(result.error?.code).toBe('LOT_MFZ_HOLD')
  })
})
```

### 2. fast-check — 속성 기반 테스트 (6개 필수)
```typescript
// packages/domain/src/__tests__/properties.test.ts
import * as fc from 'fast-check'

// 속성 1: DHU 계산
it('DHU = defectCount/inspectedQty*100', () => {
  fc.assert(fc.property(
    fc.integer({ min: 1, max: 1000 }),  // inspectedQty
    fc.integer({ min: 0, max: 100 }),   // defectCount
    (inspectedQty, defectCount) => {
      const dhu = (defectCount / inspectedQty) * 100
      return dhu >= 0 && dhu <= 100
    }
  ), { numRuns: 10000 })
})

// 속성 2: Bundle Shading 유일성
it('동일 LOT+색상에 단일 Roll만', () => {
  fc.assert(fc.property(
    fc.array(fc.record({ lotId: fc.uuid(), colorCode: fc.string(), receiptId: fc.uuid() })),
    (bundles) => {
      const key = (b: any) => `${b.lotId}-${b.colorCode}`
      const grouped = groupBy(bundles, key)
      return Object.values(grouped).every(g => new Set(g.map(b => b.receiptId)).size === 1)
    }
  ), { numRuns: 10000 })
})

// 속성 3: OEE (0~100)
// 속성 4: CP 날짜 유효성
// 속성 5: Marker 효율 (0~100%)
// 속성 6: 릴렉싱 시간 (RELAXATION_HOURS 맵 준수)
```

### 3. Dredd — OpenAPI 계약 테스트 (W5 이후)
```yaml
# dredd.yml
blueprint: packages/openapi/spec.yaml
endpoint: http://localhost:3000
reporter: cli
```
- spec.yaml에 정의된 모든 엔드포인트 자동 검증
- failures > 0 시 PR 차단
- 새 엔드포인트 추가 시 spec.yaml 먼저 업데이트

### 4. k6 — 성능 테스트 (W9 이후 PR 게이팅)
```javascript
// tests/k6/lot-trace.js
import http from 'k6/http'
import { check } from 'k6'

export const options = {
  vus: 5,
  duration: '30s',
  thresholds: { 'http_req_duration{name:lot-trace}': ['p(95)<800'] }  // ≤800ms
}

export default function () {
  const res = http.get(`${BASE_URL}/api/lots/TEST-L001/trace`)
  check(res, { 'status 200': r => r.status === 200 })
}
```

### 5. Layer C 삭제 방지 트리거 검증 (Gate 1·3·4 필수)
```typescript
it('PERMANENT 레코드 삭제 시도 → DB 예외', async () => {
  const lot = await prisma.garmentLot.create({ data: { ...data, dataStatus: 'PERMANENT' } })
  await expect(prisma.garmentLot.delete({ where: { id: lot.id } }))
    .rejects.toThrow('PERMANENT 레코드 삭제 불가')
})
```

### 6. 테스트 범위 기준
- Service: Testcontainers + fast-check, 커버리지 ≥80%
- Repository: TC (실제 PostgreSQL)
- Route: HTTP 요청/응답, 상태코드, 에러 형식 검증
- 에러 시나리오: Gate 2에서 27개 케이스 필수
