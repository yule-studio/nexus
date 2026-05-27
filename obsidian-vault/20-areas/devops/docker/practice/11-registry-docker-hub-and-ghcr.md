---
title: "Docker 실습 11 — Registry (Docker Hub / GHCR / ECR)"
kind: practice
status: current
created_at: 2026-05-22T13:20:00+09:00
tags: [devops, docker, practice, registry, ghcr, ecr]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../fundamentals/registry]]"
---

# 실습 11 — Registry (Docker Hub / GHCR / ECR)

> 시간: 30분. 학습 포인트: login / push / pull / private repo / token rotation / Docker Hub 무료
> rate limit / GHCR / AWS ECR / Harbor.

## 1. registry 종류

| 종류 | 무료 tier | 적합 |
| --- | --- | --- |
| **Docker Hub** | public 무제한 / private 1 | 오픈소스 / 학습 |
| **GitHub Container Registry (GHCR)** | repository 와 통합 | GitHub user / OSS |
| **AWS ECR** | private only, 500MB free | AWS 인프라 |
| **GCP Artifact Registry** | 0.5GB free | GCP 인프라 |
| **GitLab Container Registry** | 무료 (CE/EE) | GitLab CI |
| **Harbor** (self-hosted) | 무료 OSS | on-prem / 자체 |

