# Architecture Decision Records

## 철학
데이터 무결성(오버셀 0, 1인 1건) 최우선. 그 위에서 지연·처리량 최적화. 검증 가능한(자동 테스트로 증명되는) 동시성 보장을 선택한다.

---

### ADR-001: 재고 판정을 DB 락이 아닌 Redis 원자 연산으로
**결정**: 비관적 락/`SELECT FOR UPDATE` 대신 Redis 단일 연산으로 재고 차감.
**이유**: 1만 동시요청을 DB 락으로 직렬화하면 커넥션 고갈·락 경합으로 병목. Redis 단일 스레드 원자성이 오버셀을 구조적으로 차단.
**트레이드오프**: Redis가 진실원본이 되어 DB와의 정합성 동기화 책임이 생김(→ ADR-005 reconciliation).

### ADR-002: 중복+재고+당첨기록+큐적재를 단일 Lua 스크립트로 원자화
**결정**: `apply.lua` 하나에서 `SISMEMBER→GET→DECR→SADD→XADD` 수행.
**이유**: 여러 명령으로 쪼개면 "재고는 깠는데 큐 적재 실패" 같은 부분 실패·보상 트랜잭션이 필요. 원자화로 all-or-nothing 보장.
**트레이드오프**: 로직이 Lua에 들어가 테스트/디버깅 난이도↑. 동시성 테스트는 실제 redis(embedded-redis/Testcontainers)에서 검증.

### ADR-003: 비동기 영속화를 Redis Stream + Consumer Group으로
**결정**: 당첨 확정을 Stream에 적재, 컨슈머가 배치 INSERT.
**이유**: 쓰기 스파이크 평탄화 + Consumer Group의 pending/`XAUTOCLAIM`으로 at-least-once 재처리. List/Pub-Sub은 소비 추적·재처리가 약함.
**트레이드오프**: 신청→DB 반영에 짧은 지연(eventual). 상태 조회에 PENDING 상태 도입으로 해결.

### ADR-004: 멱등성 3중 방어 (Redis Set + Lua 원자성 + DB UNIQUE)
**결정**: `applied` set, Lua 원자성, `enrollment` UNIQUE(member,product) 3겹.
**이유**: 더블클릭·재시도·컨슈머 중복처리 어디서 새도 최종 1건 보장.
**트레이드오프**: 약간의 중복 검사 비용.

### ADR-005: 재고 워밍은 SETNX + DB 기반 재구축
**결정**: 기동 워밍은 덮어쓰지 않는 `SETNX`. Redis 유실 시 `total - count(enrollment)`로 재구축.
**이유**: 재시작 때 `SET`으로 재고를 가득 채우면 오버셀. 멱등 워밍 필요.
**트레이드오프**: 재구축 로직·기동 훅 추가 복잡도.

### ADR-006: Redis 장애 시 fail-fast(503), DB 폴백 금지
**결정**: 신청 경로에서 Redis 불가 시 즉시 503.
**이유**: DB 폴백은 막으려던 stampede를 그대로 유발. 무결성·전체 가용성 우선.
**트레이드오프**: Redis가 단일 의존 지점(SPOF). 운영선 HA(Sentinel/Cluster)로 보완 가능함을 문서화.

### ADR-007: 표준 에러 응답 + ErrorCode enum + @RestControllerAdvice
**결정**: 전 예외를 `{timestamp,status,code,message,path,traceId}` 로 통일.
**이유**: 클라이언트 일관 처리, 디버깅 상관관계(traceId), 민감정보 비노출.
**트레이드오프**: 보일러플레이트 약간.

### ADR-008: 테스트 격리 — embedded-redis(주력) + H2, Testcontainers는 보조
**결정**: 단위·통합·**동시성·Stream 테스트 모두 embedded-redis** 로 실행. embedded-redis는 *실제 redis-server 바이너리*를 띄우므로 **Lua(EVAL)·Stream(XADD/XREADGROUP)이 Docker 없이 동작**한다. Testcontainers(MySQL/Redis)는 Docker가 있을 때 풀 통합 검증용 보조.
**이유**: 하네스 자식 세션은 Docker 보장이 안 된다. 핵심 증빙인 동시성 테스트가 환경에 상관없이 `./gradlew test`로 **반드시** 통과해야 한다.
**트레이드오프**: embedded-redis 바이너리의 OS/아키텍처 호환(특히 Windows) 확인 필요. 실패 시 Testcontainers 폴백.

### ADR-009: 테스트 스키마는 JPA ddl-auto, Flyway는 local/prod 전용
**결정**: `test` 프로파일은 `spring.jpa.hibernate.ddl-auto=create-drop`(H2). `local`/`prod`(MySQL)만 Flyway 마이그레이션 적용. Flyway는 `test`에서 비활성.
**이유**: MySQL 문법 DDL(Flyway `V1__init.sql`)은 H2에서 깨진다. 엔티티는 `@Enumerated(STRING)`·portable 타입으로 H2/MySQL 양립.
**트레이드오프**: 테스트가 실제 MySQL DDL을 검증하진 않음 → Testcontainers MySQL 테스트(보조)로 Flyway 마이그레이션 별도 검증.

### ADR-010: (선택) 순진한 DB락 baseline을 함께 구현해 병목 대조
**결정**: `POST /api/products/{id}/apply-naive`(비관적 락/`SELECT FOR UPDATE` 버전, `remaining_quantity` 컬럼 사용)를 함께 제공하고, k6로 Redis 버전과 동일 부하를 걸어 TPS/p95/에러율을 대조.
**이유**: 포트폴리오의 "병목 발견·개선" 서사는 *측정된 before(naive) → after(Redis)* 대조가 가장 설득력 있다.
**트레이드오프**: 구현·테스트 분량 증가. 시간 부족 시 생략 가능(문서에 '미구현/향후과제'로 명시).
