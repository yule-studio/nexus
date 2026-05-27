---
title: "ECS 05-4 — Image Security (ECR scan / immutable / signing / OS hardening)"
kind: knowledge
status: current
created_at: 2026-05-22T14:16:00+09:00
tags: [aws, ecs, security, ecr, image, scan, sign]
home_hub: ecs
---

# ECS Image Security

> Container image supply chain. ECR scan + Enhanced (Inspector) + immutable tag + signing + non-root + minimal base.

## 1. ECR scanning — 2 mode

| Mode | 엔진 | 비용 |
| --- | --- | --- |
| **Basic** (Push scan) | Clair | 무료 |
| **Enhanced** (Inspector) | Amazon Inspector | $0.09/scan |

```bash
aws ecr put-image-scanning-configuration \
  --repository-name bkurs-be \
  --image-scanning-configuration scanOnPush=true

# Enhanced
aws inspector2 enable --resource-types ECR
```

권장: **production = Enhanced**.

## 2. Immutable tag

```bash
aws ecr put-image-tag-mutability \
  --repository-name bkurs-be \
  --image-tag-mutability IMMUTABLE
```

→ 같은 tag 덮어쓰기 금지. supply chain attack 방지. `latest` 사용 금지 → SHA digest 권장.

## 3. Lifecycle policy

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 30 prod images",
      "selection": { "tagStatus": "tagged", "tagPrefixList": ["prod-"], "countType": "imageCountMoreThan", "countNumber": 30 },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Remove untagged > 7 days",
      "selection": { "tagStatus": "untagged", "countType": "sinceImagePushed", "countUnit": "days", "countNumber": 7 },
      "action": { "type": "expire" }
    }
  ]
}
```

## 4. Image signing (Cosign / Sigstore)

```bash
# Sign
cosign sign --key awskms:///alias/cosign 123.dkr.ecr.../bkurs-be:abc123

# Verify
cosign verify --key awskms:///alias/cosign 123.dkr.ecr.../bkurs-be:abc123
```

ECS task 실행 전 admission policy 로 verify (OPA / Kyverno 외부 — ECS 자체 admission 없음, CodePipeline / GitOps stage 에서).

## 5. Dockerfile 보안

### Non-root user
```dockerfile
RUN useradd -u 10001 -m -s /sbin/nologin app
USER 10001:10001
```

### Read-only root fs
Task definition:
```json
{ "readonlyRootFilesystem": true }
```

→ 쓰기 필요한 곳만 volume / tmpfs.

### Minimal base image
| Base | size | 보안 |
| --- | --- | --- |
| `ubuntu:24.04` | ~80MB | ↑ |
| `debian:bookworm-slim` | ~80MB | ↑ |
| `alpine:3.20` | ~5MB | ↓ (musl) |
| `gcr.io/distroless/java21` | ~250MB | ↑↑ |
| `cgr.dev/chainguard/jdk` | ~150MB | ↑↑↑ |

권장: **distroless / chainguard**. shell 없음 → exec / debug 시 ephemeral sidecar.

## 6. SBOM (Software Bill of Materials)

```bash
syft 123.dkr.ecr.../bkurs-be:abc123 -o spdx-json > sbom.json
grype sbom:sbom.json
```

CI 단계에 통합. ECR Enhanced scanning + SBOM Inspector.

## 7. Trivy / Snyk / Docker Scout

| 도구 | 강점 |
| --- | --- |
| Trivy | OSS, 빠름, fixture 풍부 |
| Snyk | dev workflow integration |
| Docker Scout | Docker Hub 통합 |
| ECR Enhanced (Inspector) | AWS native |

CI 의 PR 단계 fail-on-HIGH 권장.

## 8. Build provenance — SLSA

GitHub Actions:
```yaml
permissions:
  id-token: write
  contents: read
  packages: write
  attestations: write

- uses: actions/attest-build-provenance@v1
  with: { subject-name: 123.dkr.ecr.../bkurs-be, subject-digest: ${{ steps.push.outputs.digest }}, push-to-registry: true }
```

## 9. EventBridge — scan finding

```yaml
EventPattern:
  source: [aws.ecr]
  detail-type: ["ECR Image Scan"]
  detail:
    scan-status: [COMPLETE]
    finding-severity-counts:
      CRITICAL: [{ numeric: [">", 0] }]
Targets:
  - Arn: arn:lambda:...:function:block-deployment
```

→ CRITICAL 발견 시 자동 PR / deploy 차단.

## 10. ECR Replication — multi-region DR

```bash
aws ecr put-replication-configuration \
  --replication-configuration "rules=[{destinations=[{region=ap-northeast-1,registryId=123}]}]"
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `latest` tag 사용 | immutable digest |
| Push scan only (Basic) | Inspector 활성 |
| root user | non-root + readonlyRootFilesystem |
| ubuntu full image | distroless / chainguard |
| SBOM 없음 | syft + grype CI |
| critical 알림 없음 | EventBridge |

## 12. 권장 — bkurs-be

- ECR repository: IMMUTABLE + Enhanced scan
- Dockerfile: distroless-java21 + non-root + readonlyRootFilesystem
- CI: Trivy fail-on-HIGH + cosign sign + SBOM
- task definition: digest `bkurs-be@sha256:...` 만 사용
- EventBridge: CRITICAL 발견 → Discord 알림 + 자동 rollback

## 13. 다음

- ▶ [[../practice/01-fargate-first-service]]
- ▶ [[../practice/03-blue-green-with-codedeploy]]
- ▶ [[../../../../docker/practice/11-security-scan]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [ECR Image Scanning — AWS](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html) |
| [Sigstore Cosign](https://docs.sigstore.dev/cosign/overview/) |
| [SLSA framework](https://slsa.dev) |

## 15. 참고

- [[../ecs]]
- [[../../../../docker/production/security]]
