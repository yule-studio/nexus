---
title: "Docker Desktop — Settings 전수 (Resources / Engine / Proxies / 등)"
kind: knowledge
status: current
created_at: 2026-05-22T11:37:00+09:00
tags: [devops, docker, docker-desktop, settings]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[getting-started]]"
  - "[[resource-usage]]"
---

# Docker Desktop — Settings 전수

> Settings (⌘,) 의 모든 패널 + 각 옵션 의미 + 권장값.

## 1. General

| 옵션 | 의미 | 권장 |
| --- | --- | --- |
| Start Docker Desktop when you sign in | 부팅 시 자동 실행 | 끄기 (필요 시만) |
| Open Docker Dashboard at startup | 실행 시 GUI 자동 띄움 | 끄기 |
| Choose theme | Light / Dark / System | System |
| Use the WSL 2 based engine (Windows) | WSL 2 backend (Hyper-V 대신) | 켜기 |
| Send usage statistics | telemetry | 선택 |
| Show CLI hints | 명령어 추천 | 켜기 (학습 시) |
| SBOM indexing on push | image push 시 SBOM 자동 attach | 권장 (Scout 연계) |

## 2. Resources

### 2.1 Advanced (CPU / RAM / Swap / Disk)

| 옵션 | 의미 | 권장 |
| --- | --- | --- |
| CPUs | 컨테이너용 가용 코어 | 가용 / 2 (예: 8 core 머신은 4) |
| Memory | VM RAM | 가용 / 2 (예: 16GB 머신은 8GB) |
| Swap | VM swap | 1~2GB |
| Virtual disk limit | image+volume 디스크 한도 | 100~200GB sparse |
| Disk image location | qcow2 / vmdk 위치 | 외장 SSD 옵션 |

> Apple Silicon: VM 은 **VirtioFS** + **macOS Virtualization framework** 사용 (Settings > General).
> 더 빠른 mount IO.

### 2.2 File sharing (macOS / Windows)

- 호스트의 특정 경로만 컨테이너 bind mount 허용
- default: `/Users`, `/Volumes`, `/tmp`, `/private`
- 추가: `~/Documents/code` 등
- **VirtioFS** (macOS) — 성능 ↑, **gRPC FUSE** — 호환성 ↑

### 2.3 Proxies

회사 네트워크에서 사용.

| 옵션 | 값 예 |
| --- | --- |
| HTTP proxy | `http://proxy.corp:8080` |
| HTTPS proxy | `http://proxy.corp:8080` |
| No proxy | `localhost,127.0.0.1,.corp` |
| Use system proxy | OS 설정 따름 |

CLI 가 `docker pull` 시 본 설정으로 외부 통신.

### 2.4 Network

| 옵션 | 의미 |
| --- | --- |
| Subnet | Docker bridge 의 IP 대역 (default 172.17/16) |
| Docker subnet (advanced) | 사용자 정의 subnet (회사 VPN 과 충돌 회피) |
| DNS | 컨테이너 DNS resolver |

### 2.5 WSL Integration (Windows)

WSL 2 의 어떤 distro 에서 `docker` CLI 활성화할지 선택.

## 3. Docker Engine

`/etc/docker/daemon.json` 동등. JSON 직접 편집.

권장 설정:

```json
{
  "experimental": false,
  "features": { "buildkit": true },
  "registry-mirrors": ["https://mirror.gcr.io"],
  "insecure-registries": ["192.168.1.100:5000"],
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" },
  "default-address-pools": [
    { "base": "172.30.0.0/16", "size": 24 }
  ],
  "builder": {
    "gc": { "enabled": true, "defaultKeepStorage": "20GB" }
  },
  "live-restore": true
}
```

