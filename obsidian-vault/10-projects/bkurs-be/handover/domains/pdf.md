---
title: "도메인 — pdf (PDF Generator Gateway)"
kind: handover
status: stable
created_at: 2026-05-22T00:04:00+09:00
tags: [bkurs-be, handover, domain, gateway, pdf]
related:
  - "[[../../bkurs-be]]"
  - "[[../external-gateways]]"
  - "[[record]]"
home_hub: "[[../../bkurs-be]]"
---

# pdf 도메인

> PDF 생성 gateway. 학생부 (record) PDF 출력 3 endpoint (학생 본인 / 강사 / 학원장). Mock 은
> 단순 text → byte[], 실 구현은 후속 (Puppeteer / AWS Lambda chromium / wkhtmltopdf).

## 1. 책임

- `PdfGenerator.generate(html) → byte[]` 또는 `generateFromTemplate(tplKey, vars)`
- 학생부 PDF endpoint 3개 — role 별 권한 분기
- 실 SaaS 는 후속 (사이클 2 이후)

## 2. 핵심 타입

| 타입 | 핵심 |
| --- | --- |
| `PdfGenerator` (interface) | `generate(html) / generateFromTemplate(tplKey, vars)` |
| `MockPdfGenerator` | `@Profile("!prod")` — text → byte[] fake |

## 3. 패키지 구성

```
pdf/
├── controller/
│   └── StudentRecordPdfController   # /api/v1 — 학생부 PDF 3 endpoint
├── gateway/
│   ├── PdfGenerator                 # interface
│   └── MockPdfGenerator             # fake
└── handler/                         # PdfExceptionHandler
```

## 4. API endpoint

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| GET | `/api/v1/students/me/record.pdf` | STUDENT_HIGH | 본인 학생부 PDF |
| GET | `/api/v1/teachers/me/students/{studentId}/record.pdf` | TEACHER | 본인 수업 학생 |
| GET | `/api/v1/academies/me/students/{studentId}/record.pdf` | OWNER | 학원 내 학생 |

## 5. 핵심 코드 흐름

```
StudentRecordPdfController.studentMe(@AuthenticationPrincipal Long sid)
  ▼
1. records = studentRecordService.listForStudent(sid)
2. html = renderHtml(records)                       // 템플릿
3. byte[] pdf = pdfGenerator.generate(html)
4. return ResponseEntity.ok()
     .contentType(APPLICATION_PDF)
     .body(pdf)
```

응답은 `Content-Type: application/pdf` + raw bytes — envelope 안 함 (binary).

## 6. 의존

- **의존**: record, common
- **의존받음**: FE (브라우저 download)

## cross-link

- [[../external-gateways#pdf]]
- [[record]] — content source
- [[file]] — 파일 저장 가능 (현재는 download 만)
