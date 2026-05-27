---
title: "ECR 03-1 — Scanning (Basic / Enhanced Inspector v2)"
kind: knowledge
status: current
created_at: 2026-05-22T17:48:00+09:00
tags: [aws, ecr, security, scan, inspector, cve]
home_hub: ecr
---

# ECR Scanning

> Basic (Clair) — 무료 push scan / Enhanced (Inspector v2) — 지속 모니터 + OS + 언어 package + EventBridge.

## 1. 2 종류 비교

| | Basic Scan | Enhanced Scan (Inspector v2) |
| --- | --- | --- |
| Engine | Clair | Amazon Inspector |
| Trigger | push 시 | push + 지속 |
| Package types | OS only | OS + 언어 (Python/Node/Java/Go/Ruby) |
| 비용 | 무료 | $0.09/scan + 지속 $0.01/re-scan |
| Re-scan (new CVE 발견 시) | ✗ | ✓ 자동 |
| EventBridge 통합 | ✓ (Basic) | ✓ (풍부) |
| Security Hub 통합 | △ | ✓ |
| 권장 | 시작 / 비용 절약 | production |

## 2. Basic Scan 활성

```bash
aws ecr put-registry-scanning-configuration --scan-type BASIC \
  --rules 'scanFrequency=SCAN_ON_PUSH,repositoryFilters=[{filter=*,filterType=WILDCARD}]'

# Repository 단위
aws ecr put-image-scanning-configuration --repository-name bkurs-be \
  --image-scanning-configuration scanOnPush=true
```

## 3. Enhanced Scan 활성

```bash
# Account 단위 Inspector enable
aws inspector2 enable --resource-types ECR --account-ids 123456789012

# ECR 의 scan type 변경
aws ecr put-registry-scanning-configuration --scan-type ENHANCED \
  --rules 'scanFrequency=CONTINUOUS_SCAN,repositoryFilters=[{filter=bkurs-*,filterType=WILDCARD}]'
```

> `CONTINUOUS_SCAN` — 새 CVE 발견 시 기존 image 자동 re-scan.

## 4. Scan 결과

```bash
# Latest scan
aws ecr describe-image-scan-findings --repository-name bkurs-be \
  --image-id imageTag=v1.0.0 \
  --query 'imageScanFindings.findingSeverityCounts'
# {"HIGH":2,"MEDIUM":5,"LOW":12,"INFORMATIONAL":3}

# Details
aws ecr describe-image-scan-findings --repository-name bkurs-be \
  --image-id imageTag=v1.0.0 \
  --query 'imageScanFindings.findings[?severity==`HIGH`]'
```

Enhanced 의 Inspector finding:
```bash
aws inspector2 list-findings \
  --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_IMAGE"}],"severity":[{"comparison":"EQUALS","value":"HIGH"}]}'
```

## 5. EventBridge 알림

```yaml
EventPattern:
  source: [aws.ecr]
  detail-type: ["ECR Image Scan"]
  detail:
    scan-status: [COMPLETE]
    finding-severity-counts:
      CRITICAL: [{ numeric: [">=", 1] }]
Targets:
  - Arn: arn:lambda:...:function:notify-security
```

Enhanced (Inspector):
```yaml
EventPattern:
  source: [aws.inspector2]
  detail-type: ["Inspector2 Finding"]
  detail:
    severity: [HIGH, CRITICAL]
    resources:
      details:
        awsEcrContainerImage:
          repositoryName: [bkurs-be]
Targets:
  - Arn: arn:lambda:...:function:block-deployment
```

→ Critical 발견 시 자동 deploy 차단 / Discord 알림.

## 6. Severity 수준

| Severity | CVSS | 행동 |
| --- | --- | --- |
| CRITICAL | 9.0-10.0 | 즉시 차단 / 패치 |
| HIGH | 7.0-8.9 | 24-48h 내 패치 |
| MEDIUM | 4.0-6.9 | sprint 안 |
| LOW | 0.1-3.9 | 무시 가능 |
| INFORMATIONAL | — | best practice |
| UNTRIAGED | — | 검토 필요 |