| 키 | 의미 |
| --- | --- |
| `features.buildkit` | BuildKit (modern build engine) 활성 |
| `registry-mirrors` | image pull 가속 mirror |
| `log-driver` / `log-opts` | log 로테이션 (default 무제한 — 디스크 폭주 위험) |
| `default-address-pools` | 사용자 정의 network subnet (회사 VPN 회피) |
| `builder.gc` | 빌드 캐시 GC — 디스크 절약 |
| `live-restore` | engine 재시작 시 컨테이너 keep running |

저장 후 "Apply & restart".

## 4. Kubernetes

| 옵션 | 의미 |
| --- | --- |
| Enable Kubernetes | single-node K8s cluster (kind 동등) |
| Show system containers | `kube-system` 컨테이너 dashboard 표시 |
| Reset Kubernetes Cluster | 모든 K8s 리소스 삭제 + 재초기화 |

자세히 → [[kubernetes]]

## 5. Software Updates

- Auto-update 체크 주기
- **Always update to latest** — public release 즉시
- **Update to monthly stable** — 안정성 우선

업데이트는 engine 재시작 + 컨테이너 stop 동반 — 작업 중일 때 알림.

## 6. Extensions

| 옵션 | 의미 |
| --- | --- |
| Enable Docker Extensions | extension 시스템 활성 |
| Allow only extensions distributed through Docker Marketplace | 보안 (third-party 차단) |
| Show Docker Extensions system containers | extension 자체 컨테이너 표시 |

## 7. Features in development (실험적)

자주 바뀜. 2026-05 시점 주요 토글:

| 옵션 | 의미 |
| --- | --- |
| Use Rosetta for x86_64/amd64 emulation on Apple Silicon | x86 image 빠른 실행 (M1+) |
| Enable Wasm | WebAssembly workload (`--runtime=io.containerd.wasmedge.v1`) |
| Enable host networking on Mac (beta) | `docker run --network=host` 가 실제 host network 사용 (이전엔 VM 한정) |
| Enable Compose Bridge | Compose → Kubernetes manifest 자동 변환 |
| Enable Containerd image store | image 포맷 OCI compliant (multi-platform 통합) |
| Compose v2 support 향상 | watch / sync / depends_on health |

## 8. Security

| 옵션 | 의미 |
| --- | --- |
| Restart Docker Desktop in privileged mode (sign in required) | privilege escalation 옵션 |
| Allow Hub registry access from Docker Desktop only | CLI `docker login` 차단 — GUI 만 |
| Sign on enforcement | 회사 Docker Hub 조직 SSO |

## 9. Advanced (macOS / Linux)

| 옵션 | 의미 |
| --- | --- |
| Allow privileged port mapping | host 1-1023 port `docker run -p 80:8080` 가능 (1024 미만 권장 안 함) |
| Allow the default Docker socket to be used (requires password) | `/var/run/docker.sock` symlink 활성 |
| Allow virtio-fs to use sparse files | 디스크 절약 (macOS) |

## 10. 자주 묻는 설정 — quick recipe

### CPU/메모리 부족
Resources → CPUs 4 / Memory 8GB / Swap 2GB → Apply

### M1 에서 x86 image 빠르게
Features in development → Use Rosetta for x86_64/amd64 emulation → Apply
+ image 빌드/실행 시 `--platform linux/amd64` 명시

### 회사 사설 registry
Docker Engine JSON:
```json
"insecure-registries": ["registry.corp:5000"]
```

### Compose watch 사용
Features in development → Compose v2 → Apply
→ `compose.yaml` 에 `develop.watch` 정의 후 `docker compose watch`

### Build cache 자동 정리
Docker Engine:
```json
"builder": { "gc": { "enabled": true, "defaultKeepStorage": "10GB" } }
```

## 11. 다음

- ▶ 첫 실행: [[getting-started]]
- ▶ K8s 활성화: [[kubernetes]]
- ▶ 자원 모니터링 / 정리: [[resource-usage]]
- ▶ Build / BuildKit 깊이: [[builder-and-buildkit]]
- ▶ Network / Proxy: [[networking-and-proxy]]
