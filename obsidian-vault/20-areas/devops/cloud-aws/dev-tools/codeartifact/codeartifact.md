---
title: "AWS CodeArtifact"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:33:00+09:00
tags:
  - aws
  - dev-tools
  - artifact
  - codeartifact
---

# AWS CodeArtifact

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CodeArtifact hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **패키지 저장소** — npm / pip / Maven / NuGet / Cargo / Generic. 사내 패키지 + 외부 의존성 캐시 + IAM 권한.

## 2. 왜 / 언제

- 사내 라이브러리 배포 (npm publish 사내용)
- 외부 npm / PyPI 캐시 (네트워크 안정 / 검수)
- npm Pro / Nexus / Artifactory 대안
- IAM 으로 패키지 접근 제어

## 3. 함정

- 패키지 저장 / 데이터 전송 비용
- pip / npm 의 인증 token 은 12 시간 — refresh 자동화 필요
- public CDN 보다 latency 변동 (region 선택)

## 4. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../codebuild/codebuild]]
- [[../../compute/ecr/ecr]]
