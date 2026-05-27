---
title: "Docker Desktop — Scout (Vulnerability / SBOM / Policy)"
kind: knowledge
status: current
created_at: 2026-05-22T14:14:00+09:00
tags: [devops, docker, docker-desktop, scout, security, cve, sbom]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[../practice/13-image-scan-scout-and-trivy]]"
  - "[[../fundamentals/security]]"
---

# Docker Desktop — Scout

> Docker Scout 는 Desktop 의 내장 image scan / SBOM / policy 도구. 무료 tier 충분 (개인 / 작은
> 팀). Trivy / Snyk 의 대안.

## 1. Scout 가 하는 일

| 기능 | 내용 |
| --- | --- |
| **CVE scan** | image 의 모든 layer + OS package + 언어 dependency |
| **SBOM** | CycloneDX / SPDX 자동 생성 |
| **Recommendations** | 더 안전한 base image 추천 |
| **Comparison** | 두 image 의 CVE 차이 (예: before / after) |
| **Policy** | 회사 표준 정책 enforcement (Pro+) |

영상: [Docker Scout 종합](https://www.youtube.com/watch?v=eExjP2pZ7-w)

## 2. GUI — Images 화면

Dashboard > Images > {image} > **Vulnerabilities** 탭:

| 컬럼 | 내용 |
| --- | --- |
| Severity | Critical / High / Medium / Low / Unspecified |
| Package | 영향 받는 패키지 (예: `openssl`) |
| Installed | 현재 버전 |
| Fixed in | 수정된 버전 |
| Source | OS / dependency |

각 CVE 클릭 → 상세 (CVE 번호 / 설명 / 공식 link / 영향 layer).

## 3. CLI 기본

```bash
# 빠른 view
docker scout quickview nginx:1.27-alpine
#  ✗ nginx:1.27-alpine                  3C    12H    27M    8L
#  ✓ nginx:1.27-alpine3.20 (latest)     0C    1H     20M    5L

# 자세한 CVE 목록
docker scout cves nginx:1.27-alpine

# JSON 출력
docker scout cves nginx:1.27-alpine -o json > scout.json

# severity 필터
docker scout cves --only-severity critical,high nginx:1.27-alpine
```

## 4. SBOM 생성

```bash
docker scout sbom nginx:1.27-alpine
# 또는 CycloneDX 형식
docker scout sbom --format cyclonedx-json -o sbom.json nginx:1.27-alpine
```

자세히 → [[../practice/13-image-scan-scout-and-trivy]]

## 5. Recommendations — 권장 base

```bash
docker scout recommendations nginx:1.27-alpine
# Target: nginx:1.27-alpine
# Recommended base image updates:
#   ▶ nginx:1.27-alpine3.20
#     -3 Critical, -10 High, -5 Medium
#   ▶ nginx:stable-alpine
#     ...
```

> base 만 바꾸고 빌드 → 즉시 CVE 감소.

## 6. Comparison

```bash
docker scout compare myapp:v1 --to myapp:v2
# Image differences:
#   ┌─────────────────────────────────────────┐
#   │ Critical   3 → 0    ✓ -3                │
#   │ High      12 → 2    ✓ -10               │
#   │ Medium    27 → 18   ✓ -9                │
#   └─────────────────────────────────────────┘
```

활용: PR 마다 base image 변경 시 영향 확인.

GUI: 두 image 선택 후 "Compare".

## 7. Policy (Pro+)

회사 표준 정책 정의 + image 가 위반하는지 자동 검사.

기본 정책 예:
- "No Critical / High CVE in production images"
- "Base image up-to-date within 30 days"
- "No secrets in image"
- "License compliance"

```bash
docker scout policy myapp:v1
```

Docker Hub organization 의 Scout 설정에서 정책 정의.

## 8. CI 통합

### GitHub Actions

```yaml
- uses: docker/scout-action@v1
  with:
    command: cves
    image: ghcr.io/${{ github.repository }}:${{ github.sha }}
    severity-threshold: high           # high 이상이면 fail
    write-comment: true                # PR 에 comment
    keep-previous-comments: true
```

```yaml
- uses: docker/scout-action@v1
  with:
    command: compare
    image: myapp:${{ github.sha }}
    to: myapp:latest
    only-severities: critical,high
```

### 결과 → SARIF (GitHub Security 통합)

```yaml
- uses: docker/scout-action@v1
  with:
    command: cves
    image: myapp:v1
    sarif-file: scout.sarif

- uses: github/codeql-action/upload-sarif@v3
  with: { sarif_file: scout.sarif }
```

## 9. Trivy / Grype 와 차이

| 도구 | 강점 | 약점 |
| --- | --- | --- |
| **Docker Scout** | Desktop 통합 + recommendation + policy | 일부 무료 한도 |
| **Trivy** | OSS + 빠름 + secret/misconfig 통합 scan | recommendation 약함 |
| **Grype** | OSS + SBOM 분리 (Syft) | UI 없음 |
| **Snyk** | IDE 통합 | 유료 위주 |

> 권장: Desktop 사용자는 **Scout (GUI)** + **Trivy (CI)** 조합.

## 10. 한도 / 가격

- **무료** (anonymous): image 비교 / quickview 가능, 일부 깊은 분석 한도
- **Docker Personal**: 200 pull / 6h, Scout basic
- **Pro / Team**: 무제한 + policy + SBOM attach
- **Business**: SSO + audit + 대용량

자세히: https://www.docker.com/products/docker-scout/

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Scout 분석 안 됨 | Docker Hub 로그인 필요 (anonymous 한도) |
| 큰 image scan 매우 느림 | distroless / alpine base 권장 |
| recommendation 이 base 만 변경 → app 비호환 | 변경 후 통합 테스트 |
| SBOM 형식 호환 안 됨 | `--format cyclonedx-json` 또는 SPDX |
| policy 가 화면에만 — CI fail 안 됨 | `severity-threshold` 명시 |

## 12. 영상 / 자료

| 자료 |
| --- |
| [Docker Scout Walkthrough](https://www.youtube.com/watch?v=eExjP2pZ7-w) |
| [Scout vs Trivy 비교 (커뮤니티)](https://www.docker.com/blog/docker-scout/) |
| [Sigstore + Scout — 조합 가이드](https://docs.docker.com/scout/policy/) |

## 13. 다음

- ▶ [[../practice/13-image-scan-scout-and-trivy]] — Scout / Trivy 실습
- ▶ [[../practice/14-sign-and-verify-cosign]] — signing
- ▶ [[../fundamentals/security]] — security 일반
- ▶ [[../production/patterns]] — production scan 통합

## 14. 참고

- Scout 공식: https://docs.docker.com/scout/
- Scout policy: https://docs.docker.com/scout/policy/
- CVE DB: https://nvd.nist.gov/
