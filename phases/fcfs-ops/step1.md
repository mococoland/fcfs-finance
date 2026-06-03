# Step 1 (fcfs-ops): load-test

k6 부하 테스트와 병목 분석 문서를 만든다. 포트폴리오 어필 포인트 #3의 증거를 생산한다.

## 읽어야 할 파일

- `/docs/ARCHITECTURE.md` — "부하테스트 관측 & 병목 분석 방법론"(4계층 계측·가설), "시드/부하테스트 데이터"(회원 시드, k6 인증 전략 VU=회원 1:1)
- `/docs/ADR.md` — ADR-010(naive DB락 baseline 대조)
- `/docs/PRD.md` — NFR 목표치(10k VU, p95<200ms, 오버셀 0)
- 기존: `POST /api/products/{id}/apply`, `POST /api/auth/login`, `POST /api/members`, `Product`(remaining_quantity), 상품 시드

## 작업

1. **회원 대량 시드**: 부하용 회원 M명(예 10,000) 생성 경로. `loadtest+{i}@example.com` 규칙. BCrypt 비용 때문에 **시드 전용 빠른 경로**(낮은 cost 또는 사전 계산 해시)로 생성. `@Profile("local")` 시더 또는 별도 SQL/스크립트. 멱등.
2. **k6 스크립트** `loadtest/fcfs.js`:
   - `setup()`: 시드 회원으로 로그인해 토큰 배열 확보(또는 VU 인덱스로 매핑). **VU=회원 1:1**(ALREADY_APPLIED 오탐 방지).
   - 시나리오: 점진 ramp-up + **10k 스파이크**. `thresholds`로 `http_req_duration{p(95)}`·`http_req_failed`를 pass/fail 기준화.
   - 신청 결과 분포(202/409 SOLD_OUT/409 ALREADY)를 태그·체크로 집계.
   - `handleSummary`/`--summary-export`로 결과 JSON 저장.
3. **`loadtest/README.md`**: 실행법(docker-compose up → bootRun → 시드 → `k6 run`), 4계층 관측 명령(`redis-cli INFO`, `/actuator/metrics`, `SHOW ENGINE INNODB STATUS`, `docker stats`).
4. **`docs/LOAD_TEST.md`**: 시나리오, **4계층 관측 결과**, 가설→측정→결론, TPS/p95/에러율, **개선 전후 비교표**, 오버셀 0 재확인. (수치는 실행 환경값 placeholder로 두되 표 구조·해석 틀을 완성.)
5. **(선택) `apply-naive` baseline**(ADR-010): `POST /api/products/{id}/apply-naive` — `remaining_quantity` 컬럼을 비관적 락(`SELECT ... FOR UPDATE`)으로 차감 + enrollment 동기 INSERT. 동일 k6 부하로 대조. 시간 부족 시 생략하고 LOAD_TEST.md에 '향후과제'로 명시.

## Acceptance Criteria

```bash
./gradlew test
k6 inspect loadtest/fcfs.js   # (k6 설치 시) 스크립트 문법 검증
```
- `./gradlew test`: (naive 구현 시) 동시성 테스트로 naive도 오버셀 0 보장 단언(락이 정확한지). 기존 테스트 무손상.
- k6 스크립트가 `k6 inspect`/dry-run을 통과(미설치면 이 AC는 blocked로 보고하고 설치 안내).
- `docs/LOAD_TEST.md`와 `loadtest/README.md` 존재, 전후 비교표·관측 방법론 포함.

## 검증 절차

1. AC 실행. k6 미설치 시 해당 부분은 `blocked`로 보고(설치 후 재실행).
2. 체크리스트: VU=회원 1:1이 보장되는가? thresholds로 자동 판정되는가? LOAD_TEST.md에 가설→측정→결론·전후 표가 있는가? 부하 중 오버셀 0 확인 시나리오가 있는가?
3. 결과를 `phases/fcfs-ops/step1.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "회원 대량 시드, loadtest/fcfs.js(k6 setup 토큰풀 VU=회원1:1, ramp+10k스파이크, thresholds, summary-export), loadtest/README, docs/LOAD_TEST.md(4계층 관측·가설→측정→결론·전후 비교표). (apply-naive baseline: 구현/향후과제 표기)" }`
   - k6 미설치 → `{ "status": "blocked", "blocked_reason": "k6 CLI 미설치 — 설치 후 k6 inspect/run 검증 필요" }`

## 금지사항

- 모든 VU가 같은 회원 토큰을 쓰게 하지 마라. 이유: 2번째부터 ALREADY_APPLIED만 나와 부하 무의미.
- 부하 결과로 오버셀이 1건이라도 나면 통과로 보고하지 마라. 무결성 불변식 위반.
- 기존 Redis 신청 경로를 naive로 바꾸지 마라. naive는 **별도 엔드포인트**(대조군)일 뿐.
- `index.json`을 수정하지 마라.
