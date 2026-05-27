---
title: "Docker 실습 14 — Sign / Verify with Cosign (Sigstore)"
kind: practice
status: current
created_at: 2026-05-22T13:35:00+09:00
tags: [devops, docker, practice, security, cosign, sigstore]
home_hub: docker
related:
  - "[[practice]]"
  - "[[13-image-scan-scout-and-trivy]]"
---

# 실습 14 — Sign / Verify with Cosign

> 시간: 30분. 학습 포인트: image signing 의 의미 (supply chain attack 방어) / Cosign + Sigstore /
> keyless (OIDC) / SBOM 서명 / policy enforcement (Kyverno / OPA / Sigstore policy-controller).

## 1. 왜 image signing?

**supply chain attack 시나리오**:
1. attacker 가 registry 침투
2. `myapp:v1` 의 image 를 악성 image 로 교체 (같은 tag)
3. production 가 새 image pull → 침해

**signing 으로 방어**:
- image 생산자가 private key 로 서명
- 운영 환경이 public key 로 검증
- 변조 시 검증 실패 → pull 거부

영상: [Sigstore + Cosign — CNCF](https://www.youtube.com/watch?v=KIjL3xRYNgQ)

## 2. Cosign / Sigstore

- **Cosign**: image / SBOM / attestation 서명 도구 (CNCF Sigstore project)
- **Sigstore**: keyless 서명 + 공개 transparency log (Rekor) + OIDC (Fulcio)
- **Rekor**: tamper-proof append-only log (서명 영구 기록)
- **Fulcio**: 단기 인증서 발급 (OIDC 기반)

장점: **keyless** — 영구 private key 보관 / 교체 부담 없음.

## 3. 설치

```bash
brew install cosign
cosign version
```

## 4. 기본 — key-based signing

### key 생성

```bash
cosign generate-key-pair
# Enter password for private key:
# Private key written to cosign.key
# Public key written to cosign.pub
```

> `cosign.key` 는 password 로 암호화 — git commit 가능 (그러나 `.gitignore` 권장).
> `cosign.pub` 는 모두 공개 가능.

### sign

```bash
# image push
docker tag myapp:v1 ghcr.io/org/myapp:v1
docker push ghcr.io/org/myapp:v1

# sign
COSIGN_PASSWORD=mypass cosign sign --key cosign.key ghcr.io/org/myapp:v1
# Pushed signature to: ghcr.io/org/myapp:sha256-abc...sig
```

### verify

```bash
cosign verify --key cosign.pub ghcr.io/org/myapp:v1
# Verification for ghcr.io/org/myapp:v1 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - Existence of the claims in the transparency log was verified offline
#   - The signatures were verified against the specified public key
```

## 5. keyless signing (OIDC) — 권장

private key 보관 / 교체 부담 없음. OIDC provider (GitHub / Google / Microsoft) 인증.

### 로컬

```bash
COSIGN_EXPERIMENTAL=1 cosign sign ghcr.io/org/myapp:v1
# 브라우저 열림 → Google/GitHub OIDC 로그인
# 단기 (10분) 인증서 발급 → 서명 → Rekor log 기록
```

### GitHub Actions (OIDC)

```yaml
permissions:
  id-token: write           # OIDC token 발급
  packages: write

jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & push
        id: build
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - uses: sigstore/cosign-installer@v3
      - name: Sign (keyless)
        env:
          COSIGN_EXPERIMENTAL: 1
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

### verify (keyless)

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/org/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/org/myapp:v1
```

→ "이 image 는 `org/*` GitHub repo 의 OIDC 로 서명됨" 보장.

## 6. SBOM 서명 + attestation

SBOM (실습 13) 을 image 에 attach + 서명:

```bash
# SBOM 생성
syft ghcr.io/org/myapp:v1 -o cyclonedx-json > sbom.json

# attach + sign
cosign attest --predicate sbom.json --type cyclonedx \
  --key cosign.key ghcr.io/org/myapp:v1
```

verify:
```bash
cosign verify-attestation --key cosign.pub \
  --type cyclonedx ghcr.io/org/myapp:v1
```

## 7. SLSA provenance — build 출처 증명

CI 가 어디서 / 어떤 source / 어떤 builder 로 build 했는지 증명.

`docker/build-push-action@v6` 의 `provenance: true` (default) — BuildKit 가 자동 attestation 생성:

```bash
# 검증
cosign verify-attestation --type slsaprovenance \
  --certificate-identity-regexp "https://github.com/org/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/org/myapp:v1
```

## 8. policy enforcement — K8s

서명 안 된 image / 알 수 없는 서명자 → pull 거부.

### Sigstore policy-controller (admission webhook)

```bash
helm install policy-controller \
  -n cosign-system --create-namespace \
  sigstore/policy-controller
```

`policy.yaml`:
```yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata: { name: only-signed-images }
spec:
  images:
    - glob: "ghcr.io/org/*"
  authorities:
    - keyless:
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: ^https://github.com/org/.*$
        url: https://fulcio.sigstore.dev
```

→ ghcr.io/org/* 의 image 만 사용 가능, 검증 안 되면 Pod 거부.

### Kyverno

`policy.yaml`:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: verify-image-signatures }
spec:
  rules:
    - name: verify-signature
      match: { any: [{ resources: { kinds: [Pod] } }] }
      verifyImages:
        - imageReferences: ["ghcr.io/org/*"]
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/org/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

## 9. 실습 — sign + scan + verify 종합

```bash
# 1. build
docker build -t ghcr.io/org/myapp:v1 .
docker push ghcr.io/org/myapp:v1

# 2. scan
trivy image --severity CRITICAL,HIGH ghcr.io/org/myapp:v1
docker scout cves ghcr.io/org/myapp:v1

# 3. SBOM
syft ghcr.io/org/myapp:v1 -o cyclonedx-json > sbom.json

# 4. sign (keyless)
COSIGN_EXPERIMENTAL=1 cosign sign ghcr.io/org/myapp:v1

# 5. SBOM attest
cosign attest --predicate sbom.json --type cyclonedx \
  ghcr.io/org/myapp:v1

# 6. verify (다른 머신에서)
cosign verify \
  --certificate-identity USER@example.com \
  --certificate-oidc-issuer https://accounts.google.com \
  ghcr.io/org/myapp:v1
```

## 10. cosign tree — 서명 확인

```bash
cosign tree ghcr.io/org/myapp:v1
# 📦 Supply Chain Security Related artifacts for an image: ghcr.io/org/myapp:v1
# └── 💾 Attestations for an image tag: ghcr.io/org/myapp:sha256-...att
#    ├── 🍒 sha256:111... (cyclonedx)
#    └── 🍒 sha256:222... (slsaprovenance)
# └── 🔐 Signatures for an image tag: ghcr.io/org/myapp:sha256-...sig
#    └── 🍒 sha256:333...
```

## 11. notation (CNCF alternative)

Cosign 의 대안 — Notary v2 / `notation` CLI.

```bash
notation sign ghcr.io/org/myapp:v1
notation verify ghcr.io/org/myapp:v1
```

Cosign 이 더 많이 사용 (Sigstore 표준).

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `COSIGN_EXPERIMENTAL=1` 안 켜져 keyless 안 됨 | env var 설정 또는 cosign 2.x 의 `--yes` |
| private key 분실 | keyless 사용 (영구 보관 불필요) |
| verify 시 OIDC identity 매칭 안 됨 | `--certificate-identity-regexp` 정확한 패턴 |
| Rekor log 미동기 | network — `--skip-rekor` (테스트만) |
| policy 가 system pod 까지 차단 | namespace selector + exclusion |

## 13. 다음

- ▶ [[15-healthcheck-restart-resources]] — 운영
- ▶ [[19-github-actions-build-push]] — CI 통합
- ▶ [[../fundamentals/security]] — security 일반

## 참고

- Cosign: https://docs.sigstore.dev/cosign/overview
- Sigstore: https://www.sigstore.dev/
- Rekor: https://docs.sigstore.dev/rekor/overview
- policy-controller: https://docs.sigstore.dev/policy-controller/overview
- Kyverno: https://kyverno.io/
- SLSA: https://slsa.dev/
- CNCF Sigstore 영상: https://www.youtube.com/watch?v=KIjL3xRYNgQ
