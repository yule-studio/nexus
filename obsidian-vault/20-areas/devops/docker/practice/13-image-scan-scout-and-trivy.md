---
title: "Docker 실습 13 — Image scan (Scout / Trivy / Grype / Snyk)"
kind: practice
status: current
created_at: 2026-05-22T13:30:00+09:00
tags: [devops, docker, practice, security, scan, cve, sbom]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../fundamentals/security]]"
  - "[[../desktop/scout]]"
---

# 실습 13 — Image scan (Scout / Trivy / Grype / Snyk)

> 시간: 30분. 학습 포인트: CVE 분석 / SBOM (Software Bill of Materials) / policy / 추천 base image /
> CI 통합 / severity (Critical / High / Medium / Low).

## 1. 왜 scan?

- image 안 OS package (e.g. `libxml2 2.9.10`) + 언어 dependency (e.g. `log4j 2.14`)
- 새 CVE 발견 → 즉시 영향 확인 / patch
- 보안 컴플라이언스 (SOC 2 / ISO 27001 / PCI-DSS)
- supply chain (Log4Shell / SolarWinds 사례)

영상: [Container Security 101 — Anchore](https://www.youtube.com/watch?v=KINjI1tlo2w)

## 2. 도구 비교

| 도구 | OSS | 특화 |
| --- | --- | --- |
| **Docker Scout** | freemium | Docker Desktop 통합 + policy / recommendations |
| **Trivy** (Aqua) | OSS | CVE + secret + misconfig + license, 가장 인기 |
| **Grype** (Anchore) | OSS | CVE 빠른 scan + SBOM (Syft) |
| **Snyk** | freemium | OSS + 의존성 추적 + IDE 통합 |
| **Clair** (Red Hat) | OSS | API 중심, Harbor 통합 |
| **Anchore Enterprise** | commercial | policy engine |

본 실습: **Scout** (Desktop 사용자) + **Trivy** (OSS) 둘 다.

## 3. Docker Scout — 가장 빠른 시작

### CLI

```bash
# 빠른 view
docker scout quickview nginx:1.27-alpine
#  ✗ nginx:1.27-alpine             3C    12H    27M    8L
#  ✓ nginx:1.27-alpine3.20 (latest) 0C    1H     ...

# 자세한 CVE
docker scout cves nginx:1.27-alpine

# 권장 image (lower CVE)
docker scout recommendations nginx:1.27-alpine
# -> nginx:1.27-alpine3.20

# 비교
docker scout compare myapp:v1 --to myapp:v2

# policy (Docker Hub Pro)
docker scout policy myapp:v1
```

### GUI (Docker Desktop)

Images 화면 → image 클릭 → "Vulnerabilities" 탭:
- 모든 CVE 목록 (severity 색상)
- 영향 layer / package
- "Fix available" 표시 — 권장 version

### policy

Scout policy 예:
- "No Critical / High CVE in production images"
- "Base image up-to-date within 30 days"
- "No secrets in image"

CI 통합:
```yaml
- uses: docker/scout-action@v1
  with:
    command: cves
    image: myapp:${{ github.sha }}
    severity-threshold: high           # high 이상 발견 시 fail
    write-comment: true                # PR 에 comment
```

## 4. Trivy — OSS 강력

### 설치

```bash
brew install trivy
# 또는
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image myapp:v1
```

### CVE scan

```bash
trivy image nginx:1.27-alpine
# myapp:v1 (debian 12.0)
#
# ┌────────────────┬──────────────┬──────────┬───────────┬───────────────┐
# │     Library    │ Vulnerability│ Severity │ Installed │ Fixed Version │
# ├────────────────┼──────────────┼──────────┼───────────┼───────────────┤
# │ openssl-libs   │ CVE-2024-... │ HIGH     │ 3.0.7-1   │ 3.0.7-2       │
# │ ...                                                                  │
# └────────────────┴──────────────┴──────────┴───────────┴───────────────┘
```

옵션:
```bash
# severity 필터
trivy image --severity HIGH,CRITICAL myapp:v1

# fail 조건 (CI)
trivy image --exit-code 1 --severity CRITICAL myapp:v1

# JSON 출력
trivy image -f json -o report.json myapp:v1

# SBOM 출력 (CycloneDX / SPDX)
trivy image --format cyclonedx --output sbom.json myapp:v1

# offline (vulnerability DB pre-downloaded)
trivy image --skip-update --download-db-only myapp:v1
```

### Trivy 추가 기능

```bash
# 파일 시스템 scan (소스)
trivy fs .

# secret scan (image / fs)
trivy image --scanners secret myapp:v1
trivy fs --scanners secret .

# misconfig (IaC — Terraform / k8s yaml)
trivy config ./k8s-manifests/

# license
trivy image --scanners license myapp:v1
```

## 5. Grype + Syft — Anchore stack

```bash
brew install grype syft

# SBOM 생성 (Syft)
syft myapp:v1 -o cyclonedx-json > sbom.json
syft myapp:v1 -o spdx-json > sbom-spdx.json
syft myapp:v1 -o table

# CVE scan (Grype — SBOM 또는 image 입력)
grype myapp:v1
grype sbom:./sbom.json                    # SBOM 입력 (빠름)
grype myapp:v1 --fail-on high
```

## 6. SBOM (Software Bill of Materials)

표준:
- **CycloneDX** (OWASP)
- **SPDX** (Linux Foundation)

```bash
# Docker Scout
docker scout sbom myapp:v1

# Trivy
trivy image --format cyclonedx --output sbom-cdx.json myapp:v1

# Syft
syft myapp:v1 -o cyclonedx-json > sbom.json
```

SBOM 활용:
- registry attach (cosign attest)
- 컴플라이언스 제출
- 새 CVE 발견 시 affected image 검색 (SBOM 안에 package list 있음)

## 7. 실습 — Spring Boot image scan

```dockerfile
# 의도적으로 오래된 base
FROM eclipse-temurin:21-jdk-jammy
# ...
```

```bash
docker build -t myapp:vulnerable .
docker scout quickview myapp:vulnerable
# ✗ myapp:vulnerable    5C    20H    35M    10L

# 권장 base 로 변경
docker scout recommendations myapp:vulnerable
# -> eclipse-temurin:21-jre-alpine
```

수정:
```dockerfile
FROM eclipse-temurin:21-jre-alpine   # smaller + fewer CVE
# ...
```

재 scan:
```bash
docker build -t myapp:fixed .
docker scout compare myapp:fixed --to myapp:vulnerable
# Improved: -3 Critical, -12 High, -8 Medium
```

## 8. CI 통합 — GitHub Actions

```yaml
name: security-scan
on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Build
        uses: docker/build-push-action@v6
        with:
          tags: myapp:${{ github.sha }}
          load: true

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: trivy.sarif }

      - name: Docker Scout
        uses: docker/scout-action@v1
        with:
          command: cves
          image: myapp:${{ github.sha }}
          severity-threshold: high
```

SARIF 출력 → GitHub Security tab 자동 통합.

## 9. policy as code — 회사 표준

`.trivy.yaml` (repo 안):
```yaml
severity: CRITICAL,HIGH
ignore-unfixed: true       # fix 없는 CVE 무시
scan:
  scanners: [vuln, secret, misconfig]
secret:
  config: .trivysecret.yaml
```

`.trivyignore`:
```
# 한시적 무시 (issue 추적)
CVE-2024-1234   # 2026-06-01 fix 예정
```

## 10. 줄이는 방법 (recommendation 자동화)

| 방법 | 효과 |
| --- | --- |
| **slim / alpine / distroless base** | 패키지 수 ↓ → CVE 표면 ↓ |
| **multi-stage** | 빌드 도구 제거 (실습 04) |
| **non-root user** | privilege escalation 차단 |
| **scratch / static binary** (Go) | OS package 0 |
| **renovate / dependabot** | base image 자동 PR |
| **registry mirror + policy** | 회사 표준 image 강제 |

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `:latest` tag — base 자동 갱신 → 갑자기 CVE | semver pinning + 정기 업데이트 |
| dev dependency 가 prod image 에 남음 | multi-stage (실습 04) |
| secret 이 image 안 (Dockerfile ARG) | BuildKit secret mount (실습 05) |
| OS package 만 scan — 언어 dependency 누락 | Trivy 가 둘 다 scan |
| scan DB 가 outdated | `trivy image --download-db-only` 정기 |

## 12. 다음

- ▶ [[14-sign-and-verify-cosign]] — Cosign image signing
- ▶ [[15-healthcheck-restart-resources]] — 운영 패턴
- ▶ [[../desktop/scout]] — Desktop GUI Scout
- ▶ [[../fundamentals/security]] — security 일반

## 참고

- Docker Scout: https://docs.docker.com/scout/
- Trivy: https://trivy.dev/
- Grype: https://github.com/anchore/grype
- Syft (SBOM): https://github.com/anchore/syft
- CycloneDX: https://cyclonedx.org/
- SPDX: https://spdx.dev/
- Anchore 영상: https://www.youtube.com/watch?v=KINjI1tlo2w
