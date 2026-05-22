---
title: "Docker 실습 19 — GitHub Actions (Build + Scan + Sign + Push)"
kind: practice
status: current
created_at: 2026-05-22T14:45:00+09:00
tags: [devops, docker, practice, ci, github-actions, ghcr]
home_hub: docker
related:
  - "[[practice]]"
  - "[[11-registry-docker-hub-and-ghcr]]"
  - "[[13-image-scan-scout-and-trivy]]"
  - "[[14-sign-and-verify-cosign]]"
---

# 실습 19 — GitHub Actions Build/Scan/Sign/Push

> 시간: 45분. 학습 포인트: build-push-action / OIDC / cache (gha) / multi-platform / scan / sign /
> attestation / matrix / reusable workflow / dependabot.

## 1. 최소 build + push

`.github/workflows/build.yml`:

```yaml
name: build-and-push
on:
  push:
    branches: [dev, main]
  pull_request:
    branches: [dev, main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write          # GHCR 쓰기
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # 자동 — PAT 불필요

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

push:
- PR — build only (push 안 함)
- branch push — push 함

확인:
```bash
docker pull ghcr.io/seum-running/bkurs-be:dev
```

## 2. multi-platform (amd64 + arm64)

```yaml
      - uses: docker/setup-qemu-action@v3

      - uses: docker/build-push-action@v6
        with:
          platforms: linux/amd64,linux/arm64
          # ...
```

> ARM 서버 / Apple Silicon Mac 모두 같은 tag 사용. 자세히 → [[06-multi-platform-build]]

## 3. cache 전략

### type=gha (GitHub Actions cache)
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```
- 무료
- 10GB 한도 (repo)
- 같은 repo 의 다른 PR 도 cache 공유

### type=registry
```yaml
cache-from: type=registry,ref=ghcr.io/org/myapp:buildcache
cache-to: type=registry,ref=ghcr.io/org/myapp:buildcache,mode=max
```
- registry storage 사용
- 한도 없음 (registry 정책 따름)
- cross-org / public 공유 가능

### type=inline
```yaml
cache-to: type=inline
```
- image manifest 안에 cache 포함
- 가장 단순, 효율은 떨어짐

## 4. Trivy scan + SARIF (GitHub Security)

```yaml
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1                   # critical 발견 시 fail

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

→ GitHub repo > Security > Code scanning 에 자동 표시.

## 5. Docker Scout

```yaml
      - uses: docker/scout-action@v1
        with:
          command: cves
          image: ghcr.io/${{ github.repository }}:${{ github.sha }}
          severity-threshold: high
          write-comment: true            # PR 에 comment
          keep-previous-comments: true
```

비교 (PR base 와):
```yaml
      - uses: docker/scout-action@v1
        with:
          command: compare
          image: ghcr.io/${{ github.repository }}:${{ github.sha }}
          to: ghcr.io/${{ github.repository }}:main
          only-severities: critical,high
```

자세히 → [[13-image-scan-scout-and-trivy]]

## 6. Cosign signing (keyless OIDC)

```yaml
permissions:
  id-token: write             # OIDC
  packages: write
  contents: read

jobs:
  build-sign:
    runs-on: ubuntu-latest
    steps:
      # ... build + push ...

      - uses: sigstore/cosign-installer@v3

      - name: Sign (keyless)
        env: { COSIGN_EXPERIMENTAL: 1 }
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

      - name: Verify
        env: { COSIGN_EXPERIMENTAL: 1 }
        run: |
          cosign verify \
            --certificate-identity-regexp "https://github.com/${{ github.repository_owner }}/.*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

자세히 → [[14-sign-and-verify-cosign]]

## 7. SBOM + provenance — 자동

`build-push-action@v6` 의 default:

```yaml
      - uses: docker/build-push-action@v6
        with:
          provenance: true     # SLSA provenance attestation
          sbom: true           # SBOM (SPDX)
          push: true
```

검증:
```bash
docker buildx imagetools inspect ghcr.io/org/myapp:tag
# Manifests + Attestations 표시
```

## 8. AWS ECR push (OIDC)

```yaml
      - name: Configure AWS (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123:role/github-deploy
          aws-region: ap-northeast-2

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Tag & Push
        run: |
          docker tag ${{ steps.build.outputs.imageid }} 123.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:${{ github.sha }}
          docker push 123.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:${{ github.sha }}
```

