---
title: "ECR 03-2 — Tag Immutability + Image Signing (Cosign / Sigstore)"
kind: knowledge
status: current
created_at: 2026-05-22T17:50:00+09:00
tags: [aws, ecr, security, immutable, cosign, sigstore, supply-chain]
home_hub: ecr
---

# Tag Immutability + Image Signing

> IMMUTABLE tag (덮어쓰기 차단) + Cosign (Sigstore) 서명 + SLSA provenance. supply chain attack 차단.

## 1. Tag Mutability

```bash
aws ecr put-image-tag-mutability --repository-name bkurs-be \
  --image-tag-mutability IMMUTABLE
```

| | MUTABLE (default) | IMMUTABLE |
| --- | --- | --- |
| 같은 tag 덮어쓰기 | ✓ (위험) | ✗ |
| `latest` 사용 | 가능 | 불가능 |
| Supply chain attack | 위험 | 차단 |
| Production 권장 | ✗ | ★ |

> IMMUTABLE 상태에서 같은 tag 로 push 시:
```
ImageAlreadyExistsException: Image with image-tag 'v1.0.0' already exists.
```

## 2. Versioning 전략

```
Immutable scheme:
- v1.0.0, v1.0.1, v1.1.0  (semver)
- ${git_sha}              (short SHA)
- ${branch}-${git_sha}    (multi-branch)
- ${date}-${git_sha}      (날짜 prefix)

Mutable tag (보조):
- latest (개발만)
- stable (production alias)

→ Production 은 항상 digest (`@sha256:abc...`) 사용
```

## 3. Cosign 서명 (Sigstore)

```bash
# Install
brew install cosign

# KMS key 사용 (AWS 권장)
aws kms create-key --description "cosign key" \
  --key-spec ECC_NIST_P256 --key-usage SIGN_VERIFY
KEY_ID=arn:aws:kms:ap-northeast-2:...:key/abc...

# Sign image
cosign sign --key awskms:///$KEY_ID \
  123.dkr.ecr.../bkurs-be:v1.0.0
```

서명이 같은 ECR repository 의 새 tag (`sha256-abc....sig`) 로 저장. **추가 비용 거의 없음**.

## 4. Verify

```bash
cosign verify --key awskms:///$KEY_ID \
  123.dkr.ecr.../bkurs-be:v1.0.0
```

성공 시:
```
Verification for 123.dkr.ecr.../bkurs-be:v1.0.0 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
```

## 5. ECS / EKS 의 verify

### Kyverno (admission webhook in EKS)
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-cosign-signature }
spec:
  validationFailureAction: enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: verify
      match:
        any: [{ resources: { kinds: [Pod] } }]
      verifyImages:
        - imageReferences: ["123.dkr.ecr.ap-northeast-2.amazonaws.com/bkurs-*"]
          attestors:
            - entries:
                - keys:
                    kms: "awskms:///arn:aws:kms:ap-northeast-2:...:key/abc"
```

→ 서명 안 된 image 의 pod 시작 차단.

### CodePipeline / GitOps stage
Argo CD ApplicationSet pre-sync hook:
```yaml
preSyncJobs:
  - name: verify-signature
    image: ghcr.io/sigstore/cosign:v2.2.0
    command: ["cosign","verify","--key","awskms:///...", "${IMAGE_URI}"]
```

## 6. Keyless signing (OIDC)

GitHub Actions 의 OIDC token 으로 ephemeral key 서명:
```yaml
permissions:
  id-token: write
  contents: read
  packages: write

- uses: sigstore/cosign-installer@v3
- run: |
    cosign sign --yes ${IMAGE_URI}
```

→ AWS KMS 불필요. Sigstore Rekor (public transparency log) 에 기록.

## 7. SLSA Provenance / Attestation

```yaml
- name: Build with provenance
  uses: docker/build-push-action@v5
  with:
    push: true
    provenance: true
    sbom: true
    tags: 123.dkr.ecr.../bkurs-be:v1.0.0
```

→ build provenance (어디서 누가 어떻게 build) 가 같은 image 와 함께 저장.

```bash
# Cosign 으로 attestation 검증
cosign verify-attestation --type slsaprovenance \
  --key awskms:///... \
  123.dkr.ecr.../bkurs-be:v1.0.0
```

## 8. SBOM 동봉

```bash
# syft 로 SBOM
syft 123.dkr.ecr.../bkurs-be:v1.0.0 -o spdx-json > sbom.json

# Cosign 으로 attach
cosign attach sbom --sbom sbom.json --type spdx \
  123.dkr.ecr.../bkurs-be:v1.0.0
```

## 9. AWS Signer (대안 — Lambda Container 만)

Lambda Container 의 Code Signing 은 **AWS Signer** (Cosign 아님):
[[../../lambda/05-security/02-code-signing-and-secrets]] §3 참조.

ECS / EKS 의 일반 image = **Cosign 권장**.

## 10. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| MUTABLE tag → supply chain attack 위험 | IMMUTABLE |
| `latest` production | digest 사용 |
| 서명 없음 | Cosign + KMS |
| Admission webhook 없음 → 서명 검증 skip | Kyverno / OPA |
| KMS key rotation 시 verify 깨짐 | key alias 사용 |
| Provenance / SBOM 없음 | docker/build-push-action provenance |

## 11. Production 권장 stack

```
1. ECR repository: IMMUTABLE
2. CI build: docker buildx + provenance + sbom
3. Push: ECR + ECR Enhanced scan
4. Sign: Cosign + KMS key (alias/cosign)
5. Lifecycle: Keep 30 prod, 7d untagged
6. Cross-region replication (DR)
7. EKS Admission: Kyverno verify-cosign
8. ECS / Lambda Container: CodeDeploy validation hook 로 verify
9. EventBridge: scan finding HIGH → 자동 차단
```

## 12. 다음

- ▶ [[03-resource-policy]]
- ▶ [[../practice/01-create-push-pull]]
- ▶ [[../practice/02-multi-arch-and-lifecycle]]

## 13. 영상 / 자료

| 자료 |
| --- |
| [Sigstore Cosign](https://docs.sigstore.dev/cosign/overview/) |
| [SLSA framework](https://slsa.dev) |
| [Kyverno verifyImages](https://kyverno.io/policies/?policytypes=Verify%2520Images) |

## 14. 참고

- [[../ecr]]
- [[01-scanning-and-inspector]]
- [[../../ecs/05-security/04-image-security]]
- [[../../eks/05-security/03-image-and-runtime]]
