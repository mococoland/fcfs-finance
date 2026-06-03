# Step 2: domain-persistence

JPA 엔티티·리포지토리·DB 스키마와 상품 시드를 만든다. 무결성 제약(특히 enrollment UNIQUE)이 핵심.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — "데이터 모델(ERD/제약)", 금융 정밀도(BigDecimal)·시간(Instant) 규칙, "시드/부하테스트 데이터"
- `/docs/ADR.md` — ADR-009(test=ddl-auto, Flyway는 local/prod)
- step0: `application-{local,test}.yml`, 패키지 구조
- step1: `common/exception/*` (BusinessException, ErrorCode)

## 작업

1. **엔티티** (`member/domain`, `product/domain`, `fcfs/domain`):
   - `Member`: `id`, `email`(unique, not null), `passwordHash`, `createdAt(Instant)`.
   - `Product`: `id`, `name`, `interestRate`(**`BigDecimal`**, `DECIMAL(5,2)`), `totalQuantity`(int), `remainingQuantity`(int, naive baseline용), `openAt`/`closeAt`(**`Instant`**), `createdAt`. 신청 상태(예정/진행/마감)는 저장 컬럼이 아니라 openAt/closeAt/now로 **파생 계산**하는 헬퍼 메서드 제공.
   - `Enrollment`: `id`, `memberId`, `productId`, `status`(`@Enumerated(STRING)`, 값 `CONFIRMED`), `appliedAt`, `createdAt`. **`@Table(uniqueConstraints = @UniqueConstraint(columnNames={"member_id","product_id"}))`**.
   - 시각은 전부 `Instant`. 금액/금리는 `BigDecimal`.
2. **리포지토리** (Spring Data JPA): `MemberRepository`(findByEmail, existsByEmail), `ProductRepository`, `EnrollmentRepository`(existsByMemberIdAndProductId, countByProductId, findByMemberIdAndProductId).
3. **Flyway 마이그레이션** `src/main/resources/db/migration/V1__init.sql` (MySQL 문법): 위 3테이블 + enrollment 유니크 제약 + FK. **local/prod 전용**(test는 ddl-auto).
4. **상품 시드** `ProductSeeder`(`CommandLineRunner`, `@Profile("local")`): 지금 열린(`openAt ≤ now < closeAt`) 특판 상품 3개를 멱등 생성(예: 수량 100/1,000/10,000, 금리 BigDecimal). 존재 시 skip. `remainingQuantity = totalQuantity`로 초기화.

## Acceptance Criteria

```bash
./gradlew test
```
- 리포지토리 통합 테스트(H2, ddl-auto): 회원/상품 저장·조회, **enrollment 동일 (member,product) 2회 저장 시 `DataIntegrityViolationException`** 단언, `BigDecimal` 금리 정밀도(반올림 없이) 유지, `Instant` 왕복.

## 검증 절차

1. AC 실행.
2. 체크리스트: 금리가 BigDecimal인가(double 아님)? 시각이 Instant인가? enrollment 유니크 제약이 동작하는가? Flyway는 test에서 비활성인가?
3. 결과를 `phases/fcfs-core/step2.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "엔티티 Member/Product(remainingQuantity,BigDecimal,Instant)/Enrollment(UNIQUE member_id+product_id), JPA 리포지토리, Flyway V1__init.sql(local/prod), local 상품 시드(열린 특판 3개 멱등). EnrollmentRepository.existsByMemberIdAndProductId/countByProductId 제공." }`

## 금지사항

- 금리/금액에 `double`/`float`를 쓰지 마라. 이유: 금융 정밀도(반올림 오류).
- 시각에 `LocalDateTime`+서버TZ를 쓰지 마라. 이유: 타임존 버그. `Instant` 사용.
- Flyway를 test 프로파일에서 켜지 마라. 이유: MySQL DDL이 H2에서 깨진다(ADR-009).
- 신청/캐시/Redis 로직을 만들지 마라. 이 step은 영속 계층 전용.
- `index.json`을 수정하지 마라.
