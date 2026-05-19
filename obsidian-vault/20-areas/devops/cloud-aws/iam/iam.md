---
title: "AWS IAM — Identity and Access Management"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:20:00+09:00
tags:
  - aws
  - security
  - iam
---

# AWS IAM

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | IAM 개념 + 정책 + 사용 |

**[[security|↑ Security]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 의 **인증 + 인가** 의 토대. 모든 API 호출이 IAM 검증을 거침. 무료.

---

## 2. 왜

- "누가 무엇을 할 수 있나" 의 정확한 정의
- 최소 권한 (Least Privilege)
- 임시 자격 (Role) → key 노출 없이
- audit (CloudTrail)
- 외부 (Identity Provider) 통합

---

## 3. 핵심 개념

| 개념                               | 의미                                            |
| -------------------------------- | --------------------------------------------- |
| **User**                         | 사람 / 응용 — access key + console password       |
| **Group**                        | User 들의 묶음                                    |
| **Role**                         | 임시 자격 — EC2 / Lambda / 다른 account 가 assume    |
| **Policy**                       | JSON (Effect + Action + Resource + Condition) |
| **Permission Boundary**          | role/user 의 권한 상한                             |
| **SCP** (Service Control Policy) | Organizations 의 account 수준 제한                 |
| **Trust Policy**                 | role 을 누가 assume 할 수 있는가                      |
| **Instance Profile**             | EC2 에 role 부여 wrapper                         |

---

## 4. Policy 구조

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": { "aws:SourceVpc": "vpc-xxx" }
      }
    }
  ]
}
```

평가 순서:
1. 명시적 Deny → 차단
2. Allow 있음 → 허용
3. 둘 다 없음 → 차단 (default deny)

---

## 5. Managed Policy vs Inline

| | Managed (AWS or Customer) | Inline |
| --- | --- | --- |
| 재사용 | 여러 entity | 1 entity 만 |
| 버전 관리 | ✅ (5 버전) | X |
| 권장 | 일반 | 강한 결합 필요 |

AWS Managed:
- `AdministratorAccess`
- `AmazonS3FullAccess`
- `ReadOnlyAccess`
- `PowerUserAccess` (IAM 제외 모두)

→ 옛 / wide. 신규 = customer managed (정밀).

---

## 6. Role — 가장 중요

### 6.1 EC2 instance role

```hcl
resource "aws_iam_role" "ec2" {
  name = "myapp-ec2"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "s3_read" {
  role = aws_iam_role.ec2.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["s3:GetObject"]
      Resource = "arn:aws:s3:::my-bucket/*"
    }]
  })
}

resource "aws_iam_instance_profile" "ec2" {
  name = "myapp-ec2"
  role = aws_iam_role.ec2.name
}

resource "aws_instance" "app" {
  iam_instance_profile = aws_iam_instance_profile.ec2.name
  ...
}
```

→ EC2 안에서 SDK 가 자동으로 role credentials 사용. key 코드 X.

### 6.2 Lambda role
Lambda 의 `execution role` — 비슷한 패턴.

### 6.3 ECS task role
container 안에서 사용. `taskRoleArn` (응용) + `executionRoleArn` (image pull).

### 6.4 cross-account role
다른 account 의 user/role 이 assume — `Principal` 에 다른 account ARN.

---

## 7. IRSA — EKS (K8s)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111222333444:role/myapp
```

→ pod 안의 SDK 가 OIDC 기반으로 IAM role 사용. K8s native.

---

## 8. Federation / SSO

### 8.1 AWS IAM Identity Center (옛 AWS SSO)
- 직원 / 사용자 → 여러 AWS account / role
- Okta / Entra ID / Google 와 통합
- 시간 제한 자격증명

### 8.2 OIDC / SAML
GitHub Actions → IAM role (OIDC):
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::111222333444:role/github-actions
    aws-region: ap-northeast-2
```

→ GitHub secret 의 access key 불필요.

---

## 9. CLI / SDK 자격

우선순위:
1. CLI / SDK option
2. 환경변수 (`AWS_ACCESS_KEY_ID`)
3. `~/.aws/credentials` profile
4. EC2 metadata / ECS / Lambda role (자동)

```bash
aws sts get-caller-identity                # 누가 호출 중인가
aws sts assume-role --role-arn ... --role-session-name ...
```

---

## 10. 보안 권장

1. **Root 거의 안 씀** — admin user / SSO 만
2. **MFA 강제** — console + 중요 작업
3. **Access Key 만들기 X** — Role 만 (SSO / IRSA / Instance profile)
4. **최소 권한** — `*` 회피
5. **Permission Boundary** — 개발자가 self-escalation 못 함
6. **Service Control Policy** (Org) — account 수준
7. **Access Analyzer** — 미사용 권한 발견
8. **CloudTrail** — 모든 API audit
9. **Secrets Manager / Parameter Store** — secret
10. **Rotation** — access key (필요 시) 90일

---

## 11. 분석 / 디버그

```bash
# 권한 시뮬레이션
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::...:role/myapp \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/*

# 모든 정책
aws iam list-attached-role-policies --role-name myapp

# 누가 무엇을 했나
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=alice
```

자세히 → [[cloudtrail]]

---

## 12. 비용

**무료** — IAM 자체. CloudTrail 의 일부 / Access Analyzer 일부 무료.

---

## 13. 사용 시나리오

- 모든 AWS 인프라 (필수)
- 응용 → AWS 서비스 (Role)
- 직원 access (SSO + role)
- CI/CD (OIDC + role)
- multi-account (TGW + assume role)
- cross-account 자원 공유 (Resource Access Manager)

---

## 14. 함정

### 14.1 `*` 너무 광범위
attack surface ↑. specific action / resource.

### 14.2 access key 노출
git / log / S3. **Secrets Manager + rotation + Role**.

### 14.3 root 사용
사고. 별도 admin user.

### 14.4 trust policy 잘못
잘못된 principal = 누구나 assume.

### 14.5 SCP 의 deny
사용자 policy 가 allow 여도 SCP deny → 차단.

### 14.6 boundary 무지
권한 부여하지만 적용 안 됨 — boundary 가 차단.

### 14.7 token expiration
STS token = 1-12 시간. 응용 자동 renewal (SDK 가 처리).

### 14.8 condition 의 정확한 의미
`StringEquals` vs `StringLike` 등. 잘못 = 무의미한 policy.

---

## 15. 학습 자료

- AWS IAM docs
- **IAM Best Practices** AWS whitepaper
- **Access Analyzer**
- **Cloud Security Posture Management** (CSPM) — Prowler / Steampipe / Wiz

---

## 16. 관련

- [[security]] — Security hub
- [[kms]] — 암호화 키 권한
- [[secrets-manager]] — secret + IAM
- [[cloudtrail]] — audit
- [[security-theory|↗ 보안 이론]]
