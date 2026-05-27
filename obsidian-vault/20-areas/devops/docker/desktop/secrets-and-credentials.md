---
title: "Docker Desktop — Secrets / Credential Helper / Login"
kind: knowledge
status: current
created_at: 2026-05-22T14:16:00+09:00
tags: [devops, docker, docker-desktop, secrets, credentials, auth]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[../fundamentals/registry]]"
---

# Docker Desktop — Secrets / Credentials

> registry login / credential helper (OS keychain 통합) / 빌드 시 secret / runtime env. 평문
> `config.json` 회피 + secret 안전 주입.

## 1. registry login — 3 방법

### 1.1 GUI

상단 우측 — Sign in (Docker Hub 계정 / GitHub / Google SSO):
- session cookie 보관
- profile / org 표시

### 1.2 CLI

```bash
docker login
docker login ghcr.io
docker login 123.dkr.ecr.ap-northeast-2.amazonaws.com

# token via STDIN (CI 권장)
echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
```

password 직접 인자 (`-p ...`) 는 shell history / process list 노출 — STDIN 권장.

### 1.3 OAuth (Docker Hub)

Docker Desktop > Sign in > **Continue with Google/GitHub/Microsoft**.

## 2. 자격 저장 — `~/.docker/config.json`

기본 저장 형식:

```json
{
  "auths": {
    "https://index.docker.io/v1/": { "auth": "BASE64(USER:PASS)" }
  }
}
```

> ⚠️ 평문 base64 — 권장 안 됨. credential helper 사용.

## 3. Credential Helper — OS keychain 통합

```json
{
  "credsStore": "osxkeychain"      // macOS
  // "credsStore": "wincred"       // Windows
  // "credsStore": "secretservice" // Linux (gnome-keyring / kwallet)
  // "credsStore": "pass"          // Linux (pass + GPG)
}
```

설치 (대부분 OS 에 포함):
```bash
brew install docker-credential-helper           # macOS
apt install docker-credential-secretservice     # Ubuntu
```

CLI `docker login` 시:
- credential 이 keychain 에 저장
- `config.json` 에는 reference 만 (`"<credstore>"`)
- 다른 사용자 / process 가 읽을 수 없음

확인:
```bash
cat ~/.docker/config.json
# {
#   "auths": { "https://index.docker.io/v1/": {} },
#   "credsStore": "osxkeychain"
# }
```

## 4. 여러 registry — credHelpers

```json
{
  "credHelpers": {
    "index.docker.io": "osxkeychain",
    "ghcr.io": "osxkeychain",
    "123.dkr.ecr.ap-northeast-2.amazonaws.com": "ecr-login"
  }
}
```

AWS ECR 의 `docker-credential-ecr-login` — IAM role 기반 자동 token rotation:
```bash
brew install docker-credential-helper-ecr
```

## 5. 빌드 시 secret — BuildKit mount

빌드 중에만 사용, image 에 안 남음:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM alpine
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install --omit=dev
```

```bash
# file 로
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .

# env 로
docker build --secret id=npm,env=NPM_TOKEN -t myapp .
```

자세히 → [[../practice/05-buildkit-cache-and-secret]]

## 6. runtime env — secret 주입

### compose
```yaml
services:
  api:
    image: myapp:v1
    environment:
      DB_PASSWORD: ${DB_PASSWORD}      # .env 또는 host env 에서
    env_file:
      - ./secret.env                    # gitignore!
```

### docker run
```bash
docker run -e DB_PASSWORD="$DB_PASSWORD" myapp        # host env
docker run --env-file ./secret.env myapp              # file
```

### docker secret (Swarm 한정)
```bash
echo "my-secret" | docker secret create db-pass -
docker service create --secret db-pass myapp
# 컨테이너 안 /run/secrets/db-pass 에 mount
```

K8s 의 경우 Secret resource — 별도.

## 7. Docker Hub PAT (Personal Access Token)

password 대신 사용 권장:
1. Docker Hub > Account > Security > **Access Tokens** > New
2. scope: Read / Read+Write / Admin
3. Token 복사 (한 번만 표시)

```bash
docker login -u USERNAME
# Password: <PAT>
```

장점:
- password 변경 시 영향 격리
- scope 제한 (read-only / repo 제한)
- expire 자동 (옵션)

## 8. GitHub PAT (GHCR)

GitHub Settings > Developer settings > Personal access tokens > Tokens (classic):
- scope: `write:packages`, `read:packages`, `delete:packages`
- expire 권장 90일

```bash
export CR_PAT=ghp_xxxx
echo $CR_PAT | docker login ghcr.io -u USERNAME --password-stdin
```

GitHub Actions: `secrets.GITHUB_TOKEN` 자동 — PAT 불필요.

## 9. AWS ECR

```bash
# 자동 token (12시간 만료 — credential helper 가 자동 재발급)
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin \
    123.dkr.ecr.ap-northeast-2.amazonaws.com
```

또는 helper:
```bash
brew install docker-credential-helper-ecr
# config.json 의 credHelpers 설정 후 자동
```

CI: GitHub Actions OIDC + `aws-actions/configure-aws-credentials`.

## 10. config.json 위치

| 플랫폼 | 경로 |
| --- | --- |
| macOS / Linux | `~/.docker/config.json` |
| Windows | `%USERPROFILE%\.docker\config.json` |

기타 위치:
- `/etc/docker/daemon.json` — daemon 설정 (다름)
- `~/.docker/buildx/` — buildx 설정

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `~/.docker/config.json` 의 평문 base64 | credHelpers 사용 |
| password 가 `-p` 인자에 | STDIN (`--password-stdin`) |
| PAT 가 `.bashrc` / `.zshrc` 에 export | secret manager (1Password / pass) |
| CI 에 long-lived AWS key | OIDC + assume role |
| build 시 ARG 로 secret 전달 | BuildKit `--secret` mount |
| Dockerfile 의 `ENV API_KEY=...` | runtime env 또는 secret mount |

## 12. 정리 / 검증

```bash
# 로그인된 registry 목록
docker info | grep -i registry
cat ~/.docker/config.json | jq .auths

# 로그아웃
docker logout
docker logout ghcr.io

# 모든 credential 정리
rm ~/.docker/config.json
```

## 13. 영상 / 자료

| 자료 |
| --- |
| [Docker secret management — Bret Fisher](https://www.youtube.com/watch?v=GwbjRqK01-c) |
| [BuildKit secret mount docs](https://docs.docker.com/build/building/secrets/) |
| [credential helpers 공식](https://docs.docker.com/engine/reference/commandline/login/#credential-stores) |

## 14. 다음

- ▶ [[../practice/05-buildkit-cache-and-secret]] — secret mount 실습
- ▶ [[../practice/11-registry-docker-hub-and-ghcr]] — registry 인증
- ▶ [[scout]] — image 안 secret scan
- ▶ [[cli-integration]] — docker context / login automation

## 15. 참고

- 공식 login: https://docs.docker.com/engine/reference/commandline/login/
- credential helpers: https://docs.docker.com/engine/reference/commandline/login/#credential-stores
- BuildKit secret: https://docs.docker.com/build/building/secrets/
