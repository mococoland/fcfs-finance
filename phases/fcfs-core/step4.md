# Step 4: product-query-cache

상품 목록/단건 조회 API와 Redis cache-aside를 만든다. 고빈도 조회가 DB를 거의 안 치게 한다.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — "캐싱 전략", Redis 키 스키마(`product:meta:{id}`, evict 허용), API 명세(GET /api/products, /{id}), 상품 상태(예정/진행/마감) 파생
- `/CLAUDE.md` — 신청 실패/조회는 캐시 우선, Redis 키 규칙
- step1: ErrorCode(PRODUCT_NOT_FOUND)
- step2: `Product`, `ProductRepository`, 상태 파생 헬퍼

## 작업

1. **`config/RedisConfig`**: `RedisTemplate`/`StringRedisTemplate`, JSON 직렬화(Jackson, `JavaTimeModule`로 Instant 지원). (step0에서 일부 했으면 보강)
2. **`product/cache/ProductCacheRepository`** (cache-aside):
   - `get(productId)`: `product:meta:{id}` 히트 시 역직렬화 반환, 미스 시 null.
   - `put(productDto, ttl)`: 짧은 TTL(예 60s)로 적재.
   - `evict(productId)`: 키 삭제(상품 변경 시).
3. **`product/service/ProductService`**:
   - `getProduct(id)`: 캐시 조회 → 미스 시 DB(`ProductRepository`) → 캐시 적재 → 응답 DTO. 미존재 시 `BusinessException(PRODUCT_NOT_FOUND)`(404).
   - `listProducts()`: 목록(소규모이므로 DB 직접 또는 짧은 캐시).
   - 응답 DTO에 잔여 수량·금리(BigDecimal)·openAt/closeAt(Instant)·**파생 상태**(SCHEDULED/OPEN/CLOSED) 포함. (잔여 수량의 실시간 진실원본은 Redis stock이지만, 이 step에서는 product.remainingQuantity 또는 캐시값을 노출하고, 실시간 stock 연동은 step5/6에서 일관화.)
4. **`product/controller/ProductController`**: `GET /api/products`, `GET /api/products/{id}`. permitAll.
5. **스탬피드 완화**: 짧은 TTL + (선택) 단일 리로드 가드. 과도한 복잡도는 피한다.

## Acceptance Criteria

```bash
./gradlew test
```
- 통합 테스트(embedded-redis): (a) 첫 조회 DB 적중→캐시 적재, (b) **두 번째 조회는 캐시 히트로 리포지토리 미호출**(스파이/`verify`로 호출 횟수 0 단언), (c) 미존재 id→404 PRODUCT_NOT_FOUND, (d) 파생 상태(예정/진행/마감) 계산 정확.

## 검증 절차

1. AC 실행.
2. 체크리스트: 캐시 히트 시 DB 미조회가 증명되는가? 404가 표준 ErrorCode인가? Instant/BigDecimal 직렬화가 깨지지 않는가?
3. 결과를 `phases/fcfs-core/step4.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "상품 조회 API(GET /api/products, /{id}) + cache-aside(product:meta:{id}, TTL60s, evict). ProductService.getProduct(캐시→DB→적재, 404). 응답 DTO에 잔여수량/금리(BigDecimal)/기간(Instant)/파생상태. 캐시 히트 시 DB 미조회 테스트 통과." }`

## 금지사항

- 조회마다 DB를 치게 만들지 마라. 이유: 캐싱 전략의 목적(RDB 부하 차단)을 무너뜨린다.
- `product:meta` 외 키에 손대지 마라(stock/applied/stream은 step5/6 소관).
- 신청(apply) 로직을 만들지 마라.
- `index.json`을 수정하지 마라.
