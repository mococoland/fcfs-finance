# 아키텍처

## 디렉토리 구조 (레이어드)
```
src/main/java/com/portfolio/fcfs/
├── FcfsApplication.java
├── common/                 # 횡단 관심사
│   ├── exception/          # BusinessException, ErrorCode(enum)
│   ├── response/           # ApiResponse<T>, ErrorResponse
│   ├── web/                # GlobalExceptionHandler(@RestControllerAdvice), TraceIdFilter(MDC)
│   └── config/             # RedisConfig, SecurityConfig, OpenApiConfig, AsyncConfig
├── member/                 # 회원/인증
│   ├── controller/ service/ domain/ repository/ dto/
│   └── auth/               # JwtProvider, JwtAuthenticationFilter
├── product/                # 상품 조회 + 캐시
│   ├── controller/ service/ domain/ repository/ dto/
│   └── cache/              # ProductCacheRepository (cache-aside)
└── fcfs/                   # 선착순 핵심
    ├── controller/         # ApplyController (apply, status)
    ├── service/            # FcfsService(Lua 호출), ApplyResult(enum)
    ├── redis/              # FcfsRedisRepository, apply.lua, StockInitializer
    ├── consumer/           # EnrollmentStreamConsumer, XAutoClaim 복구, DeadLetter
    └── domain/repository/  # Enrollment(JPA)
src/main/resources/
├── application.yml / application-local.yml / application-test.yml
├── db/migration/           # Flyway V1__init.sql (local/prod)
├── lua/apply.lua           # 원자 선착순 스크립트
└── static/index.html       # 최소 데모 UI
loadtest/                   # k6 스크립트
```

## 컴포넌트/데이터 흐름 — 선착순 신청 (핵심 시퀀스)
```
Client ─POST /api/products/{id}/apply (JWT)─▶ ApplyController
  └▶ FcfsService.apply(memberId, productId)
       ├ 1) 기간 검증: 캐시된 product.openAt/closeAt vs now  (실패→PRODUCT_NOT_OPEN/CLOSED)
       ├ 2) EVALSHA apply.lua  KEYS=[stock:{id}, applied:{id}, stream:enroll]  ARGV=[memberId, productId, traceId]
       │      ┌ 원자 실행(단일 스크립트) ────────────────────────────┐
       │      │ if SISMEMBER applied member == 1 → return -1 (중복)   │
       │      │ stock = GET stock; if nil → return -2 (미초기화)      │
       │      │ if stock <= 0 → return 0 (마감)                       │
       │      │ DECR stock; SADD applied member;                     │
       │      │ XADD stream * memberId .. productId .. traceId ..     │
       │      │ return 1 (성공)                                       │
       │      └──────────────────────────────────────────────────────┘
       └ 결과코드 매핑: 1→202 Accepted / 0→409 SOLD_OUT / -1→409 ALREADY_APPLIED / -2→503 NOT_INITIALIZED
                                              │ (성공 시 stream에 당첨 적재됨)
EnrollmentStreamConsumer (Consumer Group, 비동기)
  ├ XREADGROUP 신규 + XAUTOCLAIM(idle pending) 읽기
  ├ 레코드 단위 멱등 INSERT Enrollment  (member_id+product_id UNIQUE; 충돌 시 무시)
  ├ 성공 → XACK (+ 주기적으로 acked min-id 기준 트림)
  └ N회 실패 → Dead-Letter stream(dlq:enroll) 이동 + 경고 로그/메트릭
MySQL ◀─ 당첨자 N건만 기록 (실패/중복/마감 요청은 절대 도달 안 함)
```

