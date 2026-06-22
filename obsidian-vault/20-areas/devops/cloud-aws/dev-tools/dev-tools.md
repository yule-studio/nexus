---
title: "AWS Developer Tools (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:30:00+09:00
tags:
  - aws
  - dev-tools
  - iac
  - cicd
  - hub
---

# AWS Developer Tools (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> 인프라를 코드로 / CI 빌드 / 배포 자동화 / 추적·디버그 도구.

---

## 1. 서비스

### 1.1 IaC

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **CloudFormation** | [[cloudformation]] | AWS 공식 IaC (JSON / YAML) — 모든 자원 선언 |
| **CDK** | [[cdk]] | 코드 (TS / Python / Java) 로 IaC → CloudFormation 컴파일 |
| **SAM** | [[sam/sam\|sam]] | serverless 전용 IaC — CloudFormation 짧은 문법 + 로컬 디버그 (`sam local invoke`) |

### 1.2 CI·CD

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **CodeBuild** | [[codebuild/codebuild\|codebuild]] | 매니지드 CI 빌드 — `buildspec.yml` 대로 빌드 / 테스트 / 패키징 (GitHub Actions / Jenkins 대안) |
| **CodeDeploy** | [[codedeploy/codedeploy\|codedeploy]] | EC2 / ECS / Lambda 에 Blue/Green / Canary 배포 자동화 + 자동 rollback |
| **CodePipeline** | [[codepipeline/codepipeline\|codepipeline]] | CI/CD 파이프라인 오케스트레이션 (Source → Build → Test → Deploy) — GHA workflow 의 AWS 버전 |

### 1.3 패키지·아티팩트·SCM

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **CodeArtifact** | [[codeartifact/codeartifact\|codeartifact]] | 매니지드 패키지 저장소 (npm / pip / Maven / NuGet / Cargo) — 사내 패키지 + 외부 의존성 캐시 |
| **CodeCommit** | [[codecommit/codecommit\|codecommit]] | 매니지드 Git 저장소 — **신규 가입 닫힘 (2024)**, 기존 사용자만 (AWS 가 GitHub / GitLab 위주로 전환) |

### 1.4 IDE·SDLC·AI

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Cloud9** | [[cloud9/cloud9\|cloud9]] | 브라우저 IDE (EC2 위 코드 편집 / 디버그) — **신규 가입 닫힘 (2024)**, CodeCatalyst dev environment 로 이동 |
| **CodeCatalyst** | [[codecatalyst/codecatalyst\|codecatalyst]] | AWS 통합 SDLC 포털 (이슈 / 리포 / 워크플로우 / dev environment) — GitHub + Jira + GHA + Codespaces 를 한 데 |
| **Q Developer** | [[q-developer/q-developer\|q-developer]] | LLM 기반 코드 자동완성 + 채팅 + IaC / 운영 Q&A — GitHub Copilot 의 AWS 버전 (옛 CodeWhisperer) |

### 1.5 관측

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **X-Ray** | [[x-ray/x-ray\|x-ray]] | 분산 추적 — 서비스 A → B → C 호출 흐름을 trace 로 시각화, 병목 / 실패 지점 파악 (OpenTelemetry 호환) |

---

## 2. IaC 선택

| 도구 | 형태 | 언제 |
| --- | --- | --- |
| CloudFormation | YAML 선언 | AWS-only, 외부 의존성 X, 작은 스택 |
| CDK | TS/Python 코드 | 분기 / 반복 / 추상화 필요한 큰 인프라 |
| Terraform | HCL | **멀티클라우드** 표준, 가장 큰 생태계 |
| Pulumi | TS/Python/Go | Terraform 대안, 진짜 프로그래밍 |
| SAM | YAML (Lambda 특화) | 순수 serverless |

→ 실무 표준: **CDK** (AWS-only) 또는 **Terraform** (멀티클라우드).

자세히 → [[../../iac/iac]]

---

## 3. CI/CD 선택

| 흐름 | 도구 |
| --- | --- |
| GitHub 코드 + OIDC + AWS 배포 | **GitHub Actions + IAM Role (권장)** |
| AWS 안 완결 | CodeCommit + CodeBuild + CodePipeline |
| 컨테이너 표준 | ECR + (GHA or CodeBuild) → ECS/EKS |

→ 신규 프로젝트는 **GitHub Actions + OIDC** 가 기본. CodeCommit 은 사용 빈도 줄어듦.

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../../iac/iac|↗ IaC 전반]]
- [[../../cicd/cicd|↗ CI/CD 전반]]
