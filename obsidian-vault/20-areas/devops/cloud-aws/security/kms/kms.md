---
title: "AWS KMS — Key Management Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:25:00+09:00
tags:
  - aws
  - security
  - kms
  - encryption
---

# AWS KMS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | KMS 개념 + envelope encryption |

**[[security|↑ Security]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**암호화 키의 매니지드 서비스**. AES-256 / RSA / ECC 키 생성·저장·rotation·audit. AWS 의 모든 암호화 (S3 / EBS / RDS / Secrets Manager / ...) 가 KMS 기반.

---

## 2. 왜

- 응용이 key 직접 관리 = 분실 / leak / rotation 어려움
- KMS = HSM (FIPS 140-2 L3) + IAM + audit + rotation 통합
- **envelope encryption** — 큰 데이터를 DEK 로 암호화, DEK 를 KMS key 로 암호화

대안:
- **CloudHSM** — 단독 hardware HSM (FIPS L3) — 매우 비쌈, multi-tenant 보안 요구
- **외부** — Vault / Akeyless

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **CMK (Customer Master Key)** | KMS 안의 key (= "KMS key" 새 이름) |
| **AWS-managed key** | AWS 가 만든 (별 비용) |
| **Customer-managed key (CMK)** | 사용자 control + 정책 |
| **AWS-owned key** | 보이지 않는 — 일부 서비스만 |
| **Symmetric / Asymmetric** | AES vs RSA/ECC |
| **DEK (Data Encryption Key)** | 실제 데이터 암호화용 (envelope) |
| **Alias** | key 의 별명 (`alias/myapp`) |
| **Key Policy** | key 의 IAM policy |
| **Grant** | 일시 권한 (자동 rotation 용) |

---

## 4. Envelope Encryption

```
1. KMS.GenerateDataKey() →
   - plaintext DEK (응용이 사용)
   - encrypted DEK (KMS key 로 암호화됨)

2. 응용: data = AES_encrypt(plaintext, plaintext_DEK)
   plaintext DEK 즉시 폐기

3. 저장: {ciphertext_data, encrypted_DEK}

4. decrypt: KMS.Decrypt(encrypted_DEK) → plaintext_DEK → AES_decrypt
```

→ KMS 자체는 작은 (~4 KB) plaintext 만 처리. 큰 데이터는 응용 안 AES.

---

## 5. 설치 / 사용

### 5.1 Key 생성

```bash
aws kms create-key \
  --description "myapp encryption" \
  --key-spec SYMMETRIC_DEFAULT \
  --key-usage ENCRYPT_DECRYPT

aws kms create-alias --alias-name alias/myapp --target-key-id <key-id>
```

### 5.2 Terraform

```hcl
resource "aws_kms_key" "myapp" {
  description             = "myapp encryption"
  enable_key_rotation     = true
  deletion_window_in_days = 30                 # 실수 보호

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AccountAdmin"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::111222333444:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        Sid       = "AllowAppUse"
        Effect    = "Allow"
        Principal = { AWS = aws_iam_role.app.arn }
        Action    = ["kms:Encrypt","kms:Decrypt","kms:GenerateDataKey"]
        Resource  = "*"
      }
    ]
  })
}

resource "aws_kms_alias" "myapp" {
  name          = "alias/myapp"
  target_key_id = aws_kms_key.myapp.id
}
```

### 5.3 사용 (Python)

```python
import boto3
kms = boto3.client("kms")

# 작은 데이터 직접 암호화
resp = kms.encrypt(KeyId="alias/myapp", Plaintext=b"secret")
ciphertext = resp["CiphertextBlob"]

# decrypt
resp = kms.decrypt(CiphertextBlob=ciphertext)
plaintext = resp["Plaintext"]

# 큰 데이터 — envelope
resp = kms.generate_data_key(KeyId="alias/myapp", KeySpec="AES_256")
plaintext_dek = resp["Plaintext"]
encrypted_dek = resp["CiphertextBlob"]
# AES_encrypt(file, plaintext_dek) ... 저장
```

### 5.4 AWS 서비스 통합 — KMS 명시

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "site" {
  bucket = aws_s3_bucket.site.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.myapp.arn
    }
  }
}

resource "aws_db_instance" "rds" {
  storage_encrypted = true
  kms_key_id        = aws_kms_key.myapp.arn
  ...
}
```

---

## 6. Key Rotation

```hcl
enable_key_rotation = true                     # 매년 자동 (symmetric)
```

- 옛 키 유지 + 새 키 생성 → 새 암호화는 새 키, 옛 ciphertext 는 옛 키로 decrypt 가능
- transparent — 응용 변화 없음

수동:
```bash
aws kms create-key ...   # 새 키
# 응용 alias 변경
aws kms update-alias --alias-name alias/myapp --target-key-id new-key
# decrypt 는 ciphertext 안의 KeyId 자동 사용
```

---

## 7. Multi-Region Key

```hcl
resource "aws_kms_key" "primary" {
  multi_region = true
  ...
}

resource "aws_kms_replica_key" "replica" {
  provider        = aws.us_east_1
  primary_key_arn = aws_kms_key.primary.arn
}
```

→ 두 region 에서 같은 key id 로 사용 — DR + cross-region 데이터.

---

## 8. Asymmetric Key

```bash
aws kms create-key \
  --key-spec RSA_4096 \
  --key-usage SIGN_VERIFY              # 또는 ENCRYPT_DECRYPT
```

- 서명 / 검증 (JWT, code signing)
- public key 외부 공유 가능
- private key = KMS 안 (export X)

---

## 9. 비용

```
$1 / month per key
API call:
  Symmetric    $0.03 / 10K
  Asymmetric encrypt/decrypt  $0.15 / 10K
  Asymmetric sign/verify       $0.03-$0.15 / 10K
  GenerateDataKey              $0.03 / 10K
Free Tier: 20K request / month
```

→ envelope encryption 으로 KMS API 호출 최소화.

---

## 10. 사용 시나리오

- 모든 영속 데이터 암호화 (S3 / EBS / RDS / Secrets Manager / DynamoDB / ...)
- TLS 인증서 private key (ACM 내부)
- JWT 서명
- 데이터 파일 직접 (envelope)
- Cross-account 안전 공유

---

## 11. 함정

### 11.1 key 삭제 + 영구
복구 불가능 — `deletion_window_in_days` 7-30 일 설정. 그동안 disable 권장.

### 11.2 정책 잘못
잘못된 policy = 영구 lockout (root 도 access X). policy 의 root account principal 유지.

### 11.3 cross-account 사용
key policy 에 다른 account 명시 + 그쪽 IAM policy 도 허용.

### 11.4 KMS API quota
default 5,500 req/s. burst 워크로드 = 한계 도달. envelope.

### 11.5 region binding
key 가 region 단위. multi-region key 또는 envelope 의 DEK 만 region 간.

### 11.6 rotation 의 의미
**materials 만 변경** — key ID 그대로. 응용 변화 X.

### 11.7 grants 의 자동 정리
서비스 가 사용 후 자동 revoke. 수동 grant 는 retire.

---

## 12. 학습 자료

- AWS KMS docs
- **AWS KMS Cryptographic Details** whitepaper
- **AWS Encryption SDK** — 응용 envelope encryption

---

## 13. 관련

- [[security]] — Security hub
- [[iam]] — KMS policy
- [[secrets-manager]] — KMS 로 암호화
- [[s3]] — SSE-KMS
- [[rds]] — at-rest 암호화
