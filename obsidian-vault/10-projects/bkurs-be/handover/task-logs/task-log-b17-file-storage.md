---
title: "B17 task-log — FileStorageService"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T22:50:00+09:00
agent: engineering-agent/coding
tags: [task-log, file-storage, abstraction, b17, phase-b]
related:
  - "[[../bkurs-be]]"
  - "[[../hybrid-external-services]]"
  - "[[../services-accounts]]"
home_hub: bkurs-be
---

# B17 task-log — FileStorageService 추상화

## 결과

- 이슈: #67
- PR: #68 (draft, stacked on #66 envelope foundation)
- 브랜치: `feature/issue-67-b17-file-storage`
- 1 commit (foundation + 마이그레이션)
- 10 integration + 8 unit (envelope) + 회귀 영향 1곳 envelope 마이그레이션

## 핵심 결정

### A. presigned URL 패턴 (BE = 발급, Client = 직접 업로드)

- file 통과 시 BE bandwidth 2배 → presigned 로 bandwidth 절약 + secret 보호.
- 10분 만료, 1회용 → credential leak 위험 최소.
- vault `architecture-hybrid-external-services` §3 sequence diagram 참고.

### B. FileCategory enum + content-type 검증

- 6 카테고리 (INTERVIEW_VIDEO/AUDIO, HOMEWORK_AUDIO, ACADEMY_LOGO, LESSON_VIDEO, STUDENT_RECORD_PDF)
- 각 카테고리 별 S3 prefix + 허용 content-type prefix + maxSizeBytes (5MB~1GB)
- enum 확장 만 으로 새 파일 종류 추가

### C. @ConditionalOnProperty mock/real swap

- `bkurs.external.file-storage.impl=mock` (default) — MockFileStorage 활성
- `bkurs.external.file-storage.impl=s3` (후속) — S3FileStorage 활성, mock 자동 비활성
- 환경별 application yml 1줄 swap

### D. ApiEnvelope wrapping (envelope foundation 첫 적용)

- 신규 endpoint 부터 envelope 강제 정책 따름
- BusinessException → handleBusiness → envelope.fail
- Bean Validation → GlobalValidationExceptionHandler envelope.fail (마이그레이션됨)

### E. Exception.class fallback 제거 (회귀 보호)

- 기존 도메인 advice (StudentProfileExceptionHandler 등) 가 RuntimeException 을 raw shape 으로 처리 중.
- fallback 두면 envelope wrapping 으로 회귀 깨짐 → 제거.
- 153 endpoint envelope migration PR 시 도메인 advice 마이그레이션 + 본 fallback 재추가 검토.

## 파일 (11개)

```
src/main/java/com/bkurs/file/
├── domain/FileCategory.java                    (enum 6종 + 제약)
├── gateway/
│   ├── FileStorageService.java                 (interface)
│   ├── UploadUrlIssuance.java                  (record)
│   └── MockFileStorage.java                    (@Component + @ConditionalOnProperty)
├── exception/FileStorageException.java         (InvalidInput 상속)
├── dto/PresignedUrlRequest.java                (record + @Valid)
├── dto/PresignedUrlView.java                   (record + from(issuance))
└── web/FileStorageController.java              (POST upload-url, envelope)

수정:
├── common/handler/GlobalExceptionHandler.java  (Exception.class 제거)
├── common/web/GlobalValidationExceptionHandler.java (envelope 마이그레이션)
└── test/.../StudentProfileApiTest.java         (envelope shape assert 수정)

신규 테스트:
└── test/.../file/FileStorageApiTest.java       (10 integration)
```

## 10 회귀 (FileStorageApiTest)

1. INTERVIEW_VIDEO presigned URL → envelope ok shape ✅
2. HOMEWORK_AUDIO ✅
3. ACADEMY_LOGO ✅
4. STUDENT_RECORD_PDF ✅
5. LESSON_VIDEO ✅
6. content-type mismatch (audio category + video) → 400 envelope error COM-4001 ✅
7. category 누락 → Bean Validation 400 envelope fieldErrors ✅
8. contentType blank → Bean Validation 400 ✅
9. X-Request-Id echo — client header 그대로 응답 ✅
10. 인증 없음 → 401 envelope error ✅

## 후속

- B18 NotificationService (이메일 / SMS + TeacherInvite 자동 발송)
- B19 AiAssistant (Claude + 자소서/PE/모의면접)
- B20 SttService (Whisper/Clova + audio_url 자동 전사)
- B21 PdfGenerator (생기부 PDF)
- (별도) B26 S3FileStorage — AWS SDK plug-in (impl=s3 swap)

## 관련

- [[../hybrid-external-services]] §3 — file upload sequence
- [[../services-accounts]] §0 (Supabase → Spring 매핑) + §1.1 (AWS)
- [[../../../20-areas/backend/spring/java-spring/api-design/common-response-envelope]]
