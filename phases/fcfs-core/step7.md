# Step 7: queue-consumer

당첨 확정(stream:enroll)을 DB로 옮기는 비동기 컨슈머와 신청 상태 조회 API를 만든다. 멱등·장애복구가 핵심.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — 신청 시퀀스의 컨슈머 부분, 장애모드(컨슈머 크래시/DB다운/유니크충돌/배치부분실패), Redis 키(stream:enroll, dlq:enroll), 상태관리(4상태), 관측성(traceId MDC 복원)
- `/docs/ADR.md` — ADR-003(Stream+Consumer Group), ADR-004(멱등 3중)
- step2: `Enrollment`, `EnrollmentRepository`(existsByMemberIdAndProductId, findByMemberIdAndProductId)
- step5: `applied:{id}`
- step6: `stream:enroll` 메시지 필드(memberId/productId/traceId), ApplyResult

## 작업

1. **Consumer Group 초기화**: 기동 시 `stream:enroll`에 컨슈머 그룹(예 `enroll-consumers`) 생성(이미 있으면 무시, `MKSTREAM`).
2. **`fcfs/consumer/EnrollmentStreamConsumer`**:
   - `poll()`: `XREADGROUP`(신규) + `XAUTOCLAIM`(idle pending 회수)으로 메시지 배치 수신.
   - 각 메시지: traceId를 **MDC에 복원**(상관관계) 후 처리.
   - **레코드 단위 멱등 INSERT**: `existsByMemberIdAndProductId`로 선확인 또는 INSERT 후 `DataIntegrityViolationException` catch → 이미 있으면 무시. **배치 중 한 건 충돌이 나머지를 롤백시키지 않도록 레코드 단위 트랜잭션/처리.**
   - 성공·중복(이미 영속) → `XACK`. 주기적으로 acked min-id 기준 트림.
   - 처리 실패가 N회(재시도 한도) 누적된 메시지 → `dlq:enroll`로 `XADD` 후 원본 `XACK`(+경고 로그/메트릭).
   - DB 다운 시: `XACK` 안 함 → 다음 폴에서 pending으로 재처리(백오프). 신청 접수(step6)는 계속 가능.
   - 스케줄 구동: `@Scheduled`(짧은 fixedDelay)로 `poll()` 호출. **단, 테스트용으로 `poll()`을 동기 호출 가능한 public 메서드로 노출**(스케줄러 타이밍 비의존).
   - graceful shutdown: 진행 중 배치 drain/ack 시도.
3. **상태 조회** `GET /api/products/{id}/apply/status`(authenticated) — `fcfs/controller`:
   - (applied set ∈?) × (enrollment ∈?)로 4상태 판정:
     - applied ∈ & enrollment ∈ → `CONFIRMED`
     - applied ∈ & enrollment ∉ → `PENDING`
     - applied ∉ & 마감(stock<=0) → `FAILED_SOLD_OUT`
     - applied ∉ & 그 외 → `NOT_APPLIED`
   - 응답 `{ status, productId }`.

## Acceptance Criteria

```bash
./gradlew test
```
- **e2e(embedded-redis)**: 신청(step6 경로)→`consumer.poll()`(동기 호출)→DB에 enrollment 1건→`GET status`가 `CONFIRMED`. poll 전 조회는 `PENDING`.
- **멱등/크래시 복구**: 같은 메시지를 두 번 처리(또는 ack 전 재처리)해도 enrollment 1건 유지(유니크 충돌 무시ACK). pending 메시지를 `XAUTOCLAIM`으로 회수해 처리.
- **배치 부분 충돌**: 배치에 이미 영속된 건 + 신규 건 혼재 시, 신규는 영속되고 충돌 건만 무시(나머지 롤백 안 됨).

## 검증 절차

1. AC 실행.
2. 체크리스트: 멱등 INSERT가 동작하는가? 배치 부분 충돌이 나머지를 죽이지 않는가? traceId가 컨슈머 로그에 복원되는가? 4상태가 정확한가? `poll()`이 테스트에서 동기 호출되는가?
3. 결과를 `phases/fcfs-core/step7.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "EnrollmentStreamConsumer(Consumer Group, XREADGROUP+XAUTOCLAIM, 레코드단위 멱등INSERT, XACK+트림, dlq:enroll, traceId MDC복원, 테스트용 poll() 동기노출, graceful drain) + GET /apply/status(4상태). 신청→poll→CONFIRMED e2e·크래시 재처리·배치 부분충돌 테스트 통과. fcfs-core 완료(백엔드 핵심·무결성 증빙 완성)." }`

## 금지사항

- 배치 전체를 한 트랜잭션으로 묶어 한 건 충돌에 전부 롤백하지 마라. 이유: 정상 당첨 유실.
- 같은 당첨을 여러 번 INSERT해 중복 행을 만들지 마라. 유니크 충돌은 무시ACK.
- 부하테스트·resilience·UI는 만들지 마라(fcfs-ops phase 소관).
- `index.json`을 수정하지 마라.
