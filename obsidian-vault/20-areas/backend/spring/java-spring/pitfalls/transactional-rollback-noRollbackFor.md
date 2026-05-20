---
title: "@Transactional default rollback 이 부수효과(commit 필요한 변경)를 묻어버린다"
kind: pitfall
status: stable
created_at: 2026-05-20T04:50:00+09:00
tags: [pitfall, spring-data, transaction, rollback]
related:
  - "[[pitfalls]]"
  - "[[../../../../10-projects/bkurs-be/task-logs/task-log-b2b-refresh-cors]]"
home_hub: java-spring
---

# @Transactional default rollback 이 부수효과를 묻어버린다

## 한 줄 요약

Spring 의 `@Transactional` 은 **모든 unchecked exception (`RuntimeException` 자손) 에서 rollback** 한다. service 메서드가 "DB 변경 + 의도된 예외" 패턴이면 DB 변경이 묻혀버린다. **`noRollbackFor`** 로 막아야 한다.

## 왜 이렇게 동작하나

Spring 의 default rollback rule:
- `RuntimeException` / `Error` → rollback
- checked `Exception` → commit

이 default 는 "예외 = 비정상 = 되돌리기" 라는 안전한 가정이지만, **부수효과 commit + 예외 propagation 둘 다 필요한** 케이스에서는 무너진다.

## 안 하면 뭐가 문제

bkurs-be 의 refresh token reuse detection:
```java
@Transactional  // default — RefreshTokenReuseException 도 rollback 대상
public RotateResult rotate(String rawToken, ...) {
    RefreshToken existing = repo.findByTokenHash(hash).orElseThrow(...);
    if (existing.getRevokedAt() != null) {
        repo.revokeFamily(familyId, now, "REUSE_DETECTED");  // family 전체 revoke
        throw new RefreshTokenReuseException();  // ← @Transactional rollback 트리거
    }
    ...
}
```

증상:
- 회귀 테스트 `reuseRevokedToken_revokesFamily` 가 family 전체 revoke 확인 실패
- 운영에서는: 탈취자가 revoked refresh 를 reuse 해도 family 가 살아남아 정상 사용자도 함께 logout 되지 않음 → **탈취 지속 (보안 의미 무력화)**

회귀가 없었다면 silent — 운영에서 "왜 reuse detection 이 안 먹지?" 같은 디버깅 1주일 가능.

## 해결 3 선택지

### A. `noRollbackFor` 지정 — 권장

```java
@Transactional(noRollbackFor = RefreshTokenReuseException.class)
public RotateResult rotate(...) { ... }
```

- 가장 단순. 의도 명확 ("이 예외만 commit 한다").
- 다른 예외 (InvalidRefreshTokenException 등) 는 기본 rollback 유지.
- bkurs-be B2B 가 택한 방식.

### B. 부수효과를 별도 `REQUIRES_NEW` 트랜잭션으로 분리

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void revokeFamily(String familyId) { ... }
```

- 부수효과가 별도 트랜잭션 → 바깥 rollback 영향 없음.
- 단점: 트랜잭션 분리 비용 (connection 별도). 부수효과가 큰 트랜잭션이면 의미 있지만, single update 면 과함.

### C. 예외를 checked 로 바꾸기

- 단점: API surface 변화 + 호출자가 try/catch 필수 → controller / global handler 보일러플레이트 폭증.
- 비권장.

## 회귀 테스트로 못 박기

```java
@Test
void reuseRevokedToken_revokesFamilyAndReturns401() {
    // ... rotate 정상 1회 ...
    refresh(firstCookie);  // 첫 회전 — token1 revoked(ROTATION), token2 active

    // 같은 (이미 revoked) cookie 로 다시 → reuse
    ResponseEntity<Map> reuse = refresh(firstCookie);
    assertThat(reuse.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);

    // 핵심 — family 전체 revoked (noRollbackFor 검증)
    List<RefreshToken> family = repo.findAll().stream()
        .filter(t -> firstFamily.equals(t.getFamilyId())).toList();
    assertThat(family).allMatch(t -> t.getRevokedAt() != null);
    assertThat(family).anyMatch(t -> "REUSE_DETECTED".equals(t.getRevokeReason()));
}
```

## 진단 신호

- 통합 테스트의 DB assertion 이 "분명히 service 에서 update/revoke 호출했는데 row 가 그대로" → @Transactional rollback 의심.
- application log 에 `Initiating transaction rollback after rule match` (Spring DEBUG) 가 보이면 확정.

## 참고

- Spring reference §`@Transactional` rollback rules
- bkurs-be PR #6 — `RefreshTokenService.rotate()` 의 noRollbackFor
- [[spring-security-401-vs-403]] — 같은 PR 에서 묶여 디버깅된 인증 401/403 함정
