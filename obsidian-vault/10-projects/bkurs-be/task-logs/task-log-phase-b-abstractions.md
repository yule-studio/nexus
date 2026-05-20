---
title: "Phase B 외부 서비스 abstraction 5종 task-log (B17~B21)"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T23:10:00+09:00
agent: engineering-agent/coding
tags: [task-log, phase-b, abstractions, gateway]
related:
  - "[[../bkurs-be]]"
  - "[[../architecture-hybrid-external-services]]"
  - "[[../external-services-accounts]]"
  - "[[task-log-b17-file-storage]]"
home_hub: bkurs-be
---

# Phase B 외부 서비스 abstraction 5종

## 결과 (5 PR)

| PR | 이슈 | 추상화 | 통합 | 회귀 |
|---|---|---|---|---|
| #68 | #67 | **FileStorageService** (S3 presigned URL) | POST /api/v1/files/upload-url | 10 |
| #70 | #69 | **NotificationService** (EMAIL/SMS/PUSH) | TeacherInviteService.issueAndNotify | 6 |
| #72 | #71 | **AiAssistant** (Claude) | Mock canned 5 type | 5 |
| #74 | #73 | **SttService** (Whisper/Clova) | Mock 1분 transcript | 5 |
| #76 | #75 | **PdfGenerator** (OpenPDF) | StudentRecord PDF 3 endpoint | 5 |

## 공통 패턴

모든 abstraction:
1. `gateway/` 패키지 — interface + record DTOs
2. `MockXService` — `@Component @ConditionalOnProperty(impl=mock, matchIfMissing=true)`
3. **env var swap** — `bkurs.external.{name}.impl={mock|real}` (B22~B27 후속 real plug-in)
4. test fixture 사용 (Mock 의 in-memory queue 검증)
5. ApiEnvelope 적용 가능한 endpoint (B17/B21) — envelope wrapping

## 패키지 구조

```
com.bkurs.{file,notification,ai,stt,pdf}/
├── gateway/
│   ├── XService.java          (interface)
│   ├── MockXService.java      (@ConditionalOnProperty)
│   └── XRequest/Result.java   (record)
├── domain/                    (enum — file 만)
├── dto/                       (request/view — file 만)
├── exception/                 (file 만)
└── web/                       (file / pdf 만)
```

## env var 매핑 (real plug-in 시)

| Service | env | real provider | 후속 PR |
|---|---|---|---|
| FileStorage | `bkurs.external.file-storage.impl=s3` + `AWS_*` + `S3_BUCKET_NAME` | AWS S3 | B26 |
| Notification | `bkurs.external.notification.impl=ses` + `EMAIL_FROM` | AWS SES | B23 |
| AiAssistant | `bkurs.external.ai.impl=anthropic` + `ANTHROPIC_API_KEY` + `ANTHROPIC_MODEL` | Claude | B24 |
| SttService | `bkurs.external.stt.impl=clova` + `NCP_CLOVA_*` | NCP Clova Speech | B25 |
| PdfGenerator | `bkurs.external.pdf.impl=openpdf` (lib 만) | OpenPDF + NanumGothic | B27 |

## ApiEnvelope 적용 케이스

- **B17 FileStorage** — 첫 envelope endpoint (success/data/error/meta + COM-* 에러 코드)
- **B21 PdfGenerator** — envelope 제외 (PDF binary stream)
- B18/B19/B20 — endpoint 신규 없음 (서비스만 추가)

## 회귀 영향

- **B17** — `StudentProfileApiTest.invalidGrade_400` envelope shape 으로 1곳 마이그레이션
- **B17** — `GlobalValidationExceptionHandler` envelope 마이그레이션 (raw→envelope)
- **B17** — `GlobalExceptionHandler.Exception.class` fallback 제거 (도메인 advice raw shape 보존)

## 누적 통계 (Phase B 완료 시점)

- 회귀: ~290+ GREEN (Phase B 신규 31 unit/integration)
- 신규 endpoint: 4 (file upload-url + pdf 3종)
- 신규 인터페이스 + Mock 클래스: 10 (5 service + 5 mock)
- 신규 DTO/enum/record: ~20

## 후속 (Phase C / production)

### Phase C — Merge cascade (다음 작업)

28 stacked PR (#4~#76) 을 dev 로 fast-forward 머지. 사용자 동의 후 진행.

### Production plug-in (B22~B29)

- B22 PortOne PaymentGateway
- B23 SendGrid 또는 AWS SES NotificationService
- B24 ClaudeAiAssistant
- B25 ClovaSttService 또는 WhisperSttService
- B26 S3FileStorage
- B27 OpenPdfGenerator + NanumGothic TTF bundle
- B28 OAuth (카카오 + 구글)
- B29 Sentry / Datadog Monitoring

## 관련

- [[../architecture-hybrid-external-services]] — 각 서비스 흐름 sequence diagram
- [[../external-services-accounts]] — 필요 SaaS 계정 + API key + 비용
- [[../bkurs-fe-vs-bkurs-be-gap]] — FE placeholder 항목 매핑
- [[task-log-b17-file-storage]] — B17 상세 결정
- 후속: Phase C merge cascade task-log
