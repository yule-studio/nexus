---
title: "bkurs-be — 테스트 가이드"
kind: handover
status: stable
created_at: 2026-05-21T22:35:00+09:00
tags: [bkurs-be, handover, testing, integration-test]
related:
  - "[[../bkurs-be]]"
  - "[[directory-structure]]"
  - "[[database-and-migrations]]"
home_hub: bkurs-be
---

# bkurs-be — 테스트 가이드

> "새 도메인 / endpoint 추가 시 회귀 어떻게 쓰나" + "왜 새로 추가했는데 다른 테스트가
> 깨지나". 핵심은 `AbstractIntegrationTest` 가 매 테스트 cleanup 을 자동화한다는 것 + 신규
> entity 추가 시 **본 base 한 곳** 만 보강한다는 것.

## 1. 테스트 구성 한 장

```
src/test/java/com/bkurs/
├── support/AbstractIntegrationTest.java   # 부모 @BeforeEach 가 cleanup + JWT helper
├── auth/AuthApiTest.java                  # 도메인별 *ApiTest extends Abstract
├── academy/AcademyApprovalApiTest.java
├── homework/HomeworkApiTest.java
├── ...                                    # 51 클래스 (대부분 통합)
└── common/                                # envelope / handler 등 unit
```

회귀 카운트: **~295 GREEN** (cycle 1 종료 시점). cycle 2 진행 후 추가됨.

## 2. `AbstractIntegrationTest` 의 역할

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@ActiveProfiles("local")
public abstract class AbstractIntegrationTest {

    @Autowired protected TestRestTemplate rest;            // HTTP 호출
    @Autowired protected TransactionTemplate tx;           // 명시적 트랜잭션
    @Autowired protected PasswordEncoder passwordEncoder;
    @Autowired protected JwtTokenProvider tokenProvider;

    // 30+ Repository @Autowired (cleanup 대상)

    @BeforeEach
    void cleanupAllRepositories() {
        // FK 역순 deleteAll — 자식 → 부모
    }

    protected Account createAccount(Role role, ...) { ... }
    protected HttpHeaders bearer(Account a) { ... }
}
```

핵심:
- `WebEnvironment.RANDOM_PORT` — 실제 톰캣 띄워서 호출. 진짜 통합.
- `@ActiveProfiles("local")` — local profile → Mock gateway 사용.
- JUnit 5 의 부모 `@BeforeEach` 는 **자식 `@BeforeEach` 보다 먼저** — cleanup 후 자식이 fixture 생성.

## 3. 새 테스트 작성 템플릿

```java
package com.bkurs.homework;

import com.bkurs.support.AbstractIntegrationTest;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class HomeworkApiTest extends AbstractIntegrationTest {

    @Test
    void 강사_숙제_생성() {
        Account teacher = createAccount(Role.TEACHER, "t@x.com");
        Academy academy = ...;
        Course course = ...;

        ResponseEntity<ApiEnvelope<HomeworkResponse>> res = rest.exchange(
            "/v1/teachers/me/courses/" + course.getId() + "/homework",
            HttpMethod.POST,
            new HttpEntity<>(req, bearer(teacher)),
            new ParameterizedTypeReference<>() {}
        );

        assertThat(res.getStatusCode().value()).isEqualTo(201);
        assertThat(res.getBody().success()).isTrue();
        assertThat(res.getBody().data().title()).isEqualTo("...");
    }
}
```

체크리스트:
- [ ] `extends AbstractIntegrationTest`
- [ ] **본인 fixture 만** `@BeforeEach` (없으면 생략) — cleanup 은 부모가
- [ ] envelope 응답 검증 (`success / data / error / meta`)
- [ ] role 별 거부 케이스 (401 / 403) 1개 이상 — 자세히 [[auth-and-security#10]]
- [ ] entity uk / invariant 충돌 케이스 1개 (예: 중복 submission)
- [ ] 새 endpoint = 새 회귀 라인. 기존 테스트가 다 통과해도 새 회귀가 없으면 PR reject ([[CLAUDE.md]] §테스트)

## 4. 신규 entity 추가 시 base 보강

**한 곳만 수정**: `AbstractIntegrationTest.cleanupAllRepositories()`.

```java
@Autowired protected MyNewRepository myNewRepository;

