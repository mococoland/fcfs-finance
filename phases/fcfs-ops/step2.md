# Step 2 (fcfs-ops): demo-ui-and-portfolio

검증용 최소 데모 UI, CI 파이프라인, 포트폴리오용 README를 만든다. 마무리 step.

## 읽어야 할 파일

- `/docs/UI_GUIDE.md` — 색상/컴포넌트/안티패턴(전부 준수)
- `/docs/ARCHITECTURE.md` — "CI(GitHub Actions)", API 명세(데모가 호출할 엔드포인트), 디렉토리
- `/docs/PRD.md` — 목표·성공 지표(README 서사용)
- 기존 API: `/api/members`, `/api/auth/login`, `/api/products`, `/api/products/{id}/apply`, `/api/products/{id}/apply/status`

## 작업

1. **데모 UI** `src/main/resources/static/index.html`(+인라인 또는 분리 JS):
   - 흐름: 회원가입/로그인 → 상품 목록(잔여수량·금리·상태) → 신청 버튼 → **신청 결과 메시지**(접수/마감/중복/기간외, UI_GUIDE 시맨틱 색) → `apply/status` 짧은 폴링으로 PENDING→CONFIRMED 전이 표시.
   - Vanilla JS + `fetch`, 토큰 `localStorage`. Spring 정적 리소스(동일 출처, CORS 불필요).
   - UI_GUIDE 준수(무채색+포인트, 안티패턴 금지, 좌측정렬, 애니메이션 없음).
2. **Swagger**: springdoc 설정 확인/보강(`/swagger-ui.html`), 주요 엔드포인트에 요약 어노테이션.
3. **CI** `.github/workflows/ci.yml`: push/PR 트리거, `actions/setup-java`(JDK17) → `./gradlew build test`. embedded-redis라 Redis 서비스 컨테이너 불필요(필요 시 services로 redis). 캐시(gradle) 설정.
4. **README.md** 포트폴리오화:
   - 한 줄 소개 + 문제 정의(선착순 특판 트래픽).
   - **mermaid 아키텍처 다이어그램**(신청 시퀀스: Client→Lua(원자)→Stream→Consumer→DB).
   - 핵심 설계: Redis 원자(Lua)·멱등 3중·비동기 영속(Stream)·캐싱(DB 보호)·Redis fail-fast.
   - **동시성 증빙**: 1,000동시·재고100→성공100·오버셀0(테스트 링크).
   - **부하 결과 요약**: `docs/LOAD_TEST.md` 전후 표 인용.
   - 실행법: `docker-compose up -d` → `./gradlew bootRun` → `/`(데모)·`/swagger-ui.html`.
   - CI 뱃지, 기술 스택, 디렉토리 안내, ADR/PRD 링크.

## Acceptance Criteria

```bash
./gradlew build
```
- 빌드·기존 테스트 통과.
- CI 워크플로 YAML 문법 유효(파싱 가능).
- 데모 페이지가 실제 엔드포인트를 호출하도록 구성됨(경로 일치). 수동 확인 절차를 README에 명시.

## 검증 절차

1. AC 실행.
2. 체크리스트: 데모가 UI_GUIDE를 지키는가(안티패턴 없음)? README에 mermaid 다이어그램·동시성 증빙·부하 요약·실행법이 있는가? CI가 `gradlew build test`를 실행하는가?
3. 결과를 `phases/fcfs-ops/step2.result.json`에 기록:
   - 성공 → `{ "status": "completed", "summary": "static/index.html 데모(로그인→목록→신청→상태폴링, UI_GUIDE 준수), Swagger 보강, .github/workflows/ci.yml(JDK17 gradlew build test), README 포트폴리오화(mermaid 다이어그램·동시성 증빙·부하 요약·실행법·뱃지). fcfs-ops 완료." }`

## 금지사항

- UI_GUIDE의 안티패턴(glass blur, gradient-text, 보라 브랜드, glow 애니메이션 등)을 쓰지 마라.
- 데모를 위해 백엔드 API 계약(경로/상태코드)을 바꾸지 마라. UI를 API에 맞춰라.
- `index.json`을 수정하지 마라.