## 데이터 모델 (ERD / 제약)
```
member( id PK BIGINT, email VARCHAR UNIQUE NOT NULL, password_hash NOT NULL, created_at TIMESTAMP )
product( id PK BIGINT, name,
         interest_rate DECIMAL(5,2) NOT NULL,      -- 금리: BigDecimal (double 금지)
         total_quantity INT NOT NULL,
         remaining_quantity INT NOT NULL,          -- naive(DB락) baseline 비교 전용 컬럼
         open_at TIMESTAMP NOT NULL, close_at TIMESTAMP NOT NULL,  -- UTC(Instant)
         created_at TIMESTAMP )
enrollment( id PK BIGINT,
            member_id FK→member NOT NULL,
            product_id FK→product NOT NULL,
            status VARCHAR,                          -- @Enumerated(STRING): CONFIRMED
            applied_at TIMESTAMP, created_at TIMESTAMP,
            UNIQUE(member_id, product_id) )          -- BR-1/BR-6 멱등성의 DB측 최후 방어선
```
- **금융 정밀도**: 금리/금액은 `BigDecimal`/`DECIMAL`. 부동소수(`double/float`) 사용 금지.
- **시간**: 모든 시각은 **UTC `Instant`**(또는 `OffsetDateTime`)로 저장·비교. `LocalDateTime`+서버TZ 의존 금지.
- **enrollment의 UNIQUE(member_id, product_id)** 가 컨슈머 멱등성의 안전망. 컨슈머가 같은 메시지를 두 번 처리해도 DB가 중복을 거절 → 무시하고 ACK.
- `remaining_quantity` 는 **naive baseline 전용**(Redis 버전은 `stock:{id}` 사용). 두 경로가 같은 상품에 동일 조건으로 부하받아 공정 비교.

## Redis 키 스키마
| 키 | 타입 | 용도 | TTL/생명주기 |
|---|---|---|---|
| `stock:{productId}` | String(int) | 잔여 재고 카운터(진실원본) | **TTL 없음·evict 금지**. 워밍은 **SETNX**(재시작 시 덮어쓰기 금지) |
| `applied:{productId}` | Set | 신청자 memberId 집합(중복 차단) | **evict 금지**. 신청 종료 후 정리(예: closeAt+α TTL) |
| `stream:enroll` | Stream | 당첨 확정 큐(**단일 글로벌 스트림**, field에 memberId·productId·traceId) | 길이 ≤ Σ수량. acked min-id 기준 트림. **evict 금지** |
| `product:meta:{productId}` | String(JSON) | 상품 메타 캐시(cache-aside) | TTL 짧게(예 60s) + 갱신 시 무효화. evict 허용 |
| `dlq:enroll` | Stream | Dead-Letter(영속 실패 건) | 운영자 확인용, 수동 정리 |

- **단일 글로벌 스트림** 채택: 컨슈머 그룹 1개로 단순 관리. productId는 메시지 필드로 구분(Lua의 `KEYS[3]=stream:enroll`과 일치).
- **eviction 정책**: `stock/applied/stream`은 무결성 핵심이라 절대 evict되면 안 됨. Redis `maxmemory-policy`를 `noeviction`으로 두거나 이 키들에 TTL을 걸지 않는다. `product:meta`만 캐시라 evict 허용.

## Lua 원자 스크립트 (`apply.lua`) — 설계 불변식
- **중복체크 → 재고체크 → 차감 → 신청자등록 → 큐적재**를 **단일 스크립트(원자)** 로. 부분 실패 구간/보상 트랜잭션 불필요.
- 반환: `1`=성공, `0`=마감, `-1`=중복, `-2`=미초기화. 음수 재고 진입 불가(`stock<=0` 선검사).
- Spring: `DefaultRedisScript<Long>` + EVALSHA 캐싱.

## 캐싱 전략 (RDB 부하 차단)
- **상품 메타**: cache-aside. 미스 시 DB→캐시 적재, 짧은 TTL. 상품 변경 시 키 무효화.
- **재고/신청 판정**: 전적으로 Redis에서. **신청 경로는 DB를 전혀 치지 않는다.**
- **캐시 스탬피드 방지**: 짧은 TTL + 기동 시 워밍(`StockInitializer`) + (옵션) 단일 리로드 뮤텍스.
- **워밍/재구축**: 기동 시 진행중 상품의 `stock` 을 `SETNX (total - 이미당첨수)`. Redis FLUSH/장애 후 재구축은 `total_quantity - count(enrollment)` 로 stock을, applied set은 enrollment의 member들을 SADD로 재구성.

