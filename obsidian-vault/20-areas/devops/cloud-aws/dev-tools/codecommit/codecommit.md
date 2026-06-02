---
title: "AWS CodeCommit"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:32:00+09:00
tags:
  - aws
  - dev-tools
  - git
  - codecommit
---

# AWS CodeCommit

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CodeCommit hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

> **신규 가입 X (2024)** — AWS 가 신규 customer 에게 CodeCommit 신청을 막음. 기존 사용자만.

---

## 1. 한 줄

매니지드 Git 저장소. **신규 가입 닫힘** — AWS 가 GitHub / GitLab 위주로 가는 방향. 기존 사용자만 운영.

## 2. 왜 / 언제

- 이미 CodeCommit 운영 중 — 그대로 사용
- IAM 통합 (access key 없이 IAM 자격으로 git push)
- 신규 = **GitHub** + OIDC + IAM role 권장

## 3. 함정 / 대안

- 신규 = GitHub / GitLab / Bitbucket. CodeCommit 신규 가입 불가
- 기존 → GitHub 이전: `git clone --mirror` + push

## 4. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../codepipeline/codepipeline]] — CodeCommit 의 후속 단계
