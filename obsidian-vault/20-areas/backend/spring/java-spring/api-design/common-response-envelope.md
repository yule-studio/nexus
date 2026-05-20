---
title: "공통 Response Envelope (ApiEnvelope) 표준"
kind: guideline
area: backend/spring/java-spring/api-design
status: standard
created_at: 2026-05-20T22:15:00+09:00
agent: engineering-agent/tech-lead
tags: [api-design, response, envelope, java-spring, standard]
related:
  - "[[api-design]]"
  - "[[../pitfalls/pitfalls]]"
  - "[[../../../../10-projects/bkurs-be/architecture-hybrid-external-services]]"
sources:
  - "https://github.com/codwithyc/spring-sandbox"
home_hub: api-design
---

# 공통 Response Envelope 표준

> 모든 REST API 응답을 단일 envelope 으로 감싸 FE / 다중 client 가 일관된
> 구조로 처리. **success/data/error/meta** 4 필드 고정.
> 기준 구현: [codwithyc/spring-sandbox](https://github.com/codwithyc/spring-sandbox).

## 1. 왜 envelope 인가

### 장점

- **FE 단일 처리 경로**: success 분기 1번 + error 분기 1번. 도메인 별 분기 X.
- **추적성**: meta.requestId 가 모든 응답 + 로그에 동일 → 장애 조사 5초.
- **버전 변경 영향 최소**: data 안 schema 만 바꿔도 envelope 외피는 stable.
- **error 표준화**: code (machine), messageKey (i18n), message (사람) 3 layer.
- **fieldErrors**: Bean Validation 결과를 form field 별 노출 → FE 즉시 UI 매핑.

### 단점 (vs raw entity 응답)

- 한 단계 더 깊어짐 (`body.data.X`).
- 모든 controller method signature 가 `ResponseEntity<ApiEnvelope<T>>`.

→ **trade-off 우위**: 단점 (depth 1) 보다 장점 (추적/일관/i18n) 이 훨씬 큼.

## 2. 표준 응답 shape

### 성공
```json
{
  "success": true,
  "data": { ... 도메인 응답 ... },
  "error": null,
  "meta": {
    "requestId": "req-a1b2c3d4",
    "timestamp": "2026-05-20T13:34:56.789Z"
  }
}
```

### 실패
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "COM-4003",
    "status": 400,
    "message": "입력값 검증에 실패했습니다.",
    "messageKey": "error.common.validation_error",
    "fieldErrors": [
      {"field": "email", "code": "NotBlank", "message": "이메일은 필수입니다"},
      {"field": "password", "code": "Size", "message": "8자 이상"}
    ],
    "requestId": "req-a1b2c3d4"
  },
  "meta": {
    "requestId": "req-a1b2c3d4",
    "timestamp": "2026-05-20T13:34:56.789Z"
  }
}
```

### 핵심 invariant

- `success=true` → `data != null && error == null`
- `success=false` → `data == null && error != null`
- `meta` 는 항상 존재 (성공/실패 무관).
- `204 No Content` 는 envelope 없이 빈 body (브라우저 호환).

## 3. 컴포넌트 구조

```
src/main/java/com/{org}/{app}/common/
  ├── api/
  │   ├── ApiEnvelope.java       // record (success/data/error/meta + invariant)
  │   ├── ApiMeta.java           // record (requestId, timestamp)
  │   └── ApiEnvelopes.java      // @Component (Clock 주입 + 헬퍼)
  ├── error/
  │   ├── ApiError.java          // record (code/status/message/messageKey/fieldErrors/requestId)
  │   ├── ErrorCodeSpec.java     // interface (status/code/messageKey/defaultMessage)
  │   └── FieldErrorItem.java    // record (field/code/message)
  ├── enums/
  │   └── ErrorCode.java         // enum implements ErrorCodeSpec (공통 COM-4xxx/5xxx)
  ├── exception/
  │   ├── BusinessException.java       // 베이스
  │   ├── InvalidInputException.java   // 400
  │   ├── ValidationException.java     // 400 + fieldErrors
  │   ├── UnauthorizedException.java   // 401
  │   ├── ForbiddenException.java      // 403
  │   ├── NotFoundException.java       // 404
  │   ├── ConflictException.java       // 409
  │   └── ServiceUnavailableException.java  // 503
  ├── handler/
  │   └── GlobalExceptionHandler.java  // @RestControllerAdvice
  └── web/
      ├── RequestIds.java        // X-Request-Id header 상수
      └── RequestIdFilter.java   // OncePerRequestFilter — UUID 생성 + MDC 주입
```

## 4. ErrorCode 네이밍

`{도메인}-{HTTP status}{seq:3자리}` 형식.

| 예 | 의미 |
|---|---|
| `COM-4001` | Common, 400, #001 (INVALID_INPUT) |
| `COM-4003` | Common, 400, #003 (VALIDATION_ERROR) |
| `COM-5001` | Common, 500, #001 (INTERNAL_SERVER_ERROR) |
| `ACC-4040` | Account, 404, #0 (ACCOUNT_NOT_FOUND) |
| `PAY-4090` | Payment, 409, #0 (PAYMENT_DUPLICATE) |

- 도메인 prefix 는 3자 (영문 대문자) 권장.
- 같은 도메인 안 ErrorCode 는 1 enum.
- 공통 COM-* 는 envelope 패키지 default 제공.

## 5. Controller 사용 패턴

### 5-1. ok 응답
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v1/students/me")
public class StudentProfileController {

    private final StudentProfileService service;
    private final ApiEnvelopes envelopes;

    @GetMapping
    public ResponseEntity<ApiEnvelope<StudentProfileView>> get(
            @AuthenticationPrincipal Long accountId,
            HttpServletRequest request) {
        return envelopes.ok(StudentProfileView.from(service.get(accountId)), request);
    }
}
```