## 에러 핸들링 표준 (횡단)
- **`ErrorCode`(enum)**: `{ code(String), httpStatus, message }`.
- **`BusinessException(ErrorCode)`** → `@RestControllerAdvice GlobalExceptionHandler` 가 일괄 변환.
- **표준 에러 응답 `ErrorResponse`**: `{ timestamp, status, code, message, path, traceId }`. 예:
  ```json
  { "timestamp":"2026-06-03T09:00:00Z", "status":409, "code":"FCFS_SOLD_OUT",
    "message":"마감된 상품입니다.", "path":"/api/products/1/apply", "traceId":"a1b2c3" }
  ```
- **성공 응답 래퍼 `ApiResponse<T>`**: `{ data, traceId }` (선택).
- **TraceId**: `TraceIdFilter`가 MDC에 UUID 주입 → 로그/응답 상관관계.
- 검증 실패(`@Valid`)는 `MethodArgumentNotValidException` 핸들러가 필드별 메시지로 변환(400).

### ErrorCode ↔ HTTP 매핑 (전수)
| code | HTTP | 상황 |
|---|---|---|
| `VALIDATION_ERROR` | 400 | 요청 바디/파라미터 검증 실패 |
| `AUTH_INVALID_CREDENTIALS` | 401 | 로그인 자격 불일치(이메일 존재 여부 비노출) |
| `AUTH_TOKEN_MISSING` / `AUTH_TOKEN_INVALID` / `AUTH_TOKEN_EXPIRED` | 401 | JWT 없음/위조/만료 |
| `ACCESS_DENIED` | 403 | 권한 없음 |
| `MEMBER_EMAIL_DUPLICATED` | 409 | 회원가입 이메일 중복 |
| `PRODUCT_NOT_FOUND` | 404 | 상품 없음 |
| `PRODUCT_NOT_OPEN` | 409 | 신청 시작 전 |
| `PRODUCT_CLOSED` | 409 | 신청 종료 후 |
| `FCFS_SOLD_OUT` | 409 | 재고 소진(Lua 0) |
| `FCFS_ALREADY_APPLIED` | 409 | 중복 신청(Lua -1) |
| `FCFS_NOT_INITIALIZED` | 503 | 재고 카운터 미초기화(Lua -2) |
| `REDIS_UNAVAILABLE` | 503 | Redis 연결 장애(fail-fast) |
| `RATE_LIMITED` | 429 | 과도한 요청(옵션) |
| `INTERNAL_ERROR` | 500 | 예기치 못한 예외 |

## API 명세 (요약)
| Method | Path | 인증 | 성공 | 주요 에러 |
|---|---|---|---|---|
| POST | `/api/members` | - | 201 | 409 EMAIL_DUPLICATED, 400 |
| POST | `/api/auth/login` | - | 200 +JWT | 401 INVALID_CREDENTIALS |
| GET | `/api/products` | - | 200 | - |
| GET | `/api/products/{id}` | - | 200 | 404 |
| POST | `/api/products/{id}/apply` | ✅ | **202** | 409 SOLD_OUT/ALREADY/CLOSED/NOT_OPEN, 503, 401 |
| GET | `/api/products/{id}/apply/status` | ✅ | 200 (상태값) | 401, 404 |

