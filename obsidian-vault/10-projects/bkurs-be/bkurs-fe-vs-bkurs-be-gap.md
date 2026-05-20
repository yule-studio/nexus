---
title: "bkurs-fe ↔ bkurs-be 갭 — 외부 서비스 + 도메인"
kind: gap-analysis
project: bkurs-be
status: current
created_at: 2026-05-20T22:35:00+09:00
agent: engineering-agent/tech-lead
tags: [gap, fe-be, external-services, bkurs]
related:
  - "[[bkurs-be]]"
  - "[[../bkurs-fe/bkurs-fe]]"
  - "[[external-services-accounts]]"
home_hub: bkurs-be
---

# bkurs-fe ↔ bkurs-be 갭 분석

> bkurs-fe (React) 가 가정/구현 중이지만 bkurs-be 에 없는 기능 / 외부 연동.
> 두 부류:
> - **Gap A**: 외부 서비스 연동 미구현 (BE 추상화 자체 없음)
> - **Gap B**: 도메인 자체가 BE 에 없음

## 1. Gap A — 외부 서비스 연동 (BE 추상화 필요)

bkurs-fe 가 mock 또는 placeholder 로 두고 "S3 단계에서 ...", "AI 분석 ..." 등으로 표시한 항목들. BE 에 추상화 추가 필요.

| # | FE 위치 | FE 상태 | BE 필요 추상화 | 외부 서비스 |
|---|---|---|---|---|
| 1 | `master/_pages/lessons/useUploadFlow.ts` | `simulateUpload` mock | `FileStorageService` + presigned URL endpoint | S3 (강의 영상) |
| 2 | `middle-student/_pages/homework/Homework.tsx` | "mock: 녹음 시작 — S3 MediaRecorder" | `FileStorageService` + audio presigned | S3 (음성 녹음) |
| 3 | `middle-student/_pages/simulation/Simulation.tsx` | "mock: 녹화 시작 — S3 MediaRecorder" | `FileStorageService` + video presigned | S3 (영상 녹화) |
| 4 | `admin/_pages/students/detail/high-tabs/RecordTab.tsx` | "supabase.storage signed URL" | `FileStorageService` + signed download URL | S3 (생기부 PDF) |
| 5 | `admin/_pages/students/detail/high-tabs/SimulationTab.tsx` | "supabase.storage signed URL" | `FileStorageService` + signed download | S3 (영상 재생) |
| 6 | `admin/_layout/AdminSidebar.tsx` | "supabase.storage.from('academy-logos') 업로드" | `FileStorageService` + 이미지 upload | S3 (학원 로고) |
| 7 | `admin/_pages/teachers/AddTeacherModal.tsx` | "초대 메일이 발송됩니다 (mock 단계 — 실 발송은 S3)" | `NotificationService` (이메일) | SES / SendGrid |
| 8 | `master/_pages/academies/AcademyAddModal.tsx` | "원장님 이메일로 자동 발송" | `NotificationService` (이메일 — 환영) | SES / SendGrid |
| 9 | `master/_pages/billing/BillingUnpaidAlert.tsx` | "독촉 알림 발송 (Mock)" | `NotificationService` (이메일 + SMS) | SES + NCP SENS |
| 10 | `master/_pages/notices/NoticeComposeModal.tsx` | "✉️ 이메일 발송 / 🔔 푸시 알림 발송" toggle | `NotificationService` (multi-channel) | SES + FCM |
| 11 | `master/_pages/academies/detail/tabs/AcademyPaymentsTab.tsx` | "미납 독촉" | `NotificationService` | SMS / 이메일 |
| 12 | `middle-student/_pages/suhaeng/Suhaeng.tsx` | "AI 분석 대기 중" (수행평가) | `AiAssistant` + 자동 분석 endpoint | Claude API |
| 13 | `middle-student/_pages/simulation/Simulation.tsx` | "AI 분석 요청" (모의면접) | `AiAssistant` + 면접 평가 | Claude API |
| 14 | `middle-student/_pages/expect/EssayWizard.tsx` | "AI 가 자동 합성한 자소서" | `AiAssistant` + 자소서 합성 endpoint | Claude API |
| 15 | `middle-student/_pages/homework/Homework.tsx` | 음성 → 텍스트 전사 (mock) | `SttService` + async 전사 | Whisper / Clova |
| 16 | `admin/_pages/students/detail/high-tabs/expect/PdfPanel.tsx` | "pdfjs-dist + signed URL" | `PdfGenerator` + 생기부 PDF endpoint | OpenPDF (lib) + S3 storage |
| 17 | `admin/_pages/login/useAdminLogin.ts` | "supabase.auth.signInWithPassword" | BE Auth (이미 자체 JWT) — Supabase 의존성 제거 | (BE 자체) |
| 18 | `master/_pages/login/useMasterLogin.ts` | "supabase.auth.signInWithPassword" | BE Auth + MASTER role | (BE 자체) |
| 19 | `master/_pages/lessons/useUploadFlow.ts` | "supabase.storage 'lessons' upload" | `FileStorageService` + Lesson entity | S3 + BE 신규 도메인 |
| 20 | `pages/high-student/_pages/pending/Pending.tsx` | "polling 또는 서버 push" | WebSocket 또는 SSE | (BE 자체 — 후속) |

