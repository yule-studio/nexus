---
title: "Docker Desktop — Dashboard 6 화면 활용"
kind: knowledge
status: current
created_at: 2026-05-22T11:36:00+09:00
tags: [devops, docker, docker-desktop, gui]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[getting-started]]"
  - "[[settings]]"
---

# Docker Desktop — Dashboard 6 화면

> 좌측 사이드바의 6 탭 각각에서 할 수 있는 모든 액션. CLI 와 1:1 매핑 정리.

## 1. Containers

실행 중 + 정지 컨테이너 통합 view.

### GUI 액션

| 액션 | 결과 | CLI 대응 |
| --- | --- | --- |
| ▶ Play | 정지 컨테이너 재시작 | `docker start <id>` |
| ⏸ Stop | graceful stop (SIGTERM → 10s → SIGKILL) | `docker stop <id>` |
| 🔄 Restart | stop + start | `docker restart <id>` |
| 🗑 Delete | 컨테이너 제거 (running 도 force) | `docker rm -f <id>` |
| Open in terminal | 컨테이너 shell exec | `docker exec -it <id> sh` (또는 bash) |
| Open in editor | VS Code Remote Containers attach | (VS Code 통합) |
| View files | 컨테이너 파일시스템 browse + 다운로드 | (Desktop 전용) |
| View details | logs / inspect / stats / files 통합 패널 |  |
| Search filter | name / image / status 필터 | `docker ps --filter ...` |

### Details 패널 4 탭

| 탭 | 내용 |
| --- | --- |
| **Logs** | tail -f 동등. 검색 + Wrap text + Clear |
| **Inspect** | JSON inspect (network / mount / env / labels) |
| **Bind mounts** | host ↔ container path 매핑 |
| **Exec** | shell 입력 (interactive — TTY 자동) |
| **Files** | 트리 view + 우클릭 → save / edit (작은 텍스트는 인라인 편집) |
| **Stats** | 실시간 CPU / MEM / Network IO / Block IO 그래프 |

## 2. Images

local 의 모든 image 목록.

### GUI 액션

| 액션 | 결과 | CLI |
| --- | --- | --- |
| Run | "Run a new container" 다이얼로그 (name / port / env / volume / network) | `docker run ...` |
| Pull | `docker pull` GUI | `docker pull <ref>` |
| Push to Hub | Hub 인증된 image push | `docker push <ref>` |
| Delete | image 삭제 (의존 컨테이너 있으면 force 옵션) | `docker rmi -f <ref>` |
| Inspect | image inspect (history / layers / labels) | `docker image inspect` |
| Vulnerabilities | Docker Scout CVE list (severity 색상) | `docker scout cves` |
| Open file system browser | 각 layer 의 변경 파일 browse |  |

### "Run a new container" 다이얼로그 옵션

- **Container name** (`--name`)
- **Ports** — host:container 매핑 N개 (`-p`)
- **Volumes** — host path : container path : ro|rw (`-v`)
- **Environment variables** — KEY=VAL (`-e`)
- **Network** — bridge / host / 사용자 정의 (`--network`)
- **Restart policy** — no / on-failure / always / unless-stopped (`--restart`)

> 다이얼로그 하단의 "Show full configuration" 가 모든 옵션 (entrypoint / command / user / workdir 등).

## 3. Volumes

### GUI 액션

| 액션 | 결과 | CLI |
| --- | --- | --- |
| Create | named volume 생성 | `docker volume create` |
| In Use by | 어느 컨테이너가 mount 중인지 |  |
| View content | 텍스트 / 작은 파일 browse + save |  |
| Export | tar.gz 로 백업 다운로드 |  |
| Import | tar.gz 업로드 → 새 volume |  |
| Delete | volume 제거 (in-use 거부) | `docker volume rm` |
| Prune | dangling volume 일괄 삭제 | `docker volume prune` |

