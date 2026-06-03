# 프로젝트: FCFS Finance — 선착순 금융 상품 가입 시스템

은행 고금리 특판처럼 **수량이 한정된 금융 상품**에 수만 명이 동시에 몰려도, 정확히 한정 수량만·공정한 선착순으로·1인 1건 가입시키는 백엔드 시스템. RDB를 보호하면서 응답 지연을 최소화한다.

## 기술 스택
- Spring Boot 3.3.x (Spring Web, Spring Data JPA, Spring Data Redis/Lettuce, Spring Security, Validation, Actuator)
- Java 17 (LTS), Gradle (Groovy DSL)
- MySQL 8 (RDB) + Redis 7 (재고/대기열/캐시)
- 인증: JWT(jjwt) + BCrypt
- 문서화: springdoc-openapi (Swagger UI)
- 부하 테스트: k6
- 테스트: JUnit 5, Spring Boot Test, MockMvc, embedded-redis(주력), Testcontainers(보조), H2(test 프로파일)

## 아키텍처 규칙
- CRITICAL: 선착순 재고 판정은 **반드시 Redis 원자 연산(단일 Lua 스크립트)** 으로만 한다. DB 비관적 락/`SELECT FOR UPDATE`로 재고를 차감하지 마라. 이유: 1만 동시요청을 DB로 직렬화하면 커넥션 고갈·락 경합으로 병목이 발생하고 오버셀 위험이 커진다. (예외: ADR-010의 `apply-naive` baseline은 *의도적 대조군*으로만 DB 락을 쓴다.)
- CRITICAL: 신청 **실패/중복/마감 요청은 DB에 접근하지 마라.** 당첨이 확정된 건만 Redis Stream → 비동기 컨슈머 경로로 INSERT한다. 이유: 실패 요청까지 DB를 치면 보호 효과가 사라진다.
- CRITICAL: 멱등성은 3중으로 방어한다 — Redis `applied` Set + Lua 원자성 + `enrollment` 테이블 `UNIQUE(member_id, product_id)`. 어느 한 곳에서 새도 최종 1건을 보장해야 한다.
- CRITICAL: 모든 예외는 `ErrorCode`(enum) + `@RestControllerAdvice`(GlobalExceptionHandler)로 표준 응답 `{timestamp, status, code, message, path, traceId}`으로 변환한다. 새 에러 상황이 생기면 ErrorCode에 추가하라. 컨트롤러/서비스에서 직접 응답 포맷을 만들지 마라.
- CRITICAL: Redis의 `stock:*` / `applied:*` / `stream:enroll` 키는 무결성 핵심이므로 evict/임의 만료시키지 마라. 재고 워밍은 `SETNX`로(재시작 시 덮어쓰기 금지).
- 금융 정밀도: 금리/금액은 `BigDecimal`/`DECIMAL`만 사용한다. `double`/`float` 금지. 모든 시각은 UTC `Instant`로 저장·비교한다(서버 타임존 의존 금지).
- 레이어 분리: `common/`(횡단), `member/`, `product/`, `fcfs/`. 한 패키지가 다른 도메인 내부를 직접 참조하지 마라.

## 개발 프로세스
- CRITICAL: 새 기능 구현 시 반드시 테스트를 먼저 작성하고, 테스트가 통과하는 구현을 작성할 것 (TDD)
- 동시성/무결성은 자동 테스트로 증명한다: 동시 N요청·재고 M → 성공==M, 오버셀==0, 중복==0.
- 커밋 메시지는 conventional commits 형식을 따를 것 (feat:, fix:, docs:, refactor:, test:, chore:)
- 테스트는 Docker 없이도 통과해야 한다(하네스 자식 세션 대비): test 프로파일 = H2(ddl-auto) + embedded-redis. Flyway는 local/prod에서만.

## 명령어
```
./gradlew build       # 컴파일 + 테스트
./gradlew test        # 테스트
./gradlew bootRun     # 개발 서버 (local 프로파일, docker-compose up 선행)
docker-compose up -d  # MySQL + Redis 기동
k6 run loadtest/fcfs.js   # 부하 테스트
```