## 장애/복구 모드 (Failure Modes)
| 시나리오 | 동작 |
|---|---|
| Redis 다운 | 신청 즉시 503 fail-fast. **DB 폴백 금지**(stampede 유발). 조회는 캐시 미스→DB(저빈도라 허용) |
| 컨슈머 크래시 | 재기동 시 `XAUTOCLAIM`으로 pending 회수·재처리. 멱등 INSERT라 안전 |
| DB 다운 | 컨슈머가 ACK 안 함 → Stream에 버퍼링 + 재시도(백오프). 신청 접수(Redis)는 계속 가능 |
| 컨슈머 INSERT 유니크 충돌 | 이미 영속화된 것 → 무시하고 ACK(멱등) |
| XADD 실패 | Lua 원자 실행이라 재고차감과 XADD가 동시 성공/실패 → 부분 실패 없음 |
| Redis 데이터 유실(FLUSH) | 기동 reconciliation으로 `stock = total - count(enrollment)` 재구축, applied set 재구성 |
| 메시지 영속 N회 실패 | `dlq:enroll` 로 이동 + 메트릭/로그 경고 |
| 배치 INSERT 중 일부 유니크 충돌 | 배치 전체 롤백 금지 → **레코드 단위 멱등 처리**(충돌 건만 무시ACK, 나머지 정상 영속) |
| 클라이언트 응답 유실(202 못 받음) | 재시도 시 `ALREADY_APPLIED(409)` → 클라이언트는 성공으로 간주, status로 확정 확인(멱등) |

## 관측성 / 보안
- **Actuator + Micrometer**: 커스텀 카운터(apply_success/sold_out/duplicate), stream lag(XLEN−delivered), 컨슈머 처리량/지연. 노출은 `health`만 public, 그 외(`metrics`,`prometheus` 등)는 인증/내부망 한정.
- **로깅**: traceId 포함 구조적 로그. **traceId를 stream 메시지 필드로 실어 컨슈머 MDC에 복원** → 신청~확정 비동기 구간 상관관계 유지. 신청 결과 코드 집계.
- **보안**: JWT 검증 필터, 비밀번호 BCrypt, 에러 메시지에 민감정보·이메일 존재여부 비노출, (옵션) Redis 토큰버킷 rate-limit.

## 상태 관리
- 진실원본: 재고/신청자=Redis, 당첨확정 영속=MySQL. 상태 조회는 (applied set ∈ ?) × (enrollment ∈ ?) 조합으로 4상태 판정:
  - applied ∈ & enrollment ∈ → `CONFIRMED`
  - applied ∈ & enrollment ∉ → `PENDING`(컨슈머 미반영)
  - applied ∉ & (마감) → `FAILED_SOLD_OUT`
  - applied ∉ & (미신청) → `NOT_APPLIED`

## 보안 / 인증 상세
- **엔드포인트 접근 매트릭스**:
  - `permitAll`: `POST /api/members`, `POST /api/auth/login`, `GET /api/products/**`, `/swagger-ui/**`, `/v3/api-docs/**`, `/actuator/health`, 정적 리소스(`/`, `/index.html`)
  - `authenticated`: `POST /api/products/{id}/apply`, `GET /api/products/{id}/apply/status`
  - 그 외 `denyAll`(기본).
- **JWT**: HS256, 시크릿은 `application.yml`의 `${JWT_SECRET}`(env 주입, 기본값은 local 전용). access 토큰만(만료 예: 1h, 부하테스트 동안 유효하도록 충분히). `Authorization: Bearer` 헤더. 만료/위조→401(`AUTH_TOKEN_*`).
- **CORS**: 데모 UI를 Spring 정적 리소스(`/static`)로 **동일 출처 서빙** → CORS 불필요. (별도 프론트 서버 안 씀.)
- **Stateless**: `SessionCreationPolicy.STATELESS`, CSRF 비활성(토큰 기반).

## 성능 튜닝 (10k 동시 대비)
- **Tomcat**: `server.tomcat.threads.max` 상향(예 200~400), accept count 조정.
- **HikariCP**: 신청 경로가 DB를 안 치므로 풀은 **작게**(컨슈머·조회용, 예 10~20). 과대 풀은 MySQL 커넥션 고갈 유발.
- **Lettuce**: 공유 커넥션(논블로킹) 사용, 필요 시 풀링.
- **Redis**: 단일 스레드라 Lua는 짧게 유지(O(1) 명령만). 대형 Lua/루프 금지.

