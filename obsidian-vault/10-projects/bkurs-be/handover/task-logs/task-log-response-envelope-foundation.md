---
title: "Response Envelope foundation task-log"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T22:40:00+09:00
agent: engineering-agent/coding
tags: [task-log, response, envelope, foundation]
related:
  - "[[../bkurs-be]]"
  - "[[../../../20-areas/backend/spring/java-spring/api-design/common-response-envelope]]"
home_hub: bkurs-be
---

# Response Envelope foundation task-log

## 결과

- 이슈: #65
- PR: #66 (draft, stacked on #64 secret cleanup)
- 브랜치: `chore/issue-65-response-envelope`
- 1 commit (foundation)
- 8 unit PASS + 기존 회귀 영향 없음

## 신규 파일 (19개)

```
src/main/java/com/bkurs/common/
├── api/
│   ├── ApiEnvelope.java       — record (success/data/error/meta + invariant 3종)
│   ├── ApiMeta.java           — record (requestId + timestamp)
│   └── ApiEnvelopes.java      — @Component (Clock 주입, ok/created/noContent/fail 헬퍼)
├── config/
│   └── TimeConfig.java        — @Configuration (Clock systemClock bean)
├── enums/
│   └── ErrorCode.java         — enum implements ErrorCodeSpec (COM-4001~5002, 12종)
├── error/
│   ├── ApiError.java          — record (code/status/message/messageKey/fieldErrors/requestId)
│   ├── ErrorCodeSpec.java     — interface (도메인 별 enum 이 구현)
│   └── FieldErrorItem.java    — record (field/code/message)
├── exception/
│   ├── BusinessException.java — 베이스 (ErrorCodeSpec + fieldErrors)
│   ├── NotFoundException.java         — 404
│   ├── ForbiddenException.java        — 403
│   ├── UnauthorizedException.java     — 401
│   ├── ConflictException.java         — 409
│   ├── InvalidInputException.java     — 400
│   └── ServiceUnavailableException.java — 503
├── handler/
│   └── GlobalExceptionHandler.java    — @RestControllerAdvice (Order.LOWEST_PRECEDENCE)
└── web/
    ├── RequestIds.java        — 상수 (HEADER / REQUEST_ID_ATTR / MDC_KEY)
    └── RequestIdFilter.java   — OncePerRequestFilter (X-Request-Id 생성 + MDC)

src/test/java/com/bkurs/common/api/
└── ApiEnvelopeTest.java       — 8 unit
```

## GlobalExceptionHandler 매핑

| 예외 | ErrorCode | HTTP |
|---|---|---|
| BusinessException | 도메인 ErrorCodeSpec | 도메인 status |
| MethodArgumentNotValidException | COM-4003 VALIDATION_ERROR + fieldErrors | 400 |
| HttpMessageNotReadableException | COM-4002 REQUEST_BODY_NOT_READABLE | 400 |
| AccessDeniedException | COM-4005 ACCESS_DENIED | 403 |
| AuthenticationCredentialsNotFoundException | COM-4004 UNAUTHORIZED | 401 |
| HttpRequestMethodNotSupportedException | COM-4007 | 405 |
| HttpMediaTypeNotAcceptableException | COM-4008 | 406 |
| HttpMediaTypeNotSupportedException | COM-4010 | 415 |
| NoHandlerFoundException | COM-4006 NOT_FOUND | 404 |
| Exception (fallback) | COM-5001 INTERNAL_SERVER_ERROR | 500 + log.error |

## 신규 endpoint 사용 패턴 (B17+)

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/v1/files")
public class FileUploadController {

    private final FileStorageService fileStorage;
    private final ApiEnvelopes envelopes;

    @PostMapping("/upload-url")
    public ResponseEntity<ApiEnvelope<PresignedUrlView>> issuePresignedUrl(
            @Valid @RequestBody PresignedUrlRequest request,
            @AuthenticationPrincipal Long accountId,
            HttpServletRequest httpRequest) {
        PresignedUrlView view = fileStorage.issueUploadUrl(accountId, request);
        return envelopes.ok(view, httpRequest);
    }
}
```

## 선택 이유 — handover

### A. 신규 endpoint 부터 envelope 강제 (기존 153 마이그레이션 분리)

- **선택 이유**: 153 endpoint 마이그레이션 = 큰 회귀 영향 + 27 stacked PR 와 conflict.
- **trade-off**: FE 가 임시 두 shape 처리. envelope 마이그레이션 PR 시 한 번에 정리.

### B. Clock bean (TimeConfig)

- **선택 이유**: 단위 테스트 시 `Clock.fixed(...)` swap 가능 → 시간 의존 테스트 deterministic.
- **기각된 대안**: `Instant.now()` 직접 호출 (테스트 시 freeze 어려움).

### C. @Order(LOWEST_PRECEDENCE) — 도메인 advice 가 먼저

- **선택 이유**: 기존 도메인 별 @RestControllerAdvice (B7E RecordExceptionHandler 등) 가 먼저 잡고, 남는 것 GlobalExceptionHandler 가 fallback.
- **future**: 도메인 advice 도 ApiError 사용하도록 마이그레이션 (envelope migration PR 에서).

### D. RequestIdFilter — client 헤더 우선 + 없으면 생성

- **선택 이유**: 분산 trace (frontend / gateway 가 보낸 ID) 호환.
- **format**: `req-<uuid8>` — 짧고 인식 가능.

### E. ErrorCode COM-* prefix 만 (도메인 별 enum 미작성)

- **선택 이유**: foundation 만 — 도메인 별 ErrorCode (ACC-*, PAY-* 등) 는 각 도메인 마이그레이션 시 추가.
- **future**: 도메인 별 ErrorCode enum 패키지 (예: `com.bkurs.auth.error.AuthErrorCode`).

## 관련

- [[../../../20-areas/backend/spring/java-spring/api-design/common-response-envelope]] — 정책 SSoT
- [[../bkurs-be]] — 진행 로그
- 후속: ResponseEnvelopeMigration PR (153 endpoint 마이그레이션)
