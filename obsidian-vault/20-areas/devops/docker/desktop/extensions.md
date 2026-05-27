---
title: "Docker Desktop — Extensions 활용"
kind: knowledge
status: current
created_at: 2026-05-22T11:39:00+09:00
tags: [devops, docker, docker-desktop, extensions]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[dashboard]]"
---

# Docker Desktop — Extensions

> Marketplace 에서 설치하거나 자체 작성하는 GUI 패널. 일반적인 Docker 사용을 넘는 추가 기능.

## 1. Marketplace 접근

좌측 사이드바 **Extensions** → **Browse** → 카테고리별 또는 검색.

설치는 클릭 한 번. 권한 확인 후 Docker 가 extension container 를 자동 실행.

## 2. 필수 / 강력 추천 extension

### 2.1 Logs Explorer

여러 컨테이너 log 를 통합 search / filter / 시간 윈도우 limit. 일반 `docker logs` 보다 강력.

**유스케이스**:
- compose stack 의 N 컨테이너 동시 추적
- 특정 단어 (예: "ERROR", "OOM") 필터
- 시간 범위 (예: 최근 1h)

### 2.2 Disk usage

`docker system df` 의 시각화 + 더 세분화.

**보여주는 것**:
- image 별 크기 (layer 공유 고려)
- volume 별 크기 + 마지막 접근
- build cache 의 sub-tree
- container writable layer (CoW)

**1-click 정리**: dangling image / unused volume / build cache 별로.

### 2.3 Volumes Backup & Share

named volume 의 백업 / 복원 / S3 업로드.

```
- Backup volume → tar.gz local
- Restore from tar.gz / S3
- Share to teammate (SFTP / S3)
```

DB volume 백업 시 매우 유용 — `mysqldump` 대신 통째 백업.

### 2.4 Resource Saver

idle 컨테이너 자동 stop. CPU/Mem 사용량 0 임계치 + 시간 윈도우.

> Apple Silicon 의 fan 노이즈 / 배터리 절약에 효과적.

### 2.5 Security Scanners

| extension | 특화 |
| --- | --- |
| **Docker Scout** (내장) | CVE + recommendation + policy 통합 |
| **Snyk** | 의존성 + image + IaC 통합 |
| **Trivy** | OSS — CVE + secret + misconfig |
| **JFrog** | xRay 통합 |

Scout 가 기본 — 추가 비교 / 더 깊은 분석 시 Snyk / Trivy.

### 2.6 Database GUI

| extension | DB |
| --- | --- |
| **Postgres** | PostgreSQL — schema / query / explain |
| **Redis Insight** | Redis — key browser + slow log |
| **MongoDB Compass** | MongoDB — collection / aggregation |
| **MySQL** (3rd-party) | MySQL — workbench 유사 |

compose 로 띄운 DB 컨테이너에 click-through 연결 — 별도 host port 노출 불필요.

### 2.7 VS Code Dev Containers

Containers 화면에서 "Open in VS Code" → Remote Containers attach 자동.

### 2.8 Lens / k9s for Kubernetes

[[kubernetes|Desktop K8s]] 가 활성화된 경우.

- **Lens** — full cluster GUI (workload / network / config / 모니터링)
- **k9s** — terminal UI (vim-like)

## 3. 자체 extension 작성

extension 은 본질적으로 **Docker container** 가 GUI iframe 으로 marshalling.

### 구조

```
my-extension/
├── Dockerfile
├── metadata.json              # 메타 (label, icon, screenshots)
├── docker-compose.yaml        # extension 측 service
└── ui/                        # React/Vue/static — Desktop GUI 안 iframe
    ├── index.html
    └── ...
```

### 최소 metadata.json

```json
{
  "icon": "icon.svg",
  "vm": { "composefile": "docker-compose.yaml" },
  "ui": {
    "dashboard-tab": {
      "title": "My Extension",
      "root": "ui",
      "src": "index.html"
    }
  }
}
```

### Docker Desktop SDK (JS)

```ts
const ddClient = createDockerDesktopClient()

// 도커 명령 실행
const out = await ddClient.docker.cli.exec('ps', ['--format', '{{json .}}'])

// 알림
ddClient.desktopUI.toast.success('Done!')

// 네비게이션
await ddClient.desktopUI.navigate.viewContainer('myContainer')

// 외부 URL 열기
ddClient.host.openExternal('https://...')

// extension VM 내부 service 호출
const res = await ddClient.extension.vm?.service?.get('/api/foo')
```

### 개발 흐름

```bash
# build
docker buildx build -t my/extension:dev .

# install local
docker extension install my/extension:dev

# 개발 모드 (hot reload)
docker extension dev debug my/extension:dev
docker extension dev ui-source my/extension:dev http://localhost:3000

# 제거
docker extension uninstall my/extension:dev
```

### 배포

Docker Hub 에 push + extension 인증 신청 → Marketplace 등재.

## 4. 보안 / 권한

extension 도 **컨테이너** — 모든 격리 규칙 적용. 단:
- `socket: /var/run/docker.sock` 마운트로 host docker 명령 가능
- `host-binaries`, `host-volume` 옵션은 명시적 권한 필요

Settings > Extensions > **Allow only extensions distributed through Docker Marketplace** —
회사 보안 정책 시 권장.

## 5. extension CLI

```bash
docker extension ls                    # 설치된 목록
docker extension install <repo:tag>    # 설치
docker extension uninstall <repo:tag>  # 제거
docker extension update <repo:tag>     # 업데이트
docker extension dev ...               # 개발 모드 (debug / ui-source)
```

## 6. troubleshooting

| 증상 | 해결 |
| --- | --- |
| extension 안 보임 | Settings > Extensions > Enable Docker Extensions |
| 설치 실패 | image manifest 깨짐 — `docker extension install --force` |
| extension log | `docker logs <ext-container-id>` (`docker ps -a` 에 ext-container 보임) |
| UI 새로고침 | Dashboard 좌측 메뉴 우클릭 → Reload |

## 7. 다음

- ▶ Dashboard 활용: [[dashboard]]
- ▶ Dev Environments: [[dev-environments]]
- ▶ 자체 extension 가이드 (공식): https://docs.docker.com/extensions/
