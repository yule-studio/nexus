---
title: "AWS CDK (Cloud Development Kit)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:48:00+09:00
tags:
  - aws
  - iac
  - cdk
  - dev-tools
---

# AWS CDK

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CDK hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**TypeScript / Python / Java / C# / Go 코드** 로 AWS 인프라를 정의. 빌드하면 CloudFormation template 으로 컴파일되어 배포된다.

---

## 2. 왜 / 언제

- CloudFormation YAML 이 길고 반복적
- 분기 / 반복 / 유틸 함수 / 모듈 재사용이 필요
- 응용 코드와 같은 언어로 인프라 (한 PR 에 둘 다)
- AWS best-practice 가 **L2 construct** 에 녹아 있음 (보안 그룹 / 권한 자동)
- 단위 테스트 가능 (jest / pytest)

> AWS-only 환경에서는 CloudFormation 대신 **CDK 가 신규 표준**.

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **App** | 최상위 컨테이너 (CDK 실행 단위) |
| **Stack** | CloudFormation Stack 1개에 대응 |
| **Construct** | 자원 단위. L1 (raw CFN) / L2 (best-practice 래퍼) / L3 (패턴) |
| **synth** | TS/Python → CloudFormation YAML 생성 |
| **deploy** | synth + CloudFormation 배포 |
| **bootstrap** | account/region 마다 한 번 — CDK 가 쓰는 S3 / IAM 준비 |
| **Aspect** | stack 의 모든 자원에 일괄 적용 (예: 태그) |

construct level:
- **L1 (Cfn*)** — CFN raw (1:1)
- **L2 (Bucket / Function / Cluster)** — 권장. 합리적 기본값
- **L3 (Pattern, e.g. `ApplicationLoadBalancedFargateService`)** — 패턴 묶음 1줄로

---

## 4. 최소 예시 (TypeScript)

```typescript
import * as cdk from 'aws-cdk-lib';
import { Bucket } from 'aws-cdk-lib/aws-s3';
import { Function, Runtime, Code } from 'aws-cdk-lib/aws-lambda';

const app = new cdk.App();
const stack = new cdk.Stack(app, 'MyAppStack', { env: { region: 'ap-northeast-2' }});

const bucket = new Bucket(stack, 'AppBucket', {
  versioned: true,
  encryption: cdk.aws_s3.BucketEncryption.S3_MANAGED,
});

const fn = new Function(stack, 'Worker', {
  runtime: Runtime.NODEJS_20_X,
  code: Code.fromAsset('lambda'),
  handler: 'index.handler',
});

bucket.grantRead(fn);   // ← IAM policy 자동 생성. 권한 실수 X
```

```bash
npx cdk bootstrap         # account/region 마다 한 번
npx cdk diff              # CloudFormation diff
npx cdk deploy            # 배포
npx cdk synth > stack.yml # CFN template 만 추출
```

---

## 5. CDK vs CloudFormation vs Terraform

| | CDK | CloudFormation | Terraform |
| --- | --- | --- | --- |
| 형태 | 코드 (TS/Python) | YAML | HCL |
| 백엔드 | CloudFormation | CloudFormation | Terraform state |
| 멀티클라우드 | X | X | ✅ |
| best-practice 자동 | **L2 construct** | X | module |
| 디버그 | TS / pytest 가능 | template 검사 | plan |
| 함정 | bootstrap / 의존성 | template 길이 | state lock |

→ AWS only + 새 프로젝트 = **CDK 디폴트**. 멀티클라우드 / 표준 = Terraform.

---

## 6. 함정 / 비용

- **무료** (CDK 자체). CloudFormation + 자원 비용만.
- **bootstrap** — `cdk bootstrap aws://<account>/<region>` 안 하면 deploy 실패.
- **construct id 변경 = 자원 교체** — 이름 바꿀 때 주의 (`removalPolicy`).
- **synth 의 token** — runtime 값을 직접 `if` 못 함 (`Fn.condition` 사용).
- **bundle 시간** — Lambda asset bundling 이 느리면 `bundling.dockerImage` 캐시.
- **construct 버전** — `aws-cdk-lib` 메이저 = breaking. `cdk migrate` 활용.
- **drift** — 콘솔에서 직접 바꾸면 다음 deploy 가 되돌림.

---

## 7. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../cloudformation/cloudformation]]
- [[../../iac/iac|↗ IaC 전반]]
