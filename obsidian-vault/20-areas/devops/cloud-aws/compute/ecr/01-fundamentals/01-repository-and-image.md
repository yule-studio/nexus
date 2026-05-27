---
title: "ECR 01-1 — Repository + Image + Manifest"
kind: knowledge
status: current
created_at: 2026-05-22T17:34:00+09:00
tags: [aws, ecr, fundamentals, repository, image, manifest, oci]
home_hub: ecr
---

# ECR Repository + Image

> Registry / Repository / Image / Manifest / Layer / Tag / Digest. OCI Image Format 표준.

## 1. 계층 구조

```
Registry (account 의 ECR endpoint)
  └─ Repository (bkurs-be)
       ├─ Image v1.0.0 (manifest)
       │   ├─ Config (env, cmd, entrypoint, labels)
       │   └─ Layer 1, Layer 2, ... (filesystem diff)
       ├─ Image v1.0.1
       └─ Image latest (tag → manifest digest)
```

## 2. Repository 생성

```bash
aws ecr create-repository --repository-name bkurs-be \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE \
  --encryption-configuration "encryptionType=KMS,kmsKey=alias/ecr"
```

옵션:
- `scanOnPush` — push 시 자동 scan ([[../03-security/01-scanning-and-inspector]])
- `imageTagMutability` — MUTABLE / IMMUTABLE
- `encryption-configuration` — AES256 / KMS

## 3. Image / Manifest / Layer

### OCI Image Manifest
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:abc..."
  },
  "layers": [
    { "mediaType": "...", "size": 32654, "digest": "sha256:..." },
    { "mediaType": "...", "size": 16724, "digest": "sha256:..." }
  ],
  "annotations": {
    "org.opencontainers.image.source": "https://github.com/bkurs/be",
    "org.opencontainers.image.created": "2026-05-22T10:00:00Z"
  }
}
```

### Layer
- 각 `RUN` / `COPY` 의 diff (tar gzip)
- content-addressable (sha256)
- repository 안에서 dedup 됨
- 다른 image 도 같은 layer 면 storage 1번만

## 4. Tag vs Digest

```bash
# Tag (mutable by default)
docker pull 123.dkr.ecr.../bkurs-be:v1.0.0

# Digest (immutable)
docker pull 123.dkr.ecr.../bkurs-be@sha256:abc...
```

| | Tag | Digest |
| --- | --- | --- |
| 가변 | ✓ (MUTABLE 시) | ✗ |
| 가독성 | ↑ | ↓ |
| 재현성 | ↓ | ★ |
| 권장 production | IMMUTABLE tag | digest |

## 5. Image 조회

```bash
# 모든 image
aws ecr list-images --repository-name bkurs-be \
  --query 'imageIds[].[imageTag,imageDigest]' --output table

# Manifest 상세
aws ecr describe-images --repository-name bkurs-be \
  --image-ids imageTag=v1.0.0 \
  --query 'imageDetails[0]'
```

## 6. Image Index (Multi-arch)

```bash
docker buildx imagetools inspect 123.dkr.ecr.../bkurs-be:v1.0.0
```

```
Name:      .../bkurs-be:v1.0.0
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:abc...

Manifests:
  Name:        .../bkurs-be:v1.0.0@sha256:1...
  Platform:    linux/amd64
  Name:        .../bkurs-be:v1.0.0@sha256:2...
  Platform:    linux/arm64
```

→ 같은 tag 가 arch 별 manifest 선택. docker pull 시 client arch 자동 선택.

자세히 → [[../../../docker/practice/06-multi-platform-build]]

## 7. Image Index push (buildx)

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t 123.dkr.ecr.ap-northeast-2.amazonaws.com/bkurs-be:v1.0.0 \
  --push .
```

## 8. Annotation / Label

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/bkurs/be"
LABEL org.opencontainers.image.created="2026-05-22T10:00:00Z"
LABEL org.opencontainers.image.revision="abc123"
LABEL org.opencontainers.image.version="1.0.0"
```

GitHub Actions:
```yaml
- uses: docker/build-push-action@v5
  with:
    push: true
    tags: 123.../bkurs-be:${{ github.sha }}
    labels: |
      org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
      org.opencontainers.image.revision=${{ github.sha }}
      org.opencontainers.image.version=${{ github.ref_name }}
```

## 9. OCI Artifact (image 외 데이터)

ECR 은 Helm chart / SBOM / config 같은 비-Docker artifact 도 저장:
```bash
# Helm chart
helm registry login --username AWS --password $(aws ecr get-login-password)
helm push my-chart-1.0.0.tgz oci://123.dkr.ecr.ap-northeast-2.amazonaws.com/charts

# Cosign signature
cosign sign --key awskms:///alias/cosign 123.dkr.ecr.../bkurs-be:v1.0.0
```

→ signature 도 sha256:abc.sig 형식으로 같은 repo 에 저장.

## 10. Repository 삭제

```bash
# 강제 (모든 image 삭제)
aws ecr delete-repository --repository-name bkurs-be --force

# Image 단위
aws ecr batch-delete-image --repository-name bkurs-be \
  --image-ids imageTag=v0.9.0
```

## 11. Repository 한도

| | |
| --- | --- |
| Repository / Region / Account | 10000 (soft, 증액 가능) |
| Image / Repository | 10000 (soft) |
| Manifest layers | 1000 |
| Image size | 10 GB per layer, 총합 제한 없음 |
| Tag | 1 image 당 100 (별칭) |

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `latest` tag 사용 | IMMUTABLE digest |
| Repository per service 안 분리 | service 별 분리 (audit / 권한) |
| Lifecycle 없음 → storage 폭주 | Lifecycle Policy ([[../02-operations/02-lifecycle-and-replication]]) |
| Multi-arch 안 빌드 | buildx --platform |
| label / annotation 없음 | OCI 표준 label |
| Cross-region replication 없음 | replication 활성 |

## 13. 다음

- ▶ [[02-private-vs-public-vs-pull-through]]
- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-operations/01-push-pull-auth]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [ECR 공식](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html) |
| [OCI Spec](https://github.com/opencontainers/image-spec) |
| [buildx multi-platform](https://docs.docker.com/buildx/) |

## 15. 참고

- [[../ecr]]
- [[../../../docker/docker]]
- [[../../ecs/02-task-definition/02-container-definition]]