## 부하테스트 관측 & 병목 분석 방법론
"병목을 찾았다"를 **증거로** 뒷받침하기 위해, 부하 중 4계층을 동시에 계측한다.
| 계층 | 관측 지표 | 도구 |
|---|---|---|
| 클라이언트 | TPS, p50/p95/p99, error rate, 상태코드 분포 | k6 summary |
| 애플리케이션 | Tomcat busy threads, **Hikari active/pending**, GC, 힙, 커스텀 카운터 | Actuator/Micrometer, `/actuator/metrics` |
| Redis | ops/sec, command latency, 연결 수, blocked clients | `redis-cli INFO`, `--latency` |
| MySQL | active connections, lock waits, slow query, QPS | `SHOW ENGINE INNODB STATUS`, `performance_schema` |
| 시스템 | CPU/메모리/네트워크 (컨테이너별) | `docker stats` |

**예상 병목 가설(검증 대상)** — 문서에 가설→측정→결론으로 서술:
- **naive(DB락) baseline**: `SELECT FOR UPDATE` 직렬화로 **Hikari 풀 고갈 + InnoDB 락 대기** → TPS 천장, p95 급등. (병목=DB 커넥션/락)
- **Redis 버전**: DB 미접근이라 그 병목 소멸 → 다음 병목은 **Tomcat 스레드 수 / Lettuce 단일 커넥션 / 네트워크 RTT**로 이동. 스레드·커넥션 풀 튜닝으로 개선폭 측정.
- 개선 서사: *baseline 측정 → 병목 특정(연결/락) → Redis 원자화로 DB 제거 → 재측정 → 잔존 병목(스레드) 튜닝 → 재측정*. 각 단계 수치를 `docs/LOAD_TEST.md` 전후 표에 기록.

## 테스트 전략 (테스트 피라미드)
- **단위**: 서비스/도메인 로직, ErrorCode 매핑, JwtProvider. 빠르고 다수.
- **통합(슬라이스/SpringBootTest)**: 리포지토리(H2), 컨트롤러(MockMvc), 캐시 적중, 컨슈머 e2e. embedded-redis 사용.
- **동시성(핵심 증빙)**: `ExecutorService(nThreads)` + `CountDownLatch`(start gate)로 N스레드가 **동시에** `FcfsService.apply` 호출 → `AtomicInteger`로 성공/마감/중복 집계 → **성공==재고 && 오버셀==0 && 중복==0** 단언. embedded-redis(실 redis)에서 Lua 원자성 실제 검증.
- **회귀**: 위 전부 `./gradlew test`로 CI에서 매 push 실행.

## CI (GitHub Actions)
- `.github/workflows/ci.yml`: push/PR 시 JDK17 셋업 → `./gradlew build test` 실행. embedded-redis라 Redis 서비스 컨테이너 불필요(또는 services로 redis 기동). 그린 뱃지를 README에.

## 시드 / 부하테스트 데이터 (필수 — 없으면 테스트 불가)
- **상품 시드**: `local` 프로파일 기동 시 `CommandLineRunner`(또는 Flyway seed)로 **지금 열려있는**(`openAt ≤ now < closeAt`) 특판 상품 N개 생성(예: 수량 100/1,000/10,000짜리). 멱등 시드(존재 시 skip).
- **회원 시드**: 부하용으로 회원 M명 대량 생성(예 10,000). BCrypt 비용↑로 대량 생성이 느릴 수 있으니 **시드 전용 빠른 경로**(낮은 cost 또는 사전 해시) 사용.
- **k6 인증 전략**: BR-1(1인1신청) 때문에 VU마다 **고유 회원**이 필요.
  - k6 `setup()`에서 시드된 회원 풀로 로그인해 토큰 배열을 확보하거나, VU 인덱스(`__VU`)로 `loadtest+{i}@example.com`을 로그인 → 토큰 사용.
  - 동일 토큰 재사용 시 두 번째 요청부터 `ALREADY_APPLIED(409)`가 나오므로 **VU=회원 1:1 매핑**을 보장.

## 패턴
- 레이어드 아키텍처(Controller → Service → Repository/Redis). 도메인 패키지 간 내부 직접 참조 금지.
- 진실원본 분리: 실시간 재고/신청 판정 = Redis, 영속 기록 = MySQL(비동기).
