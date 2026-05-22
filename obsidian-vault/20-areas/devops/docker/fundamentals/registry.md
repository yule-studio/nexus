---
title: "Container Registry — Docker Hub / ECR / GCR / Harbor / GHCR"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:20:00+09:00
tags: [devops, docker, registry]
---

# Container Registry — Docker Hub / ECR / GCR / Harbor / GHCR

**[[docker|↑ docker]]**

---

## 1. 비교

| Registry | 사용 | 비용 | 보안 |
| --- | --- | --- | --- |
| **Docker Hub** | 개인 / OSS | 무료 (rate limit) / pro $7/월 | 기본 |
| **AWS ECR** | AWS 통합 | 데이터 전송 + 스토리지 | IAM + scan |
| **GCP Artifact Registry** | GCP 통합 | 동 | IAM + scan |
| **Azure ACR** | Azure 통합 | 동 | RBAC + scan |
| **GitHub Container Registry (GHCR)** | GitHub Actions 통합 | OSS 무료 | GitHub auth |
| **Harbor** | self-hosted | 인프라 비용 | 강 (RBAC, replication) |
| **JFrog Artifactory** | enterprise | 비쌈 | 통합 |

---

## 2. 본 vault 권장

| 환경 | Registry |
| --- | --- |
| MVP / OSS | GitHub Container Registry |
| AWS | ECR |
| GCP | Artifact Registry |
| Azure | ACR |
| self-hosted / enterprise | Harbor |

---

## 3. Push / Pull (GHCR 예시)

```bash
# Login
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag
docker tag myapp:1.0 ghcr.io/USERNAME/myapp:1.0

# Push
docker push ghcr.io/USERNAME/myapp:1.0

# Pull
docker pull ghcr.io/USERNAME/myapp:1.0
```

---

## 4. ECR (AWS)

```bash
# Login (12h 만료)
aws ecr get-login-password --region ap-northeast-2 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.ap-northeast-2.amazonaws.com

# Repo 생성
aws ecr create-repository --repository-name myapp

# Push
docker tag myapp:1.0 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:1.0
docker push 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:1.0
```

---

## 5. Tag 전략

| Tag | 용도 |
| --- | --- |
| `1.2.3` | 정확한 버전 (SemVer) |
| `1.2` | 자동 patch |
| `1` | 자동 minor |
| `latest` | 최신 (production 금지) |
| `git-${SHA}` | git SHA — 추적성 |
| `pr-123` | PR 별 preview |
| `staging` / `production` | 환경 별 (CD pipeline 가 갱신) |

→ 본 vault: `<semver>` + `<git-sha>` 둘 다 push.

---

## 6. Image scan

| 도구 | 위치 |
| --- | --- |
| ECR scan (자동) | AWS |
| Trivy (Aqua) | CI/CD |
| Snyk | SaaS |
| Grype | OSS |

```bash
trivy image myapp:1.0
```

자세히: [[security]].

---

## 7. 함정

1. **latest tag production** — 재현 X.
2. **public registry 에 secret push** (.env 포함) → 영구 leak.
3. **scan 안 함** → CVE 누출.
4. **retention 정책 없음** → registry 비용 폭주.
5. **rate limit** (Docker Hub anonymous 100 pull/6h) → CI fail.

---

## 8. 관련

- [[docker|↑ docker]]
- [[security]]
- [[../cicd/cicd|↗ cicd (build → push)]]
