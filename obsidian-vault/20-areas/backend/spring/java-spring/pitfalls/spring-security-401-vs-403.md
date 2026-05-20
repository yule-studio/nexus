---
title: "Spring Security default — 인증 실패(401)도 403 으로 내려간다"
kind: pitfall
status: stable
created_at: 2026-05-20T03:00:00+09:00
tags: [pitfall, spring-security, http-status, auth]
related:
  - "[[pitfalls]]"
  - "[[../common/security-config]]"
  - "[[../../../../10-projects/bkurs-be/task-logs/task-log-b2a-account-jwt]]"
home_hub: java-spring
---

# Spring Security default — 인증 실패(401)도 403 으로 내려간다

## 한 줄 요약

Spring Security 6 의 default 동작은 **인증 정보 없음 / 깨진 토큰 / 만료 토큰 모두 403** 으로 응답한다. 이는 표준 HTTP semantic (인증 실패 = 401, 권한 실패 = 403) 과 다르고, 프론트의 "401 이면 토큰 재발급 / 403 이면 화면 제한" 분기를 깨뜨린다.

## 왜 이렇게 동작하나

Spring Security 의 `ExceptionTranslationFilter` 는:
- `AccessDeniedException` → `AccessDeniedHandler` (default: 403)
- `AuthenticationException` → `AuthenticationEntryPoint` (default: `Http403ForbiddenEntryPoint` — 이름이 가리키는 그대로 403)

stateless API 에서는 둘을 명시적으로 분리해야 한다.

## 안 하면 뭐가 문제

- 프론트가 토큰 만료 감지 불가 → refresh 흐름 트리거 안 됨 → 사용자에게 "권한 없음" 만 표시되어 영구 로그아웃처럼 보임
- 권한 부족(403) 과 인증 부족(401) 이 같은 코드 → role 별 endpoint 도입 시 디버깅 시간 폭증
- 클라이언트 retry 로직이 401 기준이면 retry 도 트리거 안 됨

## 해결 (3 선택지)

### A. `HttpStatusEntryPoint(UNAUTHORIZED)` — 권장

```java
http.exceptionHandling(eh -> eh
    .authenticationEntryPoint(
        new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)));
```

- 가장 단순. body 없음 — REST 클라이언트가 status code 만 보면 된다.
- bkurs-be B2A 가 택한 방식.

### B. 커스텀 `AuthenticationEntryPoint`

```java
http.exceptionHandling(eh -> eh
    .authenticationEntryPoint((req, res, ex) -> {
        res.setStatus(401);
        res.setContentType("application/json");
        res.getWriter().write("{\"error\":\"unauthorized\"}");
    }));
```

- body 까지 통일하고 싶을 때.

### C. 글로벌 `@RestControllerAdvice` 로 `AuthenticationException` 포착

- filter chain 보다 뒤에서 잡히므로 일부 케이스 (filter 단계의 실패) 는 못 잡는다 — 단독 사용 비권장.

## 추가 — application code 의 인증 예외도 매핑

`AccountService.verifyCredentials()` 에서 던지는 `InvalidCredentialsException` 은 Spring Security filter 가 모르는 application 예외다 → 글로벌 `@ControllerAdvice` 로 401 매핑 필요.

```java
@RestControllerAdvice(basePackages = "com.bkurs.auth")
class AuthExceptionHandler {
    @ExceptionHandler(InvalidCredentialsException.class)
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    Map<String, String> onInvalidCredentials(InvalidCredentialsException e) {
        return Map.of("error", "invalid_credentials", "message", e.getMessage());
    }
}
```

## 회귀 테스트

bkurs-be `AuthApiTest` 의 3 회귀 라인:
- `loginWithWrongPassword_401` — application 예외 매핑 검증
- `meWithoutBearer_401` — `HttpStatusEntryPoint` 검증
- `meWithInvalidBearer_401` — JwtFilter SecurityContext 미세팅 시 검증

## 참고

- Spring Security reference §`ExceptionTranslationFilter`
- bkurs-be PR #4 — `SecurityConfig` + `AuthExceptionHandler` 도입
- [[../common/security-config]]
