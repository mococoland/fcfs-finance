# Step 6: fcfs-core ⭐ (선착순 핵심)

이 프로젝트의 심장. 중복체크·재고차감·당첨자등록·큐적재를 **단일 Lua 스크립트로 원자 실행**해 오버셀 0·1인1건을 보장한다.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — "선착순 신청 시퀀스", "Lua 원자 스크립트(apply.lua) 설계 불변식", Redis 키 스키마(stream:enroll 단일), API 명세(POST /apply), 상태관리
- `/docs/ADR.md` — ADR-001(Redis 원자), ADR-002(단일 Lua), ADR-004(멱등 3중)
- `/CLAUDE.md` — CRITICAL(Redis 원자연산만, 실패요청 DB접근 금지, 멱등 3중)
- step1: ErrorCode(FCFS_SOLD_OUT/ALREADY_APPLIED/NOT_INITIALIZED, PRODUCT_NOT_OPEN/CLOSED)
- step2: `Product`(기간 파생), `ProductRepository`
- step4: 상품 캐시(기간 검증용 메타)
- step5: `stock:{id}`/`applied:{id}` 워밍

## 작업

1. **`src/main/resources/lua/apply.lua`** — 원자 스크립트.
   - `KEYS[1]=stock:{id}`, `KEYS[2]=applied:{id}`, `KEYS[3]=stream:enroll`
   - `ARGV[1]=memberId`, `ARGV[2]=productId`, `ARGV[3]=traceId`
   - 로직(순서 고정):
     ```
     if SISMEMBER(KEYS[2], ARGV[1]) == 1 then return -1 end      -- 중복
     local s = tonumber(redis.call('GET', KEYS[1]))
     if s == nil then return -2 end                              -- 미초기화
     if s <= 0 then return 0 end                                 -- 마감
     redis.call('DECR', KEYS[1])
     redis.call('SADD', KEYS[2], ARGV[1])
     redis.call('XADD', KEYS[3], '*', 'memberId', ARGV[1], 'productId', ARGV[2], 'traceId', ARGV[3])
     return 1                                                    -- 성공
     ```
   - **음수 재고 진입 불가**(s<=0 선검사). 부분 실패 없음(단일 스크립트).
2. **`fcfs/service/ApplyResult`** (enum): `SUCCESS, SOLD_OUT, DUPLICATE, NOT_INITIALIZED`. Lua 반환(1/0/-1/-2) 매핑.
3. **`fcfs/redis/FcfsRedisRepository`**: `DefaultRedisScript<Long>`(apply.lua 로드, EVALSHA 캐싱)로 `apply(productId, memberId, traceId) → ApplyResult`.
4. **`fcfs/service/FcfsService.apply(memberId, productId)`**:
   - **기간 검증(BR-4)**: 상품 메타(step4 캐시/DB)로 `now < openAt`→`PRODUCT_NOT_OPEN`(409), `now >= closeAt`→`PRODUCT_CLOSED`(409).
   - Lua 호출 → `ApplyResult`:
     - SUCCESS → 정상 반환(접수).
     - SOLD_OUT → `BusinessException(FCFS_SOLD_OUT)`(409).
     - DUPLICATE → `BusinessException(FCFS_ALREADY_APPLIED)`(409).
     - NOT_INITIALIZED → `BusinessException(FCFS_NOT_INITIALIZED)`(503).
   - traceId는 MDC에서 가져와 Lua ARGV로 전달(컨슈머 상관관계용).
5. **`fcfs/controller/ApplyController`**: `POST /api/products/{id}/apply`(authenticated). 인증 주체(memberId) 사용. 성공 시 **202 Accepted**(접수, 비동기 확정).

> 이 step은 Redis까지만 책임진다. Stream에서 DB로 옮기는 컨슈머·상태조회는 step7.

## Acceptance Criteria

```bash
./gradlew test
```
- **동시성 테스트(핵심 증빙, embedded-redis)**: 재고 100 상품에 `ExecutorService(예: 32스레드)` + `CountDownLatch` start gate로 **동시 1,000 요청**(서로 다른 memberId) → `AtomicInteger` 집계로 **성공==100, SOLD_OUT==900, 오버셀==0**. 그리고 `stock:{id}==0`, `SCARD applied:{id}==100`, `XLEN stream:enroll==100` 단언.
- **중복 테스트**: 같은 memberId 2회 → 1회 SUCCESS, 1회 DUPLICATE. stock은 1만 감소.
- **기간 테스트**: openAt 이전→PRODUCT_NOT_OPEN, closeAt 이후→PRODUCT_CLOSED.
- **미초기화 테스트**: stock 키 없는 상품→FCFS_NOT_INITIALIZED(503).

## 검증 절차

1. AC 실행. 동시성 테스트가 실제 redis(embedded-redis)에서 통과해야 한다.
2. 체크리스트: Lua가 단일 스크립트로 원자 실행되는가? 오버셀 0이 증명되는가? 실패/중복 요청이 DB를 치지 않는가(이 step엔 DB 쓰기 자체가 없음)? 음수 재고가 불가능한가?
3. 결과를 `phases/fcfs-core/step6.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "apply.lua(SISMEMBER→GET→DECR→SADD→XADD 원자, 반환 1/0/-1/-2), FcfsRedisRepository(DefaultRedisScript), FcfsService.apply(기간검증+결과매핑), ApplyController POST /apply→202. 동시 1000요청·재고100→성공100·오버셀0·중복0 테스트 통과. 당첨은 stream:enroll에 적재됨(DB반영은 step7)." }`

## 금지사항

- 재고 차감을 DB 락이나 애플리케이션 락(synchronized 등)으로 하지 마라. 반드시 Lua 원자 연산. 이유: 병목·오버셀(ADR-001/002).
- 중복/재고/큐적재를 별도 Redis 호출로 쪼개지 마라. 단일 Lua. 이유: 부분 실패 방지.
- 실패/중복/마감 경로에서 DB에 쓰지 마라.
- `index.json`을 수정하지 마라.
