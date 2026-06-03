# Step 5: stock-warming

Redis 재고 카운터를 안전하게 초기화/재구축한다. 재시작 시 재고가 리셋돼 오버셀이 나는 것을 막는 게 핵심.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — Redis 키 스키마(`stock:{id}` TTL없음·evict금지, `applied:{id}`), "캐싱 전략"의 워밍/재구축, 장애모드(Redis FLUSH 재구축)
- `/docs/ADR.md` — ADR-005(SETNX + DB 재구축)
- `/CLAUDE.md` — stock/applied 키 규칙, 워밍은 SETNX
- step2: `Product`, `Enrollment`, `EnrollmentRepository`(countByProductId, 멤버 조회)

## 작업

1. **`fcfs/redis/StockInitializer`** (기동 훅, `ApplicationRunner` 또는 `@EventListener(ApplicationReadyEvent)`):
   - 진행 중(또는 곧 열릴) 모든 상품에 대해 재고 워밍.
   - `stock:{id}` 를 **`SETNX (totalQuantity - countByProductId)`** 로 설정. **이미 존재하면 덮어쓰지 않는다**(재시작 시 리셋 금지).
   - `applied:{id}` 가 비어 있고 enrollment가 있으면, enrollment의 member들을 `SADD`로 재구성(reconciliation).
2. **재구축 메서드** `rebuild(productId)` (Redis 유실 대비, 수동/조건부 호출 가능):
   - `stock:{id} = totalQuantity - count(enrollment)` (덮어쓰기 허용 버전, 명시적 재구축용).
   - `applied:{id}` 를 enrollment 멤버들로 재구성.
   - stock과 applied가 **동시에 정합**하도록(같은 enrollment 집합 기준).
3. **`stock:*`/`applied:*` 키에 TTL을 걸지 않는다.** (evict 금지 — ARCHITECTURE.md)

> 이 step은 Redis 카운터를 "준비"만 한다. 실제 차감(DECR)·신청 판정은 step6(Lua). 여기서 신청 API를 만들지 마라.

## Acceptance Criteria

```bash
./gradlew test
```
- 통합 테스트(embedded-redis): (a) 워밍 후 `stock:{id} == total - 기존당첨수`, (b) **재워밍(재시작 시뮬레이션) 시 기존 stock 값이 보존**(SETNX, 덮어쓰기 안 됨), (c) `rebuild` 호출 시 stock·applied가 enrollment 기준으로 정확히 재구성, (d) stock/applied 키에 TTL이 없음.

## 검증 절차

1. AC 실행.
2. 체크리스트: SETNX로 재시작 리셋이 방지되는가? rebuild가 stock+applied를 동시에 정합화하는가? 키에 TTL이 없는가?
3. 결과를 `phases/fcfs-core/step5.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "StockInitializer(기동 SETNX stock:{id}=total-count, applied 재구성) + rebuild(productId)(Redis 유실 재구축, stock+applied 동시 정합). stock/applied는 TTL 없음. 재시작 리셋 방지 테스트 통과." }`

## 금지사항

- 워밍에 `SET`(덮어쓰기)을 쓰지 마라. 반드시 `SETNX`. 이유: 재시작 시 재고가 가득 차 오버셀(ADR-005).
- stock/applied 키에 TTL을 걸지 마라. 이유: evict/만료 시 무결성 붕괴.
- 신청(apply) API·Lua를 만들지 마라. step6 소관.
- `index.json`을 수정하지 마라.
