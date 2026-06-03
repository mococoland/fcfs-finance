# Step 1: common-infra

전 도메인이 공유하는 횡단 관심사 — 표준 에러 체계, 응답 래퍼, traceId 필터 — 를 만든다. 이후 모든 step이 이걸 재사용한다.

## 읽어야 할 파일

- `/CLAUDE.md` — "모든 예외는 ErrorCode + @RestControllerAdvice로 표준 응답" CRITICAL 규칙
- `/docs/ARCHITECTURE.md` — "에러 핸들링 표준" 섹션, ErrorCode↔HTTP 매핑 전수표, ErrorResponse JSON 예시
- step0 산출물: `build.gradle`, `com.portfolio.fcfs.common` 패키지, `application-test.yml`

## 작업

`com.portfolio.fcfs.common` 아래에 구현한다.

1. **`exception/ErrorCode.java`** (enum): ARCHITECTURE.md "ErrorCode↔HTTP 매핑" 표의 **모든 코드**를 정의. 각 상수는 `{ HttpStatus httpStatus, String code, String message }` 필드를 가진다. (VALIDATION_ERROR, AUTH_INVALID_CREDENTIALS, AUTH_TOKEN_MISSING/INVALID/EXPIRED, ACCESS_DENIED, MEMBER_EMAIL_DUPLICATED, PRODUCT_NOT_FOUND, PRODUCT_NOT_OPEN, PRODUCT_CLOSED, FCFS_SOLD_OUT, FCFS_ALREADY_APPLIED, FCFS_NOT_INITIALIZED, REDIS_UNAVAILABLE, RATE_LIMITED, INTERNAL_ERROR)
2. **`exception/BusinessException.java`**: `RuntimeException` 상속, `ErrorCode`를 보유. 선택적 상세 메시지 오버라이드.
3. **`response/ErrorResponse.java`**: `{ Instant timestamp, int status, String code, String message, String path, String traceId }`. 정적 팩토리 `of(ErrorCode, path, traceId)`.
4. **`response/ApiResponse.java`** (선택 래퍼): `{ T data, String traceId }`.
5. **`web/TraceIdFilter.java`** (`OncePerRequestFilter`): 요청마다 UUID traceId를 MDC(`traceId`)에 넣고 응답 헤더(`X-Trace-Id`)에도 실음. finally에서 MDC 정리.
6. **`web/GlobalExceptionHandler.java`** (`@RestControllerAdvice`):
   - `BusinessException` → 해당 ErrorCode의 httpStatus + ErrorResponse.
   - `MethodArgumentNotValidException`/`ConstraintViolationException` → `VALIDATION_ERROR`(400), 필드별 메시지 취합.
   - Spring Security `AccessDeniedException` → `ACCESS_DENIED`(403)(또는 SecurityConfig의 EntryPoint에서 처리 — step3와 일관).
   - 그 외 `Exception` → `INTERNAL_ERROR`(500). 스택트레이스는 로그만, 응답엔 비노출.
   - 모든 응답에 MDC의 traceId 포함.

로깅은 traceId 포함 구조적 로그(logback 패턴에 `%X{traceId}`).

## Acceptance Criteria

```bash
./gradlew test
```
- 단위/슬라이스 테스트: 임의 `@RestController`에서 각 예외를 던졌을 때 (a) HTTP status, (b) body의 `code`, (c) `traceId` 존재를 단언. 최소 BusinessException 1종 + validation 1종 + 미처리 예외 1종.

## 검증 절차

1. AC 실행.
2. 체크리스트: ErrorCode 표의 코드가 누락 없이 enum에 있는가? 응답 포맷이 ARCHITECTURE.md 예시와 일치하는가? traceId가 응답·로그에 모두 나타나는가?
3. 결과를 `phases/fcfs-core/step1.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "common: ErrorCode(전수 enum), BusinessException, ErrorResponse/ApiResponse, TraceIdFilter(MDC+X-Trace-Id), GlobalExceptionHandler(@RestControllerAdvice). 이후 모든 도메인은 BusinessException(ErrorCode)로 에러를 던진다." }`

## 금지사항

- 컨트롤러/서비스에서 직접 응답 JSON을 만들지 마라. 반드시 BusinessException(ErrorCode)로 던져라. 이유: 일관된 에러 계약 유지.
- 도메인(회원/상품/신청) 로직을 만들지 마라. 이유: 이 step은 횡단 인프라 전용.
- `index.json`을 수정하지 마라.
