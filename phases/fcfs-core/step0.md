# Step 0: project-setup

Spring Boot 3 / Java 17 / Gradle 프로젝트의 뼈대와 로컬 인프라(docker-compose), 프로파일을 만든다. 아직 도메인 로직은 없다.

## 읽어야 할 파일

먼저 아래를 읽고 스택·규칙·설계 의도를 파악하라:

- `/CLAUDE.md` — 기술 스택, 아키텍처 CRITICAL 규칙, 명령어
- `/docs/ARCHITECTURE.md` — 디렉토리 구조, Redis 키 스키마, 테스트 전략
- `/docs/ADR.md` — ADR-008(테스트=embedded-redis 주력), ADR-009(test=ddl-auto, Flyway는 local/prod)

## 작업

1. **Gradle Spring Boot 프로젝트**를 레포 루트에 생성한다.
   - `build.gradle`: Spring Boot 3.3.x, Java 17 toolchain, group `com.portfolio`, 메인 클래스 `com.portfolio.fcfs.FcfsApplication`.
   - 의존성: `spring-boot-starter-web`, `-data-jpa`, `-data-redis`, `-security`, `-validation`, `-actuator`, `springdoc-openapi-starter-webmvc-ui`, `flyway-core`+`flyway-mysql`, `mysql-connector-j`, `jjwt`(api/impl/jackson).
   - 테스트 의존성: `spring-boot-starter-test`, `spring-security-test`, H2(`com.h2database:h2`), embedded-redis(`com.github.codemonstur:embedded-redis` 또는 동등 라이브러리), `org.testcontainers:junit-jupiter`/`:mysql`(보조).
   - `gradlew`/`gradlew.bat` wrapper 포함.
2. **프로파일별 설정**:
   - `application.yml`: 공통(actuator: `health`만 expose, springdoc 경로). 기본 프로파일은 `local`.
   - `application-local.yml`: MySQL(`localhost:3306`), Redis(`localhost:6379`), Flyway enabled, `ddl-auto: validate`, `JWT_SECRET` env(기본값 local 전용).
   - `application-test.yml`: H2(MySQL 호환 모드 가능), `ddl-auto: create-drop`, **Flyway disabled**, embedded-redis 포트. ADR-009 준수.
3. **docker-compose.yml**(레포 루트): `mysql:8`(DB `fcfs`, healthcheck), `redis:7`(`command: redis-server --maxmemory-policy noeviction`, healthcheck). 앱은 컨테이너로 안 띄워도 됨(로컬 `bootRun` 사용).
4. **FcfsApplication.java** + 빈 패키지 구조(`common`, `member`, `product`, `fcfs`)를 ARCHITECTURE.md대로 생성(빈 패키지는 `package-info.java`로 표시 가능).
5. **헬스 스모크 테스트**: `@SpringBootTest`(test 프로파일)로 컨텍스트 로딩 + `/actuator/health`가 UP인지 확인하는 테스트 1개.

코드 스니펫은 설정/시그니처 수준만. 구현 세부는 재량.

## Acceptance Criteria

```bash
./gradlew build
```
- 컴파일 + 테스트 통과. 컨텍스트 로딩 테스트가 test 프로파일(H2 + embedded-redis)에서 Docker 없이 통과해야 한다.

## 검증 절차

1. 위 AC를 실행한다.
2. 아키텍처 체크리스트:
   - test 프로파일이 H2 `create-drop` + embedded-redis인가(Flyway 비활성)? (ADR-009)
   - docker-compose의 redis가 `noeviction`인가?
   - `health`만 public인가?
3. 결과를 `phases/fcfs-core/step0.result.json`에 기록(index.json 직접 수정 금지):
   - 성공 → `{ "status": "completed", "summary": "Gradle Spring Boot3/Java17 셋업, local/test 프로파일, docker-compose(mysql+redis noeviction), 컨텍스트 스모크 테스트 통과. 패키지: com.portfolio.fcfs.{common,member,product,fcfs}" }`
   - 실패 → `{ "status": "error", "error_message": "..." }`
   - 사용자 개입 필요 → `{ "status": "blocked", "blocked_reason": "..." }`

## 금지사항

- 도메인 로직(엔티티/API)을 만들지 마라. 이유: 이 step은 뼈대·인프라 전용이다.
- 테스트가 Docker나 외부 MySQL/Redis에 의존하게 만들지 마라. 이유: 하네스 자식 세션은 Docker가 없을 수 있다(ADR-008).
- `index.json`을 수정하지 마라. 이유: execute.py가 단독 관리한다.
