---
title: "B7D — MockInterview + Response 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b7-content
created_at: 2026-05-20T14:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-mock-interview-model]]"
  - "[[task-log-b7c-question]]"
tags: [task-log, mock-interview, b7d]
---

# 작업

GitHub issue [#39](https://github.com/seum-running/bkurs-be/issues/39) — MockInterview + Response.

# 결정 요약

| 결정 | 내용 |
|---|---|
| 모델 | mock_interview + interview_response 별도 |
| type | PERSONAL/MOTIVATION/MAJOR/FREE |
| 미디어 | audio_url + video_url (외부 스토리지, DB 는 URL 만) |
| 평가 | 강사 본인 배정, score 0-100 + audit |
| upsert | (interview, account) UNIQUE |

# 진행

## 2026-05-20
- C1: entity + V17 + base 보강 (2 repo + 2 cleanup)
- C2: 2 service + 3 예외
- C3: 6 DTO + 2 controller + ExceptionHandler
- C4: 10 회귀 + CHANGELOG
- C5: CHANGELOG fixup
- PR [#40](https://github.com/seum-running/bkurs-be/pull/40) — **172/172 GREEN**

# Handover

| 구현 | 선택 이유 |
|---|---|
| 미디어 URL 만 (BLOB X) | 파일 시스템 분리 표준. B8+ 파일 업로드 인프라 도입 시 S3 등으로 |
| 4 type enum | bkurs-fe 학종 면접 카테고리 매칭 |
| expectedMinutes nullable | 권장만 — 강제 X (학생이 짧거나 길게 답해도 OK) |
| evaluation_score 0-100 | 표준 평가 척도 — entity invariant 강제 |
| DTO 이름 InterviewResponse2 / ResponseView | entity InterviewResponse 와 이름 충돌 회피 |

# 다음

- B7E 생기부 (StudentRecord)
- B8 파일 업로드 인프라 (S3 / 로컬 / cloud)

# 참고

- [[../decisions/decision-mock-interview-model]]
- [[task-log-b7c-question]] — 유사 패턴 (type 통합)
