---
title: "Docker Desktop — Dev Environments"
kind: knowledge
status: current
created_at: 2026-05-22T14:00:00+09:00
tags: [devops, docker, docker-desktop, dev-environments]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[dashboard]]"
---

# Docker Desktop — Dev Environments

> git repo URL 한 줄로 일관된 dev 환경 spawn. VS Code Remote Containers 자동 연결. 팀이 같은
> dependency / IDE 설정 / tool 버전 공유.

## 1. 핵심

| 기능 | 효과 |
| --- | --- |
| **Create** | git URL → Dockerfile / docker-compose 기반 환경 자동 생성 |
| **Open in VS Code** | Remote Containers attach (한 클릭) |
| **Share** | tarball export → 팀원이 import 시 동일 환경 |
| **Resume** | 종료 후 재진입 시 상태 유지 |

## 2. 사용 — 새 환경 spawn

Dashboard > **Dev Environments** > **Create**

옵션:
- **Get started** — 예제 (Node / Python / Go)
- **Existing Git repository** — URL 입력 + branch 선택
- **Local directory** — 본인 PC 의 폴더

### 자동 인식

repo 에 다음 파일 있으면 자동 환경 구성:
- `.devcontainer/devcontainer.json` — VS Code Dev Containers 표준
- `docker-compose.yaml` 또는 `compose.yaml`
- `Dockerfile`

없으면 Docker 가 base image 추천 (Node / Python / Go 등).

### `.docker/labs-randomized-name.yaml` (옵션 — Desktop dev env)

```yaml
version: "1"
services:
  app:
    build:
      context: .
      target: dev
    volumes:
      - .:/com.docker.devenvironments.code
    command: sleep infinity
```

## 3. devcontainer.json 의 핵심

VS Code Dev Containers 표준. Desktop Dev Environments 도 이를 인식.

```jsonc
// .devcontainer/devcontainer.json
{
  "name": "bkurs-be dev",
  "image": "eclipse-temurin:21-jdk",       // 또는 dockerComposeFile
  "features": {
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/node:1": { "version": "20" }
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "vscjava.vscode-java-pack",
        "redhat.vscode-yaml",
        "ms-azuretools.vscode-docker"
      ],
      "settings": {
        "java.configuration.runtimes": [{ "name": "JavaSE-21", "default": true }]
      }
    }
  },
  "postCreateCommand": "./mvnw dependency:go-offline",
  "forwardPorts": [8080],
  "remoteUser": "vscode"
}
```

| 키 | 의미 |
| --- | --- |
| `image` / `dockerComposeFile` | base 환경 |
| `features` | 표준 추가 도구 (git / gh / node / aws-cli 등) |
| `customizations.vscode.extensions` | VS Code 확장 자동 설치 |
| `postCreateCommand` | 첫 spawn 후 1회 실행 (의존성 다운로드 등) |
| `forwardPorts` | host 로 자동 port forward |

## 4. VS Code Remote 연결

Dev Environments 화면에서 환경 클릭 → **Open in VS Code** → VS Code 가 새 창에서:
- 컨테이너 내부 attach
- file edit / terminal / debug 모두 컨테이너 안
- 컨테이너의 git / npm / mvn 사용
- localhost forward port 자동

## 5. Share — 팀원에게

```
Dashboard > Dev Environments > {env} > Share
```

옵션:
1. **Image** — image 만 push (예: ghcr.io/org/bkurs-dev:latest) → 팀원이 pull
2. **Tarball** — local file export → 전달

받는 측:
```
Dashboard > Dev Environments > Create > Existing image / Tarball
```

→ 동일 환경 자동 spawn. dependency 다운로드 불필요.

## 6. 종료 / 재진입

- **Stop** — 환경 멈춤 (상태 보존)
- **Resume** — 같은 상태로 재시작
- **Delete** — 영구 삭제 (volume 보존 옵션)

## 7. 활용 시나리오

### 7.1 신규 합류자 onboarding

repo 의 `.devcontainer/devcontainer.json` 가 있으면:
```
git clone ... → Docker Desktop > Dev Environments > Create → 5분 후 같은 환경
```

기존 onboarding:
- JDK 설치 / 버전 매칭 / MySQL 설치 / IDE plugin / env 설정 / ... — 며칠
- Dev Env 사용 시: 위 다 자동.

### 7.2 다중 프로젝트 격리

```
Dev Env 1: bkurs-be (Java 21 + MySQL 9)
Dev Env 2: bkurs-fe (Node 20)
Dev Env 3: legacy (Java 8 + MySQL 5.7)
```

각각 별도 컨테이너 — 호스트 깨끗 / 버전 충돌 없음.

### 7.3 PR review

```
PR branch 에 대해 Dev Env 새로 spawn → 그 안에서 빌드 / 테스트
```

local 머신 영향 0 — 작업 끝나면 삭제.

## 8. CLI

```bash
# 모든 dev env
docker dev list

# create from git
docker dev create --type git --image ghcr.io/org/devenv:latest \
  --name my-env https://github.com/org/repo.git

# start / stop / delete
docker dev start my-env
docker dev stop my-env
docker dev delete my-env
```

## 9. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `devcontainer.json` 의 schema 깨짐 | https://containers.dev/implementors/json_reference/ 참고 |
| postCreateCommand 가 느림 | `--mount=type=cache` (BuildKit) — 의존성 캐시 |
| 큰 file (node_modules / target/) 매번 빌드 | `.dockerignore` |
| VS Code Remote 연결 실패 | Settings > Resources > File sharing 에 repo 경로 포함 |
| 팀원의 OS 차이 (Windows ↔ Mac) | 환경 통일 — devcontainer 가 그 자체로 OS 추상화 |

## 10. 영상 / 자료

| 자료 | 길이 |
| --- | --- |
| [Dev Containers 공식 — VS Code](https://www.youtube.com/watch?v=b1RavPr_878) | 15분 |
| [Docker Dev Environments 공식](https://www.youtube.com/watch?v=AJzIRH_-iZE) | 12분 |
| [containers.dev spec](https://containers.dev/) | 글 |

## 11. 다음

- ▶ [[extensions]] — extension 도 dev 환경 향상
- ▶ [[compose-in-desktop]] — compose + dev env
- ▶ [[builder-and-buildkit]] — image build 가속
- ▶ [[../orbstack/integration]] — OrbStack 의 VS Code 통합 (대안)

## 12. 참고

- 공식 Dev Environments: https://docs.docker.com/desktop/features/dev-environments/
- VS Code Dev Containers: https://code.visualstudio.com/docs/devcontainers/containers
- containers.dev: https://containers.dev/