## 7. CI 단계 — fail on HIGH

```yaml
# GitHub Actions
- uses: aws-actions/amazon-ecr-login@v2
- run: docker push 123.dkr.ecr.../bkurs-be:${{ github.sha }}

# Wait for scan
- name: Wait scan
  run: |
    aws ecr wait image-scan-complete --repository-name bkurs-be \
      --image-id imageTag=${{ github.sha }}

- name: Check vulnerabilities
  run: |
    HIGH=$(aws ecr describe-image-scan-findings \
      --repository-name bkurs-be --image-id imageTag=${{ github.sha }} \
      --query 'imageScanFindings.findingSeverityCounts.HIGH' --output text)
    if [ "$HIGH" != "None" ] && [ "$HIGH" -gt "0" ]; then
      echo "::error::$HIGH HIGH severity vulnerabilities found"
      exit 1
    fi
```

## 8. Trivy / Snyk / Docker Scout (보완)

| 도구 | 강점 |
| --- | --- |
| Trivy | OSS, fast, OS + lang + IaC + secret |
| Snyk | dev workflow integration |
| Docker Scout | Docker Hub 통합 |
| ECR Enhanced | AWS native, continuous |

CI 의 PR 단계 fail-on-HIGH 권장. Trivy 가 OSS 라 popular.

```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: 123.dkr.ecr.../bkurs-be:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    exit-code: '1'    # HIGH+ 발견 시 fail
    severity: 'CRITICAL,HIGH'
```

## 9. SBOM (Software Bill of Materials)

Enhanced Scan 이 자동 SBOM 생성:
```bash
aws inspector2 list-coverage --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_IMAGE"}]}'

# SBOM export
aws inspector2 create-cis-scan-configuration ...
```

또는 syft + grype (OSS):
```bash
syft 123.dkr.ecr.../bkurs-be:v1 -o spdx-json > sbom.json
grype sbom:sbom.json
```

## 10. Security Hub 통합

```bash
aws securityhub enable-security-hub
aws inspector2 update-organization-configuration ...
```

→ Inspector / GuardDuty / Config 의 발견 = Security Hub 통합 view.

## 11. 비용 시뮬레이션

```
Production:
- 30 image × $0.09 push scan = $2.7
- Continuous re-scan 매주 = 30 × 4 × $0.01 = $1.2
= $3.9/월/repository

작은 OSS — Basic 만 (무료)
Production — Enhanced
```

## 12. False positive 처리

```bash
# Suppress 규칙
aws inspector2 create-filter --name bkurs-suppress \
  --filter-criteria '{
    "resourceType":[{"comparison":"EQUALS","value":"AWS_ECR_IMAGE"}],
    "findingType":[{"comparison":"EQUALS","value":"PACKAGE_VULNERABILITY"}],
    "vulnerabilityId":[{"comparison":"EQUALS","value":"CVE-2024-XXXX"}]
  }' \
  --action SUPPRESS \
  --reason "False positive — patch backported"
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Scan 결과 안 봄 | EventBridge alarm |
| Push scan only (Basic) | Enhanced continuous |
| Critical 발견 시 prod 그대로 | EventBridge → 자동 차단 |
| 비용 폭주 | scan 대상 filter 좁힘 |
| False positive 무시 안 함 | Suppress filter |
| CI 안 fail | exit 1 on HIGH |

## 14. 다음

- ▶ [[02-immutability-and-signing]]
- ▶ [[03-resource-policy]]
- ▶ [[../practice/02-multi-arch-and-lifecycle]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [ECR Image Scanning](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html) |
| [Inspector v2](https://aws.amazon.com/inspector/) |
| [Trivy](https://github.com/aquasecurity/trivy) |

## 16. 참고

- [[../ecr]]
- [[../../ecs/05-security/04-image-security]]
- [[../../../docker/practice/11-security-scan]]