영상: [Docker Registry — TechWorld with Nana](https://www.youtube.com/watch?v=apdT0xKOZQ4)

## 2. Docker Hub

### login

```bash
docker login                      # docker.io (default)
# Username: myuser
# Password: ****
# 또는 access token (권장)
```

> 권장: Docker Hub > Account settings > Security > Access tokens 에서 token 생성 → password 대신.

### push

```bash
# image tag — {user}/{repo}:{tag}
docker tag myapp:v1 myuser/myapp:v1
docker push myuser/myapp:v1
# default → docker.io/myuser/myapp:v1
```

### pull

```bash
docker pull nginx:1.27-alpine           # 공식 (library/nginx)
docker pull myuser/myapp:v1
```

### rate limit

| 상태 | pull / 6h |
| --- | --- |
| anonymous | 100 |
| authenticated (free) | 200 |
| Pro / Team | 5000~ |
| Business | 무제한 |

확인:
```bash
docker pull nginx:1.27-alpine 2>&1 | grep ratelimit
# 또는 HEAD 요청으로 (헤더 확인)
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/nginx:pull" | jq -r .token)
curl -sI -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/library/nginx/manifests/latest | grep -i ratelimit
```

회사 환경에서 anonymous 100/6h 가 부족하면 → authenticated 로 전환 또는 registry mirror.

## 3. GitHub Container Registry (GHCR)

GitHub user / organization 의 모든 repo 가 무료 container registry.

### login

```bash
# Personal Access Token (classic) — scope: write:packages, read:packages, delete:packages
export CR_PAT=ghp_xxxxxxxxxxxx
echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
```

### push

```bash
# image tag — ghcr.io/{org}/{repo}:{tag}
docker tag myapp:v1 ghcr.io/seum-running/myapp:v1
docker push ghcr.io/seum-running/myapp:v1
```

기본 private. public 만들기:
- GitHub.com → 본인 profile > Packages > {package} > Package settings > Change visibility

### pull (private)

```bash
docker login ghcr.io -u USERNAME -p $CR_PAT
docker pull ghcr.io/seum-running/myapp:v1
```

### GHCR + GitHub Actions (자동 push)

`.github/workflows/build.yml`:

```yaml
permissions:
  packages: write
  contents: read

steps:
  - uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}     # 자동 — PAT 불필요
  - uses: docker/build-push-action@v6
    with:
      push: true
      tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

`secrets.GITHUB_TOKEN` 이 자동 — repo 의 packages 권한 있음.

자세히 → [[19-github-actions-build-push]]

## 4. AWS ECR

### login (AWS CLI 필요)

```bash
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.ap-northeast-2.amazonaws.com
```

### repository 생성

```bash
aws ecr create-repository --repository-name myapp --region ap-northeast-2
```

### push / pull

```bash
docker tag myapp:v1 \
  123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:v1
docker push \
  123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:v1
```

### lifecycle policy (자동 정리)

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Remove untagged images after 7 days",
    "selection": {
      "tagStatus": "untagged",
      "countType": "sinceImagePushed",
      "countUnit": "days",
      "countNumber": 7
    },
    "action": { "type": "expire" }
  }]
}
```

```bash
aws ecr put-lifecycle-policy --repository-name myapp \
  --lifecycle-policy-text file://policy.json
```

## 5. 컨벤션 — image 이름

```
[registry/][namespace/]repository[:tag][@digest]
```

| 부분 | 예 |
| --- | --- |
| registry | `docker.io` (default) / `ghcr.io` / `123.dkr.ecr.ap-northeast-2.amazonaws.com` |
| namespace | `library` (Docker Hub 공식) / `myuser` / `myorg` |
| repository | `myapp` / `nginx` |
| tag | `v1.2.3` / `latest` |
| digest | `sha256:abc...` |

## 6. credential helper — 권장 (passwords-free)

```bash
brew install docker-credential-helper

# ~/.docker/config.json
{
  "credsStore": "osxkeychain"    # macOS
  # "credsStore": "wincred"      # Windows
  # "credsStore": "secretservice" # Linux
}
```

`docker login` 시 token 이 OS keychain 에 저장 — `~/.docker/config.json` 평문 보관 회피.

## 7. registry mirror — pull 가속

```bash
# Docker Desktop > Settings > Docker Engine
{
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

`docker pull nginx` → mirror.gcr.io 가 cache (Google 의 GCR mirror).

회사 사설 mirror:
```json
{
  "registry-mirrors": ["https://registry.corp:5000"]
}
```

## 8. private registry 활용

```bash
# pull (이미 login 후)
docker pull ghcr.io/org/private-app:v1

# K8s 에서 imagePullSecret
kubectl create secret docker-registry ghcr-cred \
  --docker-server=ghcr.io \
  --docker-username=USERNAME \
  --docker-password=$CR_PAT

# Pod yaml
apiVersion: v1
kind: Pod
spec:
  containers: [...]
  imagePullSecrets:
    - name: ghcr-cred
```

ECS 의 경우:
- task definition 의 `repositoryCredentials.credentialsParameter` 에 SSM Parameter ARN
- 또는 ECS task execution role 에 ECR pull permission

## 9. token rotation

```bash
# Docker Hub access token — 90일 자동 expire 권장
# GHCR PAT — 90일 또는 only-this-action scope

# 회사 환경: SSM Parameter Store 에 저장 → 자동 rotation (AWS Lambda)
```

## 10. 검증

```bash
# image manifest 확인
docker buildx imagetools inspect ghcr.io/org/myapp:v1

# 누가 push 했는지 (registry log)
# Docker Hub: web UI 의 "Tags" 탭
# GHCR: GitHub repo > Packages

# 모든 image 의 registry 확인
docker images --format "table {{.Repository}}:{{.Tag}}\t{{.ID}}"
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `docker push myapp:v1` 가 docker.io/library/ 으로 가려 함 | `docker tag myapp:v1 myuser/myapp:v1` 후 push |
| GHCR 가 private — pull 안 됨 | login 또는 visibility 변경 |
| Hub rate limit 401 | authenticated login 또는 mirror |
| ECR token 12시간 만료 | `aws ecr get-login-password` 매번 또는 helper |
| token 평문 commit | credential helper / Secret manager |
| `latest` push 후 prod 가 의도 다른 image 받음 | semver + digest pin |

## 12. 다음

- ▶ [[12-self-hosted-registry]] — 자체 registry 띄우기
- ▶ [[13-image-scan-scout-and-trivy]] — CVE
- ▶ [[19-github-actions-build-push]] — CI 통합

## 참고

- Docker Hub docs: https://docs.docker.com/docker-hub/
- GHCR: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
- ECR: https://docs.aws.amazon.com/AmazonECR/latest/userguide/
- TechWorld with Nana: https://www.youtube.com/watch?v=apdT0xKOZQ4