> Export / Import 가 Desktop GUI 의 강점 — CLI 만으로는 tar 명령 조합 필요.

## 4. Builds

`docker build` / `docker buildx build` 진행 + 완료 history.

### GUI

| 항목 | 표시 |
| --- | --- |
| Build status | running / completed / failed |
| Source | 컨텍스트 경로 + Dockerfile |
| Target | image tag |
| Duration | 빌드 시간 |
| Cache hit rate | 캐시 적중률 (BuildKit) |
| Logs | step 별 펼치기 — 각 RUN/COPY 단계의 stdout |

### 작업

- **Re-run** — 동일 옵션으로 재실행
- **Open in terminal** — 빌드 컨텍스트 폴더로 이동
- **Inspect dockerfile** — Dockerfile 인라인 view

CLI 통합:
```bash
docker buildx build --builder desktop-linux ...
```

자세히 → [[builder-and-buildkit]]

## 5. Dev Environments

git repo clone → 일관된 dev 환경 + VS Code Remote.

### GUI

| 액션 | 결과 |
| --- | --- |
| Create | git URL 또는 local 경로 → 환경 spawn |
| Share | tarball export → 팀원이 import 시 동일 환경 |
| Open in VS Code | Remote Containers 자동 attach |
| Stop / Start | dev env 컨테이너 제어 |
| Delete | env 제거 (volume 보존 옵션) |

자세히 → [[dev-environments]]

## 6. Extensions

Marketplace + 설치된 extension 관리.

### 인기 extension

| 이름 | 책임 |
| --- | --- |
| **Logs Explorer** | 모든 컨테이너 로그 통합 search/filter |
| **Disk usage** | image / volume / build cache 별 disk 사용 시각화 |
| **Volumes Backup & Share** | named volume tar export/import + S3 |
| **Resource Saver** | idle 컨테이너 자동 stop |
| **JFrog**, **Snyk**, **Trivy** | image vulnerability 추가 scanner |
| **Postgres**, **Redis Insight**, **MongoDB Compass** | DB GUI |
| **VS Code Dev Containers** | VS Code attach 통합 |
| **Lens (kube)** | Kubernetes cluster 통합 view |

자세히 → [[extensions]]

## 7. 사이드바 우측 — 환경 / 상태

| 위치 | 표시 |
| --- | --- |
| 상단 | docker context (default / desktop-linux 등) — 클릭 전환 |
| 우상단 | Docker Hub 계정 / 알림 |
| 좌하단 | RAM / CPU / Disk 사용량 실시간 (Settings → Show resource usage) |
| 메뉴바 (🐳) | quick actions — Restart / Stop / Quit / Preferences |

## 8. 단축키

| 액션 | 단축키 (macOS) |
| --- | --- |
| Preferences | ⌘, |
| Quit Docker Desktop | ⌘Q |
| 검색 (Containers / Images) | ⌘F |
| 새로고침 | ⌘R |
| Restart engine | (Preferences → Restart) |

## 9. troubleshooting

| 증상 | 해결 |
| --- | --- |
| Container list 비어있음 | engine 정지 — 메뉴바 → Restart |
| logs 출력 없음 | `tty: true` 인 entrypoint 가 stdout 안 보낼 수도 — `docker logs <id>` 비교 |
| files browse 가 빈 트리 | 컨테이너 stopped — start 후 재시도 |
| build 진행 중인데 progress 안 보임 | BuildKit 안 켜진 image — `DOCKER_BUILDKIT=1` |
| Dashboard 가 느림 | 컨테이너 수백 개 — 정리 (`docker container prune`) |

## 10. 다음

- ▶ 세부 설정: [[settings]]
- ▶ 컨테이너 debug: [[debug-and-shell]]
- ▶ 자원 모니터링 / 정리: [[resource-usage]]
- ▶ Scout CVE: [[scout]]
- ▶ Compose 통합: [[compose-in-desktop]]