### 5-2. 201 Created
```java
@PostMapping
public ResponseEntity<ApiEnvelope<StudentProfileView>> create(...) {
    StudentProfile saved = service.create(...);
    URI location = URI.create("/api/v1/students/" + saved.getId());
    return envelopes.created(location, StudentProfileView.from(saved), request);
}
```

### 5-3. 204 No Content
```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    service.delete(id);
    return envelopes.noContent();  // envelope 없음, 표준 204
}
```

### 5-4. 도메인 예외
```java
// service 단에서
if (notFound) throw new NotFoundException(AccountErrorCode.ACCOUNT_NOT_FOUND, "id=" + id);

// controller 는 try/catch 없음 — GlobalExceptionHandler 가 ApiError 로 변환
```

## 6. GlobalExceptionHandler 패턴

```java
@RestControllerAdvice
@RequiredArgsConstructor
@Slf4j
public class GlobalExceptionHandler {

    private final ApiEnvelopes envelopes;

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiEnvelope<Void>> handleBusiness(
            BusinessException ex, HttpServletRequest request) {
        ApiError error = ApiError.of(ex.errorCode(), ex.getMessage(),
            ex.fieldErrors(), requestId(request));
        return envelopes.fail(ex.errorCode().status(), error, request);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiEnvelope<Void>> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        List<FieldErrorItem> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldErrorItem(fe.getField(), fe.getCode(),
                Optional.ofNullable(fe.getDefaultMessage()).orElse("")))
            .toList();
        ApiError error = ApiError.of(ErrorCode.VALIDATION_ERROR, null,
            fieldErrors, requestId(request));
        return envelopes.fail(HttpStatus.BAD_REQUEST, error, request);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiEnvelope<Void>> handleAny(Exception ex, HttpServletRequest request) {
        log.error("Unhandled exception", ex);
        ApiError error = ApiError.of(ErrorCode.INTERNAL_SERVER_ERROR,
            "예상치 못한 오류", null, requestId(request));
        return envelopes.fail(HttpStatus.INTERNAL_SERVER_ERROR, error, request);
    }
}
```

## 7. RequestIdFilter

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
public class RequestIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String requestId = req.getHeader("X-Request-Id");
        if (requestId == null || requestId.isBlank()) {
            requestId = "req-" + UUID.randomUUID().toString().substring(0, 8);
        }
        req.setAttribute(RequestIds.REQUEST_ID_ATTR, requestId);
        res.setHeader("X-Request-Id", requestId);
        MDC.put("requestId", requestId);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("requestId");
        }
    }
}
```

## 8. FE 처리 패턴

```typescript
// shared/api.ts
type ApiEnvelope<T> = {
  success: boolean;
  data: T | null;
  error: ApiError | null;
  meta: { requestId: string; timestamp: string };
};

type ApiError = {
  code: string;
  status: number;
  message: string;
  messageKey: string;
  fieldErrors: { field: string; code: string; message: string }[];
  requestId: string;
};

export async function api<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(path, init);
  const envelope: ApiEnvelope<T> = await res.json();
  if (!envelope.success) {
    throw new ApiClientError(envelope.error!);
  }
  return envelope.data!;
}
```

## 9. 마이그레이션 전략 (기존 raw 응답 → envelope)

기존 프로젝트 적용 시:

1. **foundation 추가** — common/ 패키지 (api/error/enums/exception/handler/web).
2. **신규 endpoint 부터 envelope 강제** — 새 PR 의 controller 는 `ApiEnvelope<T>` 만.
3. **기존 endpoint 도메인 별 점진 migration** — domain unit (controller class) 단위 PR.
4. **FE 동시 migration** — domain unit 별로 BE+FE 함께. mixed 기간 최소화.
5. **회귀 보강** — response shape 검증 (`body.success`, `body.data.x`, `body.error.code`).
6. **migration 완료 시 강제**: lint / archUnit 룰로 `ResponseEntity<ApiEnvelope<T>>` 외 금지.

## 10. 안티 패턴

- ❌ raw entity 반환 (`return entity;`) — envelope 우회.
- ❌ Map<String, Object> 응답 — schema 불명.
- ❌ controller 에서 try/catch 후 직접 ResponseEntity.status(...).body(...) — handler 우회.
- ❌ 도메인 별 다른 error shape (어떤 endpoint 는 `{error: "..."}`, 다른 endpoint 는 `{message: "..."}`).
- ❌ 200 OK 인데 body 에 `{success: false}` — HTTP status 와 envelope.success 불일치.

## 관련

- [[api-design]] — 상위 인덱스
- [[../../../../10-projects/bkurs-be/architecture-hybrid-external-services]] — bkurs 적용 사례
- [[../pitfalls/pitfalls]] — 응답 shape 깨짐 회수
- source: [codwithyc/spring-sandbox](https://github.com/codwithyc/spring-sandbox) common/* 패키지
