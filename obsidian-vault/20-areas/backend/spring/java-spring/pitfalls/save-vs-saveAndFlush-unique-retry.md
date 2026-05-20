---
title: "JpaRepository.save() 는 즉시 flush 안 함 — ConstraintViolationException 을 service 에서 못 잡는다"
kind: pitfall
status: stable
created_at: 2026-05-20T07:00:00+09:00
tags: [pitfall, spring-data-jpa, transactional, flush, retry]
related:
  - "[[pitfalls]]"
  - "[[../../../../10-projects/bkurs-be/task-logs/task-log-b3a-academy-management]]"
home_hub: java-spring
---

# JpaRepository.save() 는 즉시 flush 안 함

## 한 줄 요약

`JpaRepository.save()` 는 entity 를 영속 컨텍스트에 등록만 한다. 실제 SQL `INSERT` 는 **트랜잭션 commit 시점 (또는 명시적 flush)** 에 실행 → service 안에서 `try { save(); } catch (DataIntegrityViolationException)` 패턴이 작동 안 한다. 재시도 / fallback 시나리오면 `saveAndFlush()` 써라.

## 왜 이렇게 동작하나

JPA 의 write-behind 패턴:
- `save()` → `EntityManager.persist()` → 영속 컨텍스트만 변경
- 실제 INSERT 는 트랜잭션 commit / 명시적 flush / 같은 영속 컨텍스트 안의 SELECT 가 일관성 보장 필요할 때 (auto flush)

성능 최적화 (여러 변경을 한 트랜잭션에서 batch insert) 가 default 동작.

## 안 하면 뭐가 문제

bkurs-be 의 `AcademyCodeService.issue()`:
```java
@Transactional
public AcademyCode issue(Long ownerAccountId) {
    Academy academy = ...;
    for (int i = 0; i < MAX_RETRY; i++) {
        String code = randomCode();
        try {
            return repository.save(AcademyCode.builder()
                .academyId(academy.getId())
                .code(code)
                .build());
        } catch (DataIntegrityViolationException e) {
            // uk_academy_code 충돌 — 재시도
        }
    }
    throw new IllegalStateException(...);
}
```

증상:
- `repository.save()` 는 영속 컨텍스트 등록만 → catch 안 잡힘
- 트랜잭션 commit 시점에 INSERT 시도 → DB 충돌 → ConstraintViolationException 발생 → controller / global handler 까지 propagation
- 재시도 코드 무력화 — 첫 충돌이 곧바로 사용자 응답으로 나감

운영 영향: 32^8 ≈ 1.1조 조합이라 실제 충돌이 거의 없지만, 충돌 발생 시 retry 가 안 됨 = "한 번도 충돌 안 나야 함" 이 운영 가정이 되어버림.

## 해결

### A. `saveAndFlush()` 사용 — 권장

```java
return repository.saveAndFlush(entity);  // ← 즉시 INSERT 트리거
```

- 충돌이 즉시 service 안에서 catch 가능 → 재시도 / fallback 로직 정상 동작.
- 단점: batch insert 최적화 무효화 → 같은 트랜잭션 안에서 여러 INSERT 가 1 round-trip 못 함. 단건 처리에는 무관.

### B. `entityManager.flush()` 명시 호출

```java
@PersistenceContext EntityManager em;

repository.save(entity);
em.flush();
```

- 동일 효과. saveAndFlush 가 더 단순.

### C. 충돌 자체를 회피 — DB 시퀀스 / UUID

- 코드가 사람이 읽는 형식 (포스터 / 학원 홍보) 이면 UUID 부적합 (너무 길고 입력 부담).
- random + 재시도가 사람이 보는 코드의 표준.

## 진단 신호

- `try-catch (DataIntegrityViolationException)` 가 절대 안 들어옴 → save() 의심
- 트랜잭션 commit 시 ConstraintViolation 이 caller 까지 propagation
- Hibernate SQL 로그 (`org.hibernate.SQL=DEBUG`) 에서 INSERT 가 트랜잭션 끝에 한꺼번에 나옴

## 참고

- Spring Data JPA reference §`save` vs `saveAndFlush`
- bkurs-be PR #10 — `AcademyCodeService.issue()` 의 saveAndFlush 패턴
- [[transactional-rollback-noRollbackFor]] — 같은 트랜잭션 함정 카테고리
