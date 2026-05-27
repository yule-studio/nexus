---
title: "OrbStack — Linux Machines (VM 활용)"
kind: knowledge
status: current
created_at: 2026-05-22T12:36:00+09:00
tags: [devops, docker, orbstack, linux-vm]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[cli]]"
  - "[[integration]]"
---

# OrbStack — Linux Machines

> OrbStack 의 강점 중 하나 — Docker 외에 **다양한 Linux distribution 의 full VM** 을 5초 안에
> spawn. Multipass / Lima / UTM 대체. Finder 통합 + Rosetta + Mac CLI 직접 사용.

## 1. 무엇

OrbStack 가 distro image 를 가져와서 **container 가 아닌 full VM** 으로 띄움. 안에서 systemd /
apt / yum / SSH / Docker 등 다 동작. Mac 에서 native 처럼 보임:
- `~/OrbStack/<machine>/` 에 Finder 로 접근
- `<machine>.orb.local` 자동 도메인
- `orb shell <machine>` 으로 즉시 진입
- Mac CLI (`brew install ...`) 마치 머신 안에서 한 것처럼 sync

지원 distro: **Ubuntu / Debian / Arch / Fedora / Alpine / CentOS Stream / Rocky / Kali / NixOS** 등 15+.

## 2. 생성

### GUI (Dashboard)

좌측 **Linux Machines** → **Create** → distro 선택 → 이름 입력 → Create.

### CLI

```bash
# Ubuntu 24.04
orb create ubuntu my-ubuntu

# Debian 12 + arm64
orb create -a arm64 debian my-debian

# Alpine (가장 가벼움)
orb create alpine my-alpine

# 다른 release
orb create ubuntu:22.04 my-old
```

옵션:
- `-a amd64 / arm64` — arch 강제 (default native)
- `-u user` — 초기 user (default `root`)
- `-p password` — sudo password

## 3. 진입

```bash
# shell 직접
orb shell my-ubuntu
# 또는
orb -m my-ubuntu

# 특정 명령 실행
orb -m my-ubuntu -- apt update
orb -m my-ubuntu -- whoami

# SSH (정식 — VSCode Remote 등이 사용)
ssh my-ubuntu@orb           # auto config (~/.ssh/config 자동)

# 또는 기본 user
ssh my-ubuntu
```

## 4. 파일 sync — Finder + CLI 양방향

| 위치 | 동작 |
| --- | --- |
| `~/OrbStack/my-ubuntu/` (Mac) | VM 의 root 그대로 — Finder 로 browse / 편집 |
| `/mnt/mac/` (VM 안) | Mac 의 home directory 그대로 access |
| `/mnt/mac/Users/masterway/code/` | Mac 의 code 폴더 즉시 |

양방향 sync — VirtioFS 기반 — Mac 에서 파일 변경 → VM 즉시 반영 (vice versa).

```bash
# Mac 에서 VM 의 file 편집
code ~/OrbStack/my-ubuntu/etc/nginx/nginx.conf

# VM 안에서 Mac 의 file 사용
orb -m my-ubuntu
cd /mnt/mac/Users/masterway/code/myapp
ls
```

## 5. networking — orb.local 자동 도메인

각 machine 이 `<name>.orb.local` 도메인 자동 부여:

```bash
# VM 안에서 nginx
orb -m my-ubuntu
sudo apt install -y nginx
sudo systemctl start nginx

# Mac 에서 직접 접근
curl http://my-ubuntu.orb.local
# nginx welcome page
```

port forwarding 자동 — VM 의 모든 listen port 가 Mac 에서 자동 노출. 별도 `-p 80:80` 설정 불필요.

## 6. 시스템 통합

| 기능 | 동작 |
| --- | --- |
| **Spotlight** | "my-ubuntu" 검색 → 머신 진입 |
| **Finder** | sidebar 에 OrbStack machines 표시 |
| **VS Code Remote-SSH** | `ssh my-ubuntu` config 자동 — Remote 즉시 |
| **brew** (Mac) | VM 에 sync 안 됨 — VM 안에서는 apt/yum/pacman |
| **Mac CLI 명령** | VM 안에서 `mac` 명령으로 호출 가능 — 예 `mac open /Applications/Safari.app` |

