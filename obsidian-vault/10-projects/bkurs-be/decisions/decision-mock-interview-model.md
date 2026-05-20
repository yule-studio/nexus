---
title: "B7D 결정 — MockInterview + Response (텍스트/오디오/비디오 URL + 평가)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T14:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, mock-interview, b7d]
related:
  - "[[../bkurs-be]]"
  - "[[decision-question-attempt-model]]"
home_hub: bkurs-be
---

# B7D 결정 — MockInterview + Response

## 결론

1. **`mock_interview` + `interview_response` 별도 테이블** — interview type enum (PERSONAL/MOTIVATION/MAJOR/FREE).
2. **response 는 텍스트 + audio_url + video_url** — 미디어 본체는 외부 스토리지 (S3 등 B8+), DB 는 URL 만.
3. **(interview, account) UNIQUE → upsert** — 학생 재응답 = update.
4. **평가는 강사** — `evaluation_score` 0-100 + `evaluation` 텍스트 + `evaluated_by` audit.

## 왜 필요한가

- 모의면접 = 입시 핵심 콘텐츠 (특히 학종 / 자소서 면접).
- 학생이 음성/영상 응답 → 강사가 들어보고 평가 → 피드백 워크플로우.
- B8+ 파일 업로드 인프라 도입 전 URL 기반으로 시작 (학생이 외부 녹음/녹화 → URL 제출).

## 4 대안 비교

### A. interview/response 별도 테이블 + 미디어 URL — **채택**

- 장점: B7A/B7B/B7C 패턴 일관. 미디어는 외부 (S3) → DB 단순.
- 단점: URL 검증 필요 (운영자가 모니터).

### B. 파일 업로드 직접 처리 (DB BLOB)

- 단점: DB 비대 + 백업/복원 비용. 파일 시스템 분리가 표준.
- **기각**.

### C. interview 가 question 의 하위 type (Question.type 확장)

- 단점: Question.body 외에 audio/video URL 필요 — 컬럼 추가는 모든 type 영향.
- **기각**: 도메인 분리가 깔끔.

### D. response 다중 row (시도 이력)

- 미래 후보 (B8+).

## 테이블 설계

```sql
CREATE TABLE mock_interview (
  id BIGINT PK,
  course_id BIGINT FK CASCADE,
  type VARCHAR(32),       -- PERSONAL/MOTIVATION/MAJOR/FREE
  author_account_id BIGINT FK,
  title VARCHAR(300),
  question_text TEXT,
  expected_minutes INT,    -- 권장 답변 시간
  created_at, updated_at
);

CREATE TABLE interview_response (
  id BIGINT PK,
  interview_id BIGINT FK CASCADE,
  account_id BIGINT FK,
  content TEXT,            -- 텍스트 답변 (또는 transcript)
  audio_url VARCHAR(500),
  video_url VARCHAR(500),
  submitted_at,
  evaluation_score INT,    -- 0-100
  evaluation TEXT,
  evaluated_at,
  evaluated_by_account_id BIGINT,
  UNIQUE (interview_id, account_id),
  KEY idx (interview, evaluated_at), KEY idx (account)
);
```

## 회귀 라인 (10)

1. 강사 출제 (PERSONAL) → 201
2. STUDENT 출제 403
3. type 필터
4. 학생 응답 (텍스트만)
5. 학생 응답 (텍스트 + audioUrl + videoUrl)
6. 학생 재응답 → upsert
7. 강사 평가 (score 85 + 코멘트)
8. 다른 학원 403
9. DELETE → response CASCADE
10. 본인 응답 이력

## 관련

- [[decision-question-attempt-model]] — 동일 패턴 재사용
- 후속: B7E 생기부 / B8 파일 업로드 인프라
