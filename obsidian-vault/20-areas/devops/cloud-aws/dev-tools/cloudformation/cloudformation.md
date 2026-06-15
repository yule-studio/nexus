---
title: "AWS CloudFormation"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:47:00+09:00
tags:
  - aws
  - iac
  - cloudformation
  - dev-tools
---

# AWS CloudFormation

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CloudFormation hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS **공식 IaC**. YAML / JSON 으로 자원을 선언하면 AWS 가 그대로 만들고 / 바꾸고 / 지운다. **무료**.

---

## 2. 왜 / 언제

- AWS 자원을 콘솔 클릭 대신 코드로
- 환경 (dev / staging / prod) 동일하게 복제
- 변경 사항을 PR 리뷰 / git history 로 추적
- 자원 의존성 자동 처리 (DB → 응용 순서)
- 실패 시 자동 rollback

> CDK / SAM / Terraform 도 결국 CloudFormation 또는 그에 준하는 API 를 호출. CloudFormation 은 **AWS-only IaC 의 토대**.

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Template** | YAML / JSON 파일 — 자원 선언 |
| **Stack** | template 의 한 배포 인스턴스 |
| **Change Set** | apply 전 변경사항 미리보기 (`diff` 같은 것) |
| **Drift Detection** | 콘솔에서 직접 바꾼 변경 발견 |
| **Stack Set** | **여러 account / region** 에 같은 stack 배포 |
| **Nested Stack** | stack 안에 stack |
| **Output / Export / Import** | stack 간 값 전달 |
| **Parameter** | 입력 변수 |
| **Mapping / Condition** | 환경별 분기 |

---

## 4. 최소 예시

```yaml
AWSTemplateFormatVersion: "2010-09-09"

Parameters:
  Env:
    Type: String
    Default: dev

Resources:
  AppBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "myapp-${Env}-${AWS::AccountId}"
      VersioningConfiguration:
        Status: Enabled

Outputs:
  BucketArn:
    Value: !GetAtt AppBucket.Arn
    Export:
      Name: !Sub "myapp-${Env}-bucket-arn"
```

```bash
aws cloudformation deploy \
  --template-file template.yml \
  --stack-name myapp-dev \
  --parameter-overrides Env=dev \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## 5. CloudFormation vs CDK vs Terraform

| | CloudFormation | CDK | Terraform |
| --- | --- | --- | --- |
| 형태 | YAML 선언 | TS/Python 코드 → CFN | HCL 선언 |
| 멀티클라우드 | X (AWS only) | X (AWS) | ✅ |
| 학습 곡선 | 낮음 | 중간 | 중간 |
| 표현력 | 낮음 (Mapping 정도) | **높음** (loop / 추상화) | 중간 (count / for_each) |
| state | AWS 가 관리 | AWS 가 관리 (CFN) | **별도 (S3 + lock)** |
| 함정 | 길어짐 / drift | bootstrap / 의존성 | state 관리 / 권한 |

→ 권장:
- **작고 AWS-only** → CloudFormation
- **크고 추상화 필요** → **CDK**
- **멀티클라우드 / 표준** → **Terraform**

---

## 6. 함정 / 비용

- **무료** (Stack 자체). 단, CloudFormation 이 만든 자원의 비용은 발생.
- **Stack 삭제** = 자원 모두 삭제 — `DeletionPolicy: Retain` 으로 보호.
- **수동 변경 (drift)** — 콘솔에서 직접 바꾸면 다음 apply 가 실패.
- **변경 종류 차이** — `Replacement` (자원 재생성, 데이터 손실 위험) vs `Modification` vs `No interruption` — Change Set 으로 확인 후 apply.
- **IAM capability** — IAM 자원 생성 시 `CAPABILITY_NAMED_IAM` 명시 필요.
- **YAML 길이 폭발** — 500줄 넘으면 nested stack / CDK 로 전환 고려.

---

## 7. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../cdk/cdk]] — 코드 기반 CFN
- [[../../iac/iac|↗ IaC 전반]]
