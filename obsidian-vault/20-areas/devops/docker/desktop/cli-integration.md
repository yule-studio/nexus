---
title: "Docker Desktop — CLI Integration / docker context / Automation"
kind: knowledge
status: current
created_at: 2026-05-22T14:18:00+09:00
tags: [devops, docker, docker-desktop, cli, context, automation]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[../orbstack/cli]]"
---

# Docker Desktop — CLI Integration

> Desktop GUI 와 별개로 CLI 자동화. docker context / desktop quit/restart / 자동 startup /
> Apple Script / PowerShell.

## 1. docker context — multi-engine 전환

```bash
docker context ls
# NAME              DESCRIPTION                ENDPOINT
# default*          Default endpoint           unix:///var/run/docker.sock
# desktop-linux     Docker Desktop             unix:///Users/me/.docker/desktop/docker.sock
# orbstack          OrbStack                   unix:///Users/me/.orbstack/run/docker.sock
# remote-prod       remote prod server         ssh://prod-server
```

전환:
```bash
docker context use desktop-linux
docker context use orbstack
docker context use remote-prod
```

확인:
```bash
docker context show       # 현재
docker info               # 자세히
```

### 새 context 생성

```bash
# SSH remote docker
docker context create remote-prod \
  --docker host=ssh://user@prod-server.example.com \
  --description "Production server"

# TCP socket
docker context create tcp-test \
  --docker host=tcp://192.168.1.100:2376,ca=/path/to/ca.pem
```

활용:
- local dev (desktop-linux)
- 회사 prod 서버 ssh 직접 (remote-prod)
- 동료 PC 디버깅 (peer-machine)

## 2. Desktop 자동화 — Start / Stop / Restart

### macOS

```bash
# 시작
open -a Docker
# 또는
open --background -a Docker

# 종료
osascript -e 'quit app "Docker"'
# 또는 더 강력
pkill -f "Docker Desktop"

# 재시작 (combination)
osascript -e 'quit app "Docker"' && sleep 5 && open -a Docker
```

### Windows (PowerShell)

```powershell
# 시작
Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"

# 종료
Stop-Process -Name "Docker Desktop"
# 또는 docker daemon 만
docker version  # 안 됨 확인
```

### Linux

```bash
systemctl --user start docker-desktop
systemctl --user stop docker-desktop
systemctl --user restart docker-desktop
```

## 3. Desktop CLI helper (4.30+)

```bash
docker desktop start
docker desktop stop
docker desktop restart
docker desktop status
docker desktop disable kubernetes
docker desktop enable extensions
```

> 일부 명령은 platform / version 한정.

## 4. Boot 시 자동 실행

### macOS

System Settings > General > Login Items > +
- `/Applications/Docker.app` 추가

또는 GUI: Docker > Settings > General > Start Docker Desktop when you sign in.

### Windows

Settings > Apps > Startup → Docker Desktop 토글.

## 5. shell completion

```bash
# bash
echo "source <(docker completion bash)" >> ~/.bashrc

# zsh
echo "source <(docker completion zsh)" >> ~/.zshrc

# fish
docker completion fish | source
```

자동 완성 활성 — tab 으로 명령 / option / image name 추천.

## 6. .docker 디렉토리 구조

```
~/.docker/
├── config.json           # credential / context / log / settings
├── contexts/             # named contexts
├── buildx/               # buildx 빌더 설정
├── cli-plugins/          # docker subcommand plugin (buildx / scout / compose)
└── daemon.json           # daemon 설정 (Linux only — Desktop 은 GUI)
```

## 7. CLI plugin 시스템

`docker {subcommand}` 가 `~/.docker/cli-plugins/docker-{subcommand}` 로 위임:
- `docker buildx` → `docker-buildx`
- `docker compose` → `docker-compose`
- `docker scout` → `docker-scout`
- `docker dev` → `docker-dev`
- `docker extension` → `docker-extension`

목록:
```bash
ls ~/.docker/cli-plugins/
```

자체 plugin 작성 가능 — 어떤 실행 파일이든 `docker-{name}` 으로 명명 후 path 에.

## 8. docker daemon API 직접 호출

```bash
# Unix socket 통해 HTTP
curl --unix-socket /var/run/docker.sock http://localhost/version
curl --unix-socket /var/run/docker.sock http://localhost/containers/json
curl --unix-socket /var/run/docker.sock http://localhost/images/json

# 형식
curl --unix-socket ~/.docker/run/docker.sock http://localhost/v1.43/info | jq
```

활용: 자체 monitoring / automation script.

API docs: https://docs.docker.com/engine/api/

## 9. 자동화 예 — pre-build cleanup

```bash
#!/bin/bash
# .git/hooks/pre-commit

set -e

# build cache 가 50GB 넘으면 정리
USAGE=$(docker system df --format json | jq -r '.BuildCache | .[].Size' | head -1)
echo "Build cache: $USAGE"

# 정리
docker buildx prune -af --keep-storage 10GB
```

## 10. PowerShell / WSL 통합

### PowerShell
```powershell
# WSL 안의 docker 명령 호출
wsl -d Ubuntu docker ps

# context 명령으로 WSL 의 docker 사용
docker context use ...
```

### WSL 안에서
```bash
# Windows 의 Docker Desktop 통신
docker version
# Server: ... Docker Desktop ...
```

## 11. JSON 출력 — script 친화

```bash
# 모든 컨테이너 (JSON)
docker ps -a --format json

# 한 줄에 하나의 JSON object — jq 처리
docker ps -a --format json | jq -r '.[] | "\(.ID) \(.Image) \(.Status)"'

# 특정 필드만 (Go template)
docker ps --format '{{.ID}}\t{{.Names}}\t{{.Status}}'

# image manifest
docker image inspect --format '{{json .RootFS.Layers}}' nginx:alpine | jq
```

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `docker` 명령이 sock 찾지 못함 | `docker context use desktop-linux` |
| context 전환 후 다른 context 의 컨테이너 보임 | 의도된 동작 — `docker context show` 확인 |
| Desktop quit 후 CLI 자동 재시작 안 됨 | `open -a Docker` 명시 |
| Apple Script 권한 거부 | System Settings > Privacy > Automation 허용 |
| WSL 에서 docker 안 보임 | Settings > WSL Integration 활성 |

## 13. 영상 / 자료

| 자료 |
| --- |
| [docker context 종합 — Bret Fisher](https://www.youtube.com/watch?v=hKZcVi-d4o4) |
| [Docker API 직접 호출](https://docs.docker.com/engine/api/) |
| [CLI plugin 작성 가이드](https://docs.docker.com/engine/extend/cli_plugins/) |

## 14. 다음

- ▶ [[settings]] — Settings GUI
- ▶ [[../orbstack/cli]] — OrbStack CLI 비교
- ▶ [[../practice/19-github-actions-build-push]] (예정) — CI 통합
- ▶ [[secrets-and-credentials]] — login automation

## 15. 참고

- 공식 CLI: https://docs.docker.com/reference/cli/docker/
- docker context: https://docs.docker.com/engine/manage-resources/contexts/
- Engine API: https://docs.docker.com/engine/api/