IAM role trust policy:
```json
{
  "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com",
  "Condition": {
    "StringLike": { "token.actions.githubusercontent.com:sub": "repo:org/repo:*" }
  }
}
```

## 9. matrix build (다중 service)

```yaml
strategy:
  matrix:
    service: [api, worker, web]
steps:
  - uses: actions/checkout@v4
  - uses: docker/build-push-action@v6
    with:
      context: ./${{ matrix.service }}
      tags: ghcr.io/${{ github.repository }}/${{ matrix.service }}:${{ github.sha }}
```

## 10. reusable workflow

`/.github/workflows/build-and-push.yml`:
```yaml
on:
  workflow_call:
    inputs:
      image-name: { required: true, type: string }
      dockerfile: { default: Dockerfile, type: string }
      context:    { default: ., type: string }
    secrets:
      ghcr-token: { required: true }
jobs:
  build:
    # ... 위 build 흐름 ...
```

호출:
```yaml
jobs:
  api:
    uses: ./.github/workflows/build-and-push.yml
    with: { image-name: myapp/api }
    secrets: { ghcr-token: ${{ secrets.GITHUB_TOKEN }} }
```

## 11. Dependabot — base image 자동 PR

`.github/dependabot.yml`:
```yaml
version: 2
updates:
  - package-ecosystem: docker
    directory: /
    schedule: { interval: weekly }
    open-pull-requests-limit: 5
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
```

→ Dockerfile 의 `FROM` / GH Actions 의 version 매주 PR.

## 12. ECS deploy (build 후)

```yaml
      - uses: aws-actions/amazon-ecs-render-task-definition@v1
        id: td
        with:
          task-definition: deploy/ecs-task.json
          container-name: bkurs-be
          image: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.td.outputs.task-definition }}
          service: bkurs-be
          cluster: bkurs-prod
          wait-for-service-stability: true
```

자세히 → [[../../10-projects/bkurs-be/handover/operations/env-deployment]]

## 13. 종합 workflow 예

```yaml
name: ci
on:
  push: { branches: [dev, main] }
  pull_request: { branches: [dev, main] }

permissions:
  contents: read
  packages: write
  id-token: write
  security-events: write

jobs:
  build-scan-sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha
      - uses: docker/build-push-action@v6
        id: build
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true
      - uses: aquasecurity/trivy-action@master
        if: github.event_name != 'pull_request'
        with:
          image-ref: ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
      - uses: github/codeql-action/upload-sarif@v3
        if: always() && github.event_name != 'pull_request'
        with: { sarif_file: trivy.sarif }
      - uses: sigstore/cosign-installer@v3
        if: github.event_name != 'pull_request'
      - name: Sign
        if: github.event_name != 'pull_request'
        env: { COSIGN_EXPERIMENTAL: 1 }
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `permissions:` 누락 — push 거부 | `packages: write` 명시 |
| cache 가 안 먹음 | `cache-to: mode=max` 명시 |
| PR 에서 push 됨 | `push: ${{ github.event_name != 'pull_request' }}` |
| Cosign keyless OIDC fail | `id-token: write` permission |
| Trivy 가 image 못 찾음 | push 후 시도 또는 `image-ref:` 가 digest |

## 15. 영상 / 자료

| 자료 |
| --- |
| [Docker + GitHub Actions — Bret Fisher](https://www.youtube.com/watch?v=mxsobl7sgwY) |
| [build-push-action github](https://github.com/docker/build-push-action) |
| [metadata-action](https://github.com/docker/metadata-action) |
| [Cosign action](https://github.com/sigstore/cosign-installer) |
| [AWS OIDC + GH Actions](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html) |

## 16. 다음

- ▶ [[20-end-to-end-bkurs-style]] — 종합 프로젝트
- ▶ [[../desktop/builder-and-buildkit]] — Build Cloud
- ▶ [[../production/patterns]] — production 통합

## 17. 참고

- 공식 build-push: https://docs.docker.com/build/ci/github-actions/
- Trivy GH action: https://github.com/aquasecurity/trivy-action
- Scout action: https://github.com/docker/scout-action
- SLSA provenance: https://slsa.dev/
