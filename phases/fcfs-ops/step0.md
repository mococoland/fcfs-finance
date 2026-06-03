# Step 0 (fcfs-ops): resilience-observability

Redis 장애 시 안전한 실패(fail-fast)와 운영 관측성(메트릭)을 추가한다. fcfs-core가 완료된 코드 위에서 작업한다.

> 사전 조건: 이 phase는 **fcfs-core의 코드가 포함된 기준 브랜치**에서 실행돼야 한다(feat-fcfs-core를 main에 머지한 뒤 실행 권장). 아래 클래스들이 이미 존재한다고 가정한다.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — "장애/복구 모드"(Redis 다운 fail-fast), "관측성/보안", 성능 튜닝
- `/docs/ADR.md` — ADR-006(Redis 장애 fail-fast, DB 폴백 금지)
- `/CLAUDE.md` — Redis 원자연산·DB 폴백 금지 규칙
- 기존: `fcfs/service/FcfsService`, `fcfs/redis/FcfsRedisRepository`, `common/exception/ErrorCode`(REDIS_UNAVAILABLE, RATE_LIMITED), `common/web/GlobalExceptionHandler`

## 작업

1. **Redis 장애 fail-fast**:
   - 신청 경로에서 Redis 연결 예외(`RedisConnectionFailureException` 등)를 잡아 `BusinessException(REDIS_UNAVAILABLE)`(503)로 변환. GlobalExceptionHandler에 매핑 추가(또는 서비스에서 래핑).
   - **DB로 폴백하지 마라**(ADR-006). 조회(상품)는 캐시 미스 시 DB 허용이지만, 신청 판정은 폴백 금지.
2. **관측성(Micrometer/Actuator)**:
   - 커스텀 카운터: `fcfs.apply.success`, `fcfs.apply.sold_out`, `fcfs.apply.duplicate`, `fcfs.apply.not_initialized`. FcfsService 결과 분기에서 증가.
   - 컨슈머 지표: 처리량, 실패/DLQ 건수, stream lag(`XLEN` − 처리수) 게이지.
   - actuator 노출은 `health`만 public, `metrics`/`prometheus`는 내부/인증 한정(기존 SecurityConfig와 일관).
3. **(선택) rate-limit**: Redis 토큰버킷으로 회원/IP 과도 요청 차단 → `BusinessException(RATE_LIMITED)`(429). 시간 부족 시 생략하고 결과 summary에 '미구현'으로 명시.
4. **성능 설정**: `application.yml`에 Tomcat threads.max(예 200~400), Hikari 풀(작게, 예 10~20), Lettuce 공유 커넥션을 명시(부하테스트 기준선).

## Acceptance Criteria

```bash
./gradlew test
```
- Redis 장애 시뮬레이션(연결 끊김/모킹) → 신청이 **503 REDIS_UNAVAILABLE**, DB 폴백·쓰기 없음 단언.
- 신청 결과별 커스텀 카운터가 증가하는지(Micrometer `MeterRegistry`로 단언).
- 기존 fcfs-core 테스트가 깨지지 않음.

## 검증 절차

1. AC 실행.
2. 체크리스트: Redis 다운 시 503이고 DB 폴백이 없는가? 카운터/게이지가 노출되는가? 보안 노출 범위가 유지되는가?
3. 결과를 `phases/fcfs-ops/step0.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "Redis 장애 fail-fast(503 REDIS_UNAVAILABLE, DB폴백 없음), Micrometer 카운터(apply.success/sold_out/duplicate)·컨슈머/stream lag 지표, Tomcat/Hikari/Lettuce 튜닝값. (rate-limit: 구현/미구현 표기)" }`

## 금지사항

- Redis 장애 시 DB로 폴백해 신청을 받지 마라. 이유: stampede 유발(ADR-006).
- 기존 신청/컨슈머 동작을 바꾸지 마라(지표·예외 변환만 추가).
- `index.json`을 수정하지 마라.
