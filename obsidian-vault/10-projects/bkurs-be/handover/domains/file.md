---
title: "도메인 — file (Storage Gateway / Presigned URL)"
kind: handover
status: stable
created_at: 2026-05-22T00:03:00+09:00
tags: [bkurs-be, handover, domain, gateway, file]
related:
  - "[[../../bkurs-be]]"
  - "[[../external-gateways]]"
home_hub: "[[../../bkurs-be]]"
---

# file 도메인

> 파일 저장소 추상화. **서버는 파일을 거치지 않고 presigned URL 만 발급** — 클라가 직접 S3 /
> Storage 에 PUT. interview audio/video / pdf 첨부 / record 첨부 등이 의존.

## 1. 책임

- presigned upload URL 발급 (TTL + 카테고리별)
- 구현: Mock (local FS / in-memory) ↔ 실 SaaS (사이클 2 후속)

## 2. 핵심 타입

| 타입 | 핵심 |
| --- | --- |
| `FileStorageService` (interface) | `issuePresignedUploadUrl(category, ttl) → UploadUrlIssuance` |
| `UploadUrlIssuance` | `{ url, fields, expiresAt }` — 클라가 PUT 에 사용 |
| `FileCategory` (enum) | INTERVIEW_AUDIO / RECORD_ATTACHMENT / 등 |
| `MockFileStorage` | `@Profile("!prod")` — fake URL |
| `PresignedUrlRequest` / `PresignedUrlView` | DTO |
| `FileStorageException` | gateway 예외 |

## 3. 패키지 구성

```
file/
├── controller/
│   └── FileStorageController        # /api/v1/files — presigned URL 발급
├── gateway/
│   ├── FileStorageService           # interface
│   ├── MockFileStorage              # local 구현
│   └── UploadUrlIssuance            # result record
├── domain/                          # FileCategory enum
├── dto/{request,response}/          # PresignedUrlRequest / View
├── exception/                       # FileStorageException
└── handler/                         # FileExceptionHandler
```

## 4. API endpoint

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/files/upload-url` | category + ttl → presigned URL 발급 |

## 5. 핵심 코드 흐름

```
FileStorageController.issue(PresignedUrlRequest req)
  ▼
UploadUrlIssuance issuance = fileStorageService.issuePresignedUploadUrl(
    req.category(), Duration.ofSeconds(req.ttlSeconds()));
return envelopes.ok(PresignedUrlView.from(issuance), http);
```

Mock 구현은 `https://mock.local/upload/{uuid}` 형태 URL 반환 + DB 어디에도 저장 안 함. 실 구현 후속:

| 후속 | 구현 |
| --- | --- |
| AWS S3 | `S3FileStorage` — AWS SDK `S3Presigner` |
| Supabase Storage | `SupabaseFileStorage` — REST adapter |
| GCS | `GcsFileStorage` |

## 6. 의존

- **의존**: common.api
- **의존받음**: interview (audio/video URL), pdf (생성된 PDF 저장), record (첨부)

## cross-link

- [[../external-gateways#file]]
- [[interview]]
- [[pdf]]
