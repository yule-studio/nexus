---
title: "@Valid 실패 시 401 응답 — Spring 의 /error 가 permitAll 에 없을 때"
kind: pitfall
status: stable
created_at: 2026-05-20T08:50:00+09:00
tags: [pitfall, spring-security, validation, error-dispatch]
related:
  - "[[pitfalls]]"
  - "[[spring-security-401-vs-403]]"
  - "[[../../../../10-projects/bkurs-be/task-logs/task-log-b4a-academy-info]]"
home_hub: java-spring
---

# @Valid 실패 시 401 응답 — Spring 의 /error 가 permitAll 에 없을 때

## 한 줄 요약

`@Valid + @RequestBody` 검증 실패 → Spring 이 ERROR dispatch 로 `/error` 호출 → SecurityConfig 의 `permitAll` 에 `/error` 없으면 인증 필터가 가로채 **401**. 사용자는 400 받을 줄 알고 디버깅한다. 해결: 명시적 `@ExceptionHandler(MethodArgumentNotValidException)` 가 ERROR dispatch 거치지 않고 곧바로 400 응답.

## 왜 이렇게 동작하나

Spring MVC 의 ERROR dispatch 메커니즘:
- 컨트롤러 메서드에서 unhandled exception → Spring 이 응답 status 설정 + servlet 컨테이너에 ERROR dispatch 요청
- 컨테이너가 `/error` 경로로 내부 forward
- `BasicErrorController` 가 응답 body 생성
- **이 모든 단계가 SecurityFilterChain 재진입**

`/error` 가 `permitAll` 에 없으면 → Spring Security 가 anyRequest().authenticated() 적용 → SecurityContext 가 ERROR dispatch 시점에 다시 평가됨. 일부 환경에서는 인증 정보가 보존되지만, 검증 실패 같은 케이스는 anonymous 처럼 처리되어 401 반환.

## 안 하면 뭐가 문제

bkurs-be 의 `AcademyInfoApiTest.patchInvalidUrl_400`:
```java
// PATCH 잘못된 logoUrl ("not-a-url") → Bean Validation 실패
ResponseEntity<Map> res = patch(ownerToken, Map.of("logoUrl", "not-a-url"));
assertThat(res.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
// expected: 400 BAD_REQUEST
//  but was: 401 UNAUTHORIZED
```

서버 로그:
```
WARN  ... MethodArgumentNotValidException: Validation failed ...
DEBUG ... Completed 400 BAD_REQUEST    ← DispatcherServlet 은 400 응답
DEBUG [Test worker] ... Response 401 UNAUTHORIZED   ← 클라이언트는 401 받음
```

"DispatcherServlet 은 400 으로 끝났는데 왜 클라는 401?" — ERROR dispatch 가 그 사이에 끼어든 결과.

운영 영향:
- 프론트가 validation 에러를 인증 만료로 오인 → 재로그인 시도 → 무한 루프
- 디버깅 시간 폭증 (서버 로그는 400, 클라는 401 — 양쪽 진실이라 원인 추적 어려움)

## 해결 3 선택지

### A. 명시적 `@ExceptionHandler` — 권장

```java
@RestControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)
public class GlobalValidationExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> onValidation(MethodArgumentNotValidException e) {
        Map<String, String> fields = new LinkedHashMap<>();
        e.getBindingResult().getFieldErrors().forEach(fe ->
            fields.put(fe.getField(), fe.getDefaultMessage()));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(Map.of("error", "validation_failed", "fields", fields));
    }
}
```

- 명시적 핸들러가 매칭되면 ERROR dispatch 건너뜀.
- 응답 body 도 클라이언트가 필드별로 파싱 가능.
- bkurs-be B4A 가 택한 방식.

### B. `/error` 를 SecurityConfig `permitAll` 에 추가

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/error", "/api/v1/health", ...).permitAll()
    .anyRequest().authenticated());
```

- 작동하지만 `/error` 가 공개 → 인증 필요한 endpoint 의 500 응답이 anonymous 에게도 노출.
- 비권장.

### C. `ResponseEntityExceptionHandler` 상속

```java
@RestControllerAdvice
public class GlobalHandler extends ResponseEntityExceptionHandler {
    // handleMethodArgumentNotValid override
}
```

- Spring 기본 핸들러 확장. A 와 거의 동등.
- 도메인 specific 핸들러가 같이 있으면 충돌 가능 — A 가 더 깔끔.

## 진단 신호

- 통합 테스트의 `assertThat(status).isEqualTo(BAD_REQUEST)` 실패 + 실제 응답 401
- 서버 로그에 "Completed 400" 와 클라 응답 401 이 동시에 나옴
- `@Valid` 가 붙은 endpoint 만 영향

## 회귀 테스트로 못 박기

```java
@Test
void patchInvalidUrl_400() {
    ResponseEntity<Map> res = patch(token, Map.of("logoUrl", "not-a-url"));
    assertThat(res.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
    assertThat(res.getBody().get("error")).isEqualTo("validation_failed");
}
```

## 참고

- Spring MVC reference §ERROR dispatch
- Spring Security reference §`/error` 경로 처리
- bkurs-be PR #16 — `GlobalValidationExceptionHandler` 도입
- [[spring-security-401-vs-403]] — 같은 401 의미 함정 카테고리