**요약**: 외부 서비스 연동 갭 **5종** + **인터페이스 5개** 필요:
- `FileStorageService` (S3) — 항목 1-6, 19 (7곳)
- `NotificationService` (이메일/SMS/Push) — 항목 7-11 (5곳)
- `AiAssistant` (Claude) — 항목 12-14 (3곳)
- `SttService` (Whisper/Clova) — 항목 15 (1곳)
- `PdfGenerator` (OpenPDF + S3) — 항목 16 (1곳)

## 2. Gap B — BE 도메인 자체가 없음

bkurs-fe 가 화면을 만들었지만 BE 에 해당 도메인이 없는 항목들.

### 2.1 Master (시스템 관리자) 도메인

bkurs-fe 의 `src/pages/master/` 전체 영역 — bkurs-be 에 거의 없음:

| FE 경로 | 기능 | BE 상태 |
|---|---|---|
| `master/_pages/login/` | MASTER 로그인 | `Account.role=MASTER` enum 만 있고 endpoint 없음 |
| `master/_pages/academies/` | 학원 추가/삭제/조회 (master 권한) | OWNER 본인 학원만 관리. master 전용 endpoint 없음 |
| `master/_pages/staff/` | 마스터 admin 그룹 관리 | 없음 — Account.role MASTER 만 |
| `master/_pages/notices/` | 전체 학원 공지 발송 | Announcement 는 학원 단위. master 전체 공지 없음 |
| `master/_pages/billing/` | 전체 결제 내역 + 미납 + 차트 | `/api/v1/admin/payments` 만 있음 (B4C). 미납/차트 없음 |
| `master/_pages/audit/` | 감사 로그 조회 + Excel export | `AuditLog` entity 자체 없음 |
| `master/_pages/data/` | 학원 별 통계 비교 | `AcademyDashboard` (B16) 만, master 전체 view 없음 |
| `master/_pages/lessons/` | 강의 영상 업로드/관리 | `Lesson` entity 없음 (`Course` 와 다름) |

**추가 필요 도메인**:
- `MasterAuthController` — MASTER 로그인 + 권한 검증
- `MasterAcademyController` — 학원 CRUD (master)
- `MasterStaffController` — admin 그룹 관리
- `MasterNoticeController` — 전체 공지 broadcast
- `MasterBillingController` — billing aggregation + 미납 + 차트
- `MasterAuditController` + `AuditLog` entity + Audit interceptor (모든 admin action log)
- `MasterDataController` — 학원 cross-stats
- `LessonController` + `Lesson` entity + 영상 storage 연동

### 2.2 학생 워크플로 분기 (Middle / High)

bkurs-fe 가 학년 별 (중등/고등) 다른 UI 흐름:

| FE 경로 | 기능 | BE 상태 |
|---|---|---|
| `middle-student/_pages/suhaeng/Suhaeng.tsx` | 수행평가 4단계 (학생 답안 → AI 분석 → 1차 피드백 → 재제출 → 최종) | `PerformanceEvaluation` (B7H) 는 1 round (제출 → 채점). 4단계 workflow 없음 |
| `middle-student/_pages/expect/EssayWizard.tsx` | 자소서 wizard (단계 입력 + AI 합성 + 미리보기) | `PersonalStatement` (B7B) 는 단순 DRAFT/SUBMITTED. wizard endpoint 없음 |
| `high-student/_pages/past/` | 기출 문제 | `Question` (B7C) `type=PAST` 가능. 대학별 filter 등 후속 |
| `high-student/_pages/expect/` | 예상질문 | `Question.type=EXPECTED` |
| `high-student/_pages/saenggibu/` | 생기부 | `StudentRecord` (B7E) ✅ |
| `high-student/_pages/pending/` | 대기 (가입 승인 대기) | `Membership.PENDING_APPROVE` (B2C) ✅ |

**추가 필요**:
- `PerformanceEvaluation` 에 단계 상태 (`StudentSubmitted` → `AiAnalyzed` → `TeacherFeedback` → `Resubmitted` → `Finalized`) — state machine
- `PersonalStatement` wizard endpoint (steps 별 partial save + AI 합성)

### 2.3 알림 entity

FE 가 학원장/학생/강사 별 알림 (notification feed) 가정:

- BE: 이메일/SMS 발송 (Gap A) 외에 in-app notification feed 없음
- 추가 필요: `Notification` entity (read/unread + actor + 도메인 link) + endpoint

### 2.4 실시간 (WebSocket / SSE)

- FE: `pending/Pending.tsx` 가 "polling 또는 서버 push" 가정
- BE: 없음
- 추가 필요: WebSocket gateway (Spring `@MessageMapping`) 또는 SSE `EventSource`

## 3. 우선순위 분류

### Phase 1 (즉시 — 외부 연결 직전 상태 마감)

**Gap A 의 5 추상화 추가 (이번 사이클 작업)**:
- B17 `FileStorageService` (interface + Mock + presigned URL endpoint)
- B18 `NotificationService` (interface + Mock + TeacherInvite 통합)
- B19 `AiAssistant` (interface + Mock + 자소서/수행평가/면접/채점 통합)
- B20 `SttService` (interface + Mock + audio_url 자동 전사 trigger)
- B21 `PdfGenerator` (interface + Mock + 생기부 PDF endpoint)

### Phase 2 (정식 출시 전 필요)

**Gap B 의 Master 도메인 추가**:
- B30 MasterAuth + Account.role MASTER endpoint
- B31 MasterAcademy CRUD
- B32 AuditLog entity + interceptor + endpoint
- B33 MasterBilling aggregation
- B34 MasterNotice broadcast
- B35 Lesson entity + 영상 storage 연동
- B36 MasterStaff
- B37 MasterData (학원 cross-stats)

**Gap B 의 학생 워크플로 보강**:
- B38 PerformanceEvaluation 4 단계 state machine
- B39 PersonalStatement wizard endpoint + AI 합성 통합
- B40 Question 대학/과목 filter 보강

### Phase 3 (후속)

- B41 Notification entity (in-app feed)
- B42 WebSocket / SSE realtime
- B43 Supabase 의존성 완전 제거 가이드 (FE 측 작업)

## 4. Supabase 의존성 제거 (FE 측)

bkurs-fe 가 현재 `@supabase/supabase-js` 의존:
- `src/lib/supabase.ts` — client 생성
- 13 곳에서 import (auth / storage / db 가정)

**전환 후**:
- Auth → bkurs-be `/api/v1/auth/*` 호출
- Storage → bkurs-be `/api/v1/files/upload-url` (B17) + S3 직접 업로드
- DB → bkurs-be 도메인 endpoint

→ Supabase 패키지 / env 제거 가능 (별도 FE PR).

## 관련

- [[external-services-accounts]] — 필요 SaaS 계정 + API key + 비용
- [[architecture-hybrid-external-services]] — Hybrid 아키텍처
- [[bkurs-be]] — 진행 로그
- [[../bkurs-fe/bkurs-fe]] — FE 쪽 hub