## 7. systemd / service

```bash
orb -m my-ubuntu
sudo systemctl status                  # systemd 동작 (다른 VM 도구와 차이)
sudo apt install -y nginx mysql-server
sudo systemctl enable --now mysql
```

> Docker container 와 차이 — full init system / multiple service / cron / SSH 등.

## 8. Mac ↔ Linux clipboard / open

```bash
# VM 안에서 Mac 에 알림
orb -m my-ubuntu
notify "hello"                         # Mac 알림 센터

# Mac 에서 VM 의 파일 열기 (Finder)
open ~/OrbStack/my-ubuntu/etc/
```

## 9. snapshot / clone

```bash
# 현재 상태 snapshot
orb snapshot create my-ubuntu before-experiment

# rollback
orb snapshot restore my-ubuntu before-experiment

# clone
orb clone my-ubuntu my-ubuntu-2
```

> 학습 / 실험 시 매우 유용 — 망쳐도 rollback.

## 10. delete

```bash
orb delete my-ubuntu              # 영구 삭제
orb delete my-ubuntu --keep-data  # disk 보존 (재생성 시 같은 data)
```

## 11. 활용 시나리오

### 11.1 다른 distro 에서 Docker 사용

OrbStack 의 default Docker 는 자체 VM. **추가**로 Ubuntu 안에 Docker 설치 가능:

```bash
orb create ubuntu my-docker-host
orb -m my-docker-host
sudo apt update && sudo apt install -y docker.io
sudo docker run hello-world
```

용도: 회사 환경 (Ubuntu 22.04 + 특정 docker version) 재현.

### 11.2 다양한 Linux 환경 테스트

```bash
orb create alpine alpine-test         # 가벼움
orb create arch arch-test             # bleeding edge
orb create nixos nixos-test           # NixOS
# ...
```

### 11.3 VPN / network 격리

VM 마다 별도 IP / DNS — 회사 VPN / 로컬 dev / 격리 환경 분리.

### 11.4 K8s 다중 cluster

```bash
orb create ubuntu k8s-1
orb -m k8s-1 -- curl -sfL https://get.k3s.io | sh -

orb create ubuntu k8s-2
orb -m k8s-2 -- curl -sfL https://get.k3s.io | sh -

# 두 cluster 동시 운영 (다른 kubectl context)
```

## 12. Multipass / Lima / UTM 와 비교

| 항목 | OrbStack | Multipass | Lima | UTM |
| --- | --- | --- | --- | --- |
| spawn 속도 | ★★★★★ 5초 | ★★ 30초 | ★★★ 15초 | ★ 1분+ |
| distro 수 | 15+ | 4 (Ubuntu 위주) | 다수 | 모든 OS |
| Mac 통합 | ★★★★★ | ★★ | ★★★ | ★ |
| Windows support | ✗ | ✓ | ✓ (Linux on Mac/Win) | ✓ |
| 무료 | personal | 무료 | 무료 | 무료 |
| Docker 통합 | ✓ 같은 앱 | ✗ | colima 별도 | ✗ |

## 13. troubleshooting

| 증상 | 해결 |
| --- | --- |
| `orb shell` 실패 | machine stopped — `orb start my-ubuntu` |
| Finder 에서 안 보임 | OrbStack Settings > General > Finder integration 활성 |
| SSH config 없음 | OrbStack 가 `~/.ssh/config` 의 `Include orbstack` 추가 — 확인 |
| disk 사용 폭주 | snapshot 정리 + `orb delete --keep-data` 후 재생성 |
| network 격리 안 됨 | Settings > Network > 각 machine 의 NAT mode |

## 14. 다음

- ▶ CLI 전수: [[cli]]
- ▶ Networking: [[networking]]
- ▶ K8s: [[k8s]]
- ▶ VS Code / Finder 통합: [[integration]]

## 참고

- 공식: https://docs.orbstack.dev/machines/
- Multipass: https://multipass.run/
- Lima: https://lima-vm.io/