@BeforeEach
void cleanupAllRepositories() {
    // ... 기존 자식부터 ...
    myNewRepository.deleteAll();           // ← 자식 위치에 추가 (FK 역순)
    // ... 기존 부모로 ...
}
```

**FK 역순 규칙**:
- `homework_submission` (자식) → `homework` (부모) → `course` (조부모) → `academy` → `account`
- bridge 가 있는 도메인 (performance / discussion / simulation / reading) → record 보다 먼저 delete

## 5. cleanup 순서 위반 시 증상

- `Cannot delete or update a parent row: a foreign key constraint fails` — cleanup 순서 잘못. 자식 먼저.
- previous 테스트의 row 가 남음 → 다음 테스트에서 unique violation → 일부 random fail.
- 본 base 가 도입되기 전 (REFACTOR PR #36) 까지는 매 PR 마다 16 테스트의 cleanup 수동 보강 반복 — 본 base 로 그 패턴 종결.

## 6. test 만의 profile 동작

- gateway: `@Profile("!prod")` 의 Mock 자동 사용. SendGrid / Claude / Clova / PortOne 실 호출 없음.
- batch: `bkurs.batch.scheduling.enabled=false` (테스트 profile 에서 토글) — `@Scheduled` job 자동 실행 회피.
- DB: `application-local.yml` 의 `bkurs-local` (실 MySQL). H2 미사용 — Flyway MySQL 문법 차이.

> 로컬 MySQL 이 없으면 테스트 실패. `brew services start mysql` 또는 `docker compose -f
> docker-compose.dev.yml up -d mysql` 후 테스트.

## 7. 헬퍼 메서드 예

```java
// account 빠르게 만들기
Account teacher = createAccount(Role.TEACHER, "t@x.com");

// JWT bearer header
HttpHeaders h = bearer(teacher);

// 다른 트랜잭션에서 검증 (event listener 동기 검증 등)
tx.execute(status -> {
    StudentRecordEntry e = recordRepo.findBySourceTypeAndSourceId(REC_BRIDGE_PERFORMANCE, subId).orElseThrow();
    assertThat(e.getCategory()).isEqualTo(SUBJECT);
    return null;
});
```

## 8. CI 통합

`./gradlew clean check` 가 모든 통합 테스트 실행. GitHub Actions:

- MySQL 9 service container
- JDK 21 (Eclipse Temurin)
- ENV: `DB_PASSWORD`, `JWT_SECRET` (throwaway secret)
- 실행 시간: 약 1m43s ~ 2분 (cycle 2 시점). PR / dev push 모두 GREEN 필수.

## 9. unit 테스트는 어디?

- `common/` 의 envelope / handler / academic-period 같은 무 DB 검증.
- `ApiEnvelopeTest` (8 unit) — invariant 검증.
- 도메인 한정 unit 은 거의 없음 — service 단계의 cross-domain validation 이 많아 통합으로 검증 효율이 더 좋음.

> 단위 테스트가 정말 필요한 helper (예: `AcademicPeriod.from`, `JwtTokenProvider`) 는 별도
> unit. 도메인 service 는 통합 권장.

## 10. 깨질 때 디버깅 절차

1. **하나만 실행** — `./gradlew test --tests com.bkurs.homework.HomeworkApiTest`. 통합 격리.
2. **로그 출력** — `application-local.yml` 의 `format_sql: true` 가 actual SQL 보여줌.
3. **DB 직접 확인** — 테스트가 종료되어도 (실패 path) row 가 남을 수 있음. cleanup 누락 의심.
4. **새 V 추가 후 깨짐** — `ddl-auto: validate` 가 entity vs schema mismatch 알려줌. column type 일치 확인.
5. **MockMvc vs TestRestTemplate** — 본 repo 는 `TestRestTemplate` 일관. MockMvc 섞지 말 것.

## cross-link

- [[../bkurs-be]]
- [[directory-structure]] — src/test/ 위치
- [[database-and-migrations]] — Flyway / cleanup FK 순서
- [[auth-and-security]] — JWT bearer / role 거부 케이스
- [[common]] — envelope / handler 검증 패턴
