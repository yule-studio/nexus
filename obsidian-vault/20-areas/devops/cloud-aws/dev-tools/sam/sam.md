---
title: "AWS SAM (Serverless Application Model)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:33:00+09:00
tags:
  - aws
  - dev-tools
  - serverless
  - sam
---

# AWS SAM

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | SAM hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

Lambda / API Gateway / DynamoDB / EventBridge 등 **serverless 전용 IaC** — CloudFormation 의 짧은 문법 + 로컬 디버그 (`sam local invoke`).

## 2. 왜 / 언제

- Lambda + API Gateway + DynamoDB 같은 짧은 serverless 응용
- 로컬에서 Lambda invoke / API Gateway 시뮬레이션 (`sam local`)
- 함수별 IAM policy / event trigger 짧게 정의
- CDK 까지는 필요 없는 작은 응용

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Template** | `template.yaml` (CloudFormation + SAM 매크로) |
| **AWS::Serverless::Function** | Lambda 짧은 정의 (event source 포함) |
| **AWS::Serverless::Api** | API Gateway 짧은 정의 |
| **AWS::Serverless::SimpleTable** | DynamoDB 짧은 정의 |
| **sam build** | 의존성 설치 + 패키징 |
| **sam local** | 로컬 Docker 로 Lambda 실행 |
| **sam deploy** | CloudFormation 배포 |

## 4. 최소 template

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Resources:
  GetUser:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: python3.12
      Handler: app.handler
      CodeUri: src/
      Events:
        Api:
          Type: HttpApi
          Properties:
            Path: /users/{id}
            Method: GET
      Policies:
        - DynamoDBReadPolicy: { TableName: !Ref Users }
  Users:
    Type: AWS::Serverless::SimpleTable
```

```bash
sam build
sam local invoke GetUser -e events/get.json
sam deploy --guided
```

## 5. SAM vs CDK vs Terraform (Lambda)

| | SAM | CDK | Terraform |
| --- | --- | --- | --- |
| Lambda 친화 | **최강** | 좋음 | 보통 |
| 로컬 invoke | **sam local** | X | X |
| 표현력 | 낮음 (YAML) | 높음 (TS) | 보통 |
| 일반 자원 | 약함 | 강함 | 강함 |

→ Lambda 만의 작은 응용 = SAM. Lambda 외 자원이 많아지면 = CDK.

## 6. 비용 / 함정

- **무료** (SAM 자체). 배포 자원만 비용.
- `sam local` 은 Docker 필요. 큰 의존성은 빌드 시간 ↑
- Lambda layer 와 함께 쓸 때 layer 경로 주의
- IAM policy `Policies:` 의 SAM managed policy 가 가독성 좋음

## 7. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../cdk/cdk]]
- [[../cloudformation/cloudformation]]
- [[../../compute/lambda/lambda]]
