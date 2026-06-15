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

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **CloudFormation** | [[cloudformation]] | AWS 공식 IaC (JSON / YAML) — 모든 자원 선언 |
| **CDK** | [[cdk]] | 코드 (TS / Python / Java) 로 IaC → CloudFormation 컴파일 |
| **CodeBuild / CodeDeploy / CodePipeline / CodeArtifact** | (★★☆, 별도 예정) | AWS 내장 CI/CD |
| **SAM / X-Ray / CodeCatalyst / Q Developer** | (★★☆, 별도 예정) | Lambda IaC / 분산 추적 / SDLC 포털 / AI 코드 |

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
