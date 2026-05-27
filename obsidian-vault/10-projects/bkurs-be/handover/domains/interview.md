---
title: "도메인 — interview (모의면접 / 미디어 응답)"
kind: handover
status: stable
created_at: 2026-05-21T23:50:00+09:00
tags: [bkurs-be, handover, domain, interview]
related:
  - "[[../../bkurs-be]]"
  - "[[file]]"
  - "[[stt]]"
  - "[[../decisions/decision-mock-interview-model]]"
home_hub: "[[../../bkurs-be]]"
---

# interview 도메인

> 강사가 모의면접 (4 type) 발행, 학생이 텍스트 + audio / video URL 로 응답. 강사가 0~100 평가.
> audio URL 은 [[file]] gateway 의 presigned upload URL, transcript 는 [[stt]] gateway 후속 연동.

## 1. 책임

- 강사 MockInterview CRUD (`InterviewType` 4종)
- 학생 응답 — uk upsert + 텍스트 / audioUrl / videoUrl
- 강사 평가 (0~100 + 코멘트)

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `MockInterview` | id / courseId / type / title / questionsBody / expectedMinutes |
| `InterviewResponse` | uk(interview_id, student_account_id) + textAnswer / audioUrl (nullable) / videoUrl (nullable) / evaluation 0~100 (nullable) / submittedAt / evaluatedAt |
| `InterviewType` | enum 4종 (예: BASIC / DEPTH / GROUP / PRESENTATION) |

## 3. 패키지 구성

```
interview/
├── controller/
│   ├── TeacherInterviewController  # /api/v1/teachers/me — interview + 평가
│   └── StudentInterviewController  # /api/v1/students/me — 응답
├── service/                        # InterviewService / ResponseService
├── domain/
├── dto/{request,response}/
├── enums/                          # InterviewErrorCode (ITV-)
├── exception/
└── handler/
```

## 4. API endpoint

### TEACHER (6)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/teachers/me/courses/{courseId}/interviews` | 발행 |
| GET | `/api/v1/teachers/me/courses/{courseId}/interviews` | 목록 |
| PATCH/DELETE | `/api/v1/teachers/me/interviews/{id}` | |
| GET | `/api/v1/teachers/me/interviews/{id}/responses` | 응답 목록 |
| POST | `/api/v1/teachers/me/responses/{id}/evaluate` | 평가 |

### STUDENT (3)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/interviews` | 내 수업 면접 |
| POST | `/api/v1/students/me/interviews/{id}/respond` | 응답 (uk upsert) |
| GET | `/api/v1/students/me/interview-responses` | 내 응답 목록 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V17 | `mock_interview` (type enum + expected_minutes) + `interview_response` (uk + 미디어 URL + 평가) |

## 6. 미디어 흐름

```
학생: presigned upload URL 요청 (/api/v1/files/upload-url)  ─ FileStorageService.issuePresignedUploadUrl
   ▼
S3 / Storage 에 직접 PUT  (서버는 거치지 않음)
   ▼
받은 fileUrl 을 /interviews/{id}/respond 의 audioUrl/videoUrl 필드로 전달
   ▼
(후속) STT gateway 로 audio → transcript
```

자세히 [[file]], [[stt]].

## 7. 의존

- **의존**: course (FK), file (presigned URL), stt (transcript — 후속)
- **의존받음**: record (interview 평가 → SUBJECT 카테고리 자동 bridge 후보, 현재 없음)

## 서비스 로직 (`interview/service/`)

2 service. audio/video URL 응답 → 후속 STT transcript.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `InterviewService` | `@Transactional` | 강사 면접 CRUD (4 type) |
| `ResponseService` | `@Transactional` | 학생 응답 (text + URL) upsert + 강사 evaluate |

### 핵심 메서드

```java
// InterviewService
public MockInterview createByTeacher(Long teacherAccountId, Long courseId, InterviewType type, String title, String questionsBody, Integer expectedMinutes)
public MockInterview update(...)
public void delete(...)
public List<MockInterview> listByTeacherCourse(Long teacherAccountId, Long courseId, InterviewType typeFilter)

// ResponseService
public InterviewResponse submit(Long studentAccountId, Long interviewId, String textAnswer, String audioUrl, String videoUrl)
                                                          // uk(interview_id, student_id) upsert
                                                          // audioUrl/videoUrl 은 file gateway 의 presigned URL (서버는 파일 직접 거치지 않음)
public InterviewResponse evaluateByTeacher(Long teacherAccountId, Long responseId, Integer evaluation, String feedback)
                                                          // 강사 권한 + evaluation 0~100 + audit
public List<InterviewResponse> listByInterview(Long teacherAccountId, Long interviewId)
public List<InterviewResponse> listMine(Long studentAccountId)
```

후속 STT 연동 시점에 `ResponseService.attachTranscript(responseId)` 추가 예정 — audioUrl → [[stt|Clova]] async.

## cross-link

- [[course]]
- [[file]] / [[stt]]
- [[../decisions/decision-mock-interview-model]]
