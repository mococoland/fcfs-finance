# Step 3: auth

회원가입·로그인·JWT 인증과 Spring Security 설정을 만든다. 이후 신청 API는 이 인증을 요구한다.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — "보안/인증 상세"(엔드포인트 접근 매트릭스, JWT HS256/`${JWT_SECRET}`, STATELESS, CSRF off), API 명세(`/api/members`, `/api/auth/login`)
- step1: `common/exception`(ErrorCode: AUTH_*, MEMBER_EMAIL_DUPLICATED, ACCESS_DENIED), GlobalExceptionHandler
- step2: `Member`, `MemberRepository`

## 작업

1. **회원가입** `POST /api/members` (`member/controller`,`service`,`dto`):
   - 요청 `{ email, password }` `@Valid`(이메일 형식, 비밀번호 최소 8자).
   - 이메일 중복 시 `BusinessException(MEMBER_EMAIL_DUPLICATED)`(409).
   - 비밀번호 `BCryptPasswordEncoder`로 해시 저장. 201 응답.
2. **로그인** `POST /api/auth/login`:
   - 자격 검증 실패 시 `BusinessException(AUTH_INVALID_CREDENTIALS)`(401). **이메일 존재 여부를 노출하지 마라**(없는 이메일·틀린 비번 동일 메시지).
   - 성공 시 JWT(access) 발급. 200 + `{ accessToken, tokenType: "Bearer" }`.
3. **`auth/JwtProvider`**: HS256, 시크릿 `${JWT_SECRET}`(application.yml), 만료(예: 1h). `generate(memberId)`, `parse/validate(token) → memberId`. 만료/위조 구분해 예외(→ AUTH_TOKEN_EXPIRED / AUTH_TOKEN_INVALID).
4. **`auth/JwtAuthenticationFilter`** (`OncePerRequestFilter`): `Authorization: Bearer` 파싱→검증→`SecurityContext`에 인증 주입. 토큰 없음/위조/만료는 통과시키되 보호 자원 접근 시 401이 되도록(EntryPoint) 처리. TraceIdFilter 이후 순서.
5. **`config/SecurityConfig`**: `SessionCreationPolicy.STATELESS`, CSRF off. **엔드포인트 매트릭스**(ARCHITECTURE.md) 적용:
   - permitAll: `POST /api/members`, `POST /api/auth/login`, `GET /api/products/**`, `/swagger-ui/**`, `/v3/api-docs/**`, `/actuator/health`, 정적 리소스.
   - authenticated: `POST /api/products/*/apply`, `GET /api/products/*/apply/status`.
   - 그 외 denyAll.
   - 인증 실패 EntryPoint/AccessDenied 핸들러 → ErrorResponse(AUTH_TOKEN_* / ACCESS_DENIED) 일관 응답.
   - `PasswordEncoder` 빈(BCrypt).

## Acceptance Criteria

```bash
./gradlew test
```
- MockMvc 통합 테스트: (a) 가입→로그인→발급 토큰으로 보호 엔드포인트 접근 200, (b) 토큰 없이 보호 엔드포인트→401, (c) 위조/만료 토큰→401, (d) 이메일 중복 가입→409, (e) 틀린 비번 로그인→401(이메일 존재여부 비노출 메시지 동일).

## 검증 절차

1. AC 실행.
2. 체크리스트: 매트릭스대로 permit/authenticated가 적용됐는가? 비번 BCrypt인가? 로그인 실패가 이메일 존재여부를 노출하지 않는가? STATELESS·CSRF off인가?
3. 결과를 `phases/fcfs-core/step3.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "회원가입(BCrypt,409중복)/로그인(401,존재여부비노출)/JWT(HS256 ${JWT_SECRET}, JwtProvider.generate/validate→memberId)/JwtAuthenticationFilter/SecurityConfig(STATELESS,CSRF off,엔드포인트 매트릭스). 인증 주체=memberId." }`

## 금지사항

- 로그인 실패에서 "이메일 없음"과 "비밀번호 틀림"을 구분해 노출하지 마라. 이유: 계정 열거 공격.
- 비밀번호를 평문/약한 해시로 저장하지 마라. BCrypt 사용.
- 신청/상품/캐시 로직을 만들지 마라. 이 step은 인증 전용.
- `index.json`을 수정하지 마라.
