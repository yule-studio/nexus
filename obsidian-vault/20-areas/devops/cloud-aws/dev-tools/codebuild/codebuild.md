---
title: "AWS CodeBuild"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:30:00+09:00
tags:
  - aws
  - dev-tools
  - cicd
  - build
---

# AWS CodeBuild

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CodeBuild hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **CI 빌드** 서비스. 컨테이너 기반 환경에서 `buildspec.yml` 대로 빌드 / 테스트 / 패키징. GitHub Actions / Jenkins 대안.

## 2. 왜 / 언제

- AWS 안에서 CI 빌드 (VPC private subnet 안에서 빌드 등)
- 빌드 결과를 ECR / S3 / CodeArtifact 에 직접 push
- IAM role 로 안전한 자격 (GitHub secret 없이)
- Docker image 빌드 (privileged 모드)
- GitHub / GitLab / Bitbucket webhook 연동

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Project** | 빌드 정의 (source / env / buildspec / artifact) |
| **buildspec.yml** | 빌드 단계 정의 (install / pre_build / build / post_build) |
| **Compute** | small / medium / large / xlarge / 2xlarge (vCPU/mem) |
| **Environment** | 빌드 컨테이너 이미지 (AWS managed / 자체 ECR) |
| **Source** | GitHub / CodeCommit / S3 / GitHub Enterprise |
| **Artifacts** | 빌드 결과 → S3 |
| **Reports** | 테스트 / coverage 결과 |

## 4. 최소 buildspec.yml

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      python: 3.12
  pre_build:
    commands:
      - pip install -r requirements.txt
  build:
    commands:
      - pytest --junitxml=results.xml
      - docker build -t $REPO_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION .
  post_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR
      - docker push $REPO_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
reports:
  pytest:
    files: [results.xml]
    file-format: JUNITXML
artifacts:
  files: [appspec.yml]
```

## 5. 비용 / 함정

- 분 단위 과금. Linux small ~$0.005/분, large ~$0.02/분
- **Docker build 는 privileged mode** 필요 (project 설정)
- **VPC 빌드** — 사설 자원 (RDS / ECR private) 접근 시. 단 시작 latency ↑
- **cache** — Docker layer / S3 / local cache 활용으로 빌드 시간 단축
- **GitHub Actions 비교** — AWS 안 통신 / VPC 필요 / IAM role 친화 = CodeBuild. 그 외엔 GHA 가 보통 더 빠른 UX.

## 6. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../codedeploy/codedeploy]]
- [[../codepipeline/codepipeline]]
- [[../../compute/ecr/ecr]]
