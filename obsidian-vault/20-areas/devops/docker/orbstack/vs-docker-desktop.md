---
title: "OrbStack vs Docker Desktop — 비교"
kind: knowledge
status: current
created_at: 2026-05-22T12:32:00+09:00
tags: [devops, docker, orbstack, comparison]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[../desktop/getting-started]]"
---

# OrbStack vs Docker Desktop

> 2026-05 기준. **Mac 사용자만 OrbStack 선택지** — Docker Desktop 은 Mac/Win/Linux 다 지원.
> 핵심 차이: OrbStack 는 메모리 / 시작 속도 / 디스크 효율 / VirtioFS 성능 우세, Docker Desktop 은
> 다양한 통합 (Scout / Extensions / Dev Environments / Build Cloud) 우세.

## 1. 한 장 비교

| 항목 | Docker Desktop | OrbStack |
| --- | --- | --- |
| **플랫폼** | macOS / Windows / Linux | macOS only |
| **라이선스** | Personal free / Pro / Team / Business | Personal free / Pro / Business |
| **첫 시작 시간** | 30~60초 | **5초** |
| **idle RAM** | 2~4GB | **200~400MB** (dynamic) |
| **disk image** | 64GB sparse (default) | dynamic, 사용한 만큼 |
| **VM backend** | Linux VM (VZ / HyperKit) | **Apple Virtualization framework** + VirtioFS |
| **file sync** | VirtioFS / gRPC FUSE | **VirtioFS** (M1 native, 매우 빠름) |
| **x86 emulation on M1** | Rosetta (옵션) 또는 QEMU | **Rosetta** native |
| **Kubernetes** | single-node (kubelet/etcd) | single-node (K3s) |
| **Linux VM 다중** | ✗ | ✓ (Ubuntu / Debian / Arch / Fedora / Alpine / NixOS) |
| **Spotlight / Finder 통합** | 제한적 | ✓ |
| **VS Code Remote** | Dev Containers extension | ✓ (자동) |
| **Docker Extensions / Marketplace** | ✓ 풍부 | ✗ |
| **Docker Scout (CVE)** | ✓ 통합 | ✗ (외부 도구 OK: Trivy / Snyk) |
| **Docker Hub 통합** | ✓ | 부분 (CLI 가능) |
| **Build Cloud** | ✓ | ✗ |
| **Dev Environments** | ✓ | ✗ |
| **버그 / 안정성** | 안정 (10+ 년) | 안정 (2+ 년, 빠르게 개선) |
| **회사 지원 / 영업** | Docker Inc | OrbStack Inc (작은 팀) |

## 2. 무엇을 고를까

### OrbStack 가 좋은 경우
- **Apple Silicon Mac** — 메모리 / 속도 차이 매우 큼
- **Linux VM 자주 사용** — 빠른 spawn / 직접 ssh / Finder 접근
- **간소함 선호** — UI 가 단순, 토글 적음
- **idle 시 자원 절약** — 노트북 배터리 / 다른 앱 (IDE / Brave / Slack) 자원 양보

### Docker Desktop 가 좋은 경우
- **Windows / Linux** — OrbStack 안 됨
- **Docker Scout 필수** — CVE / SBOM / policy 통합
- **회사 Docker organization** — SSO / 정책 enforcement
- **Extensions ecosystem** — Logs Explorer / Disk Usage / Volumes Backup 등 GUI 도구 다수
- **Dev Environments / Build Cloud** — 팀 share / cloud builder
- **공식 지원** — 회사 SLA / 영업

## 3. 성능 벤치 (M1 Pro 기준 — 비공식)

| 작업 | Docker Desktop | OrbStack |
| --- | --- | --- |
| 첫 시작 | ~45초 | ~5초 |
| `docker run hello-world` (cold) | ~3초 | ~0.5초 |
| idle 30분 후 RAM | 2.5GB | 250MB |
| bind mount 100MB file write | 2.5초 | 0.4초 |
| amd64 image build (Rosetta) | 2분 | 1분 |
| K8s pod 시작 | ~15초 | ~5초 |

> 출처: 커뮤니티 벤치마크. 본인 환경에서 직접 측정 권장.

## 4. 마이그레이션 (Desktop → OrbStack)

OrbStack 첫 실행 시 자동 import 옵션:
- container / image / volume 통째
- docker context 추가
- compose project 그대로 사용

수동:
```bash
# 1. Desktop 에서 image 저장
docker save myapp:v1 > /tmp/myapp.tar

# 2. Desktop 종료 + OrbStack 시작 + context 전환
osascript -e 'quit app "Docker"'
open -a OrbStack
docker context use orbstack

# 3. 새 환경에 load
docker load < /tmp/myapp.tar
```

volume:
```bash
# Desktop 측 volume backup (volume 백업 extension 또는)
docker run --rm -v mydb:/data -v $PWD:/backup alpine tar czf /backup/db.tar.gz /data
# OrbStack 에서 restore
docker run --rm -v mydb:/data -v $PWD:/backup alpine tar xzf /backup/db.tar.gz -C /
```

## 5. 함께 쓰기

가능 — 동시 실행 + context 로 전환:
```bash
docker context use desktop-linux  # Docker Desktop
docker context use orbstack       # OrbStack
```

단점: 두 VM 이 같이 RAM 점유 → 권장 안 함.

## 6. 다음

- ▶ OrbStack 시작: [[getting-started]]
- ▶ OrbStack 으로 Docker 사용: [[docker-engine]]
- ▶ Linux VM 활용: [[linux-machines]]
- ▶ Docker Desktop 가이드: [[../desktop/getting-started]]

## 참고

- 공식 비교: https://docs.orbstack.dev/compare/docker-desktop
- 커뮤니티 리뷰: HN / Reddit r/macOS
