---
title: "Docker Desktop — 설치 / 첫 실행"
kind: knowledge
status: current
created_at: 2026-05-22T11:35:00+09:00
tags: [devops, docker, docker-desktop, install]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[dashboard]]"
  - "[[settings]]"
  - "[[../orbstack/vs-docker-desktop]]"
---

# Docker Desktop — 설치 / 첫 실행

> macOS / Windows / Linux 사용자가 가장 빠르게 Docker 환경 셋업하는 방법. CLI 만 필요하면
> Docker Engine (Linux) / Colima / OrbStack 도 가능 — 비교는 [[../orbstack/vs-docker-desktop]].

## 1. 시스템 요구사항

| 플랫폼 | 요구사항 |
| --- | --- |
| **macOS** | macOS 13+ (Ventura 이상). Apple Silicon (M1/M2/M3) 또는 Intel. RAM 4GB 이상 (8GB 권장) |
| **Windows** | Windows 11 64-bit (Home/Pro/Enterprise/Education). WSL 2 활성. RAM 4GB 이상. BIOS Virtualization 활성 |
| **Linux** | x86_64 또는 aarch64. systemd. KVM virtualization. Ubuntu 22.04+, Debian 12+, Fedora 39+ 지원 |

> Apple Silicon Mac 은 **Rosetta 2** 필요 (x86_64 image emulation). `softwareupdate --install-rosetta`.

## 2. 설치

### macOS

```bash
# Homebrew (권장)
brew install --cask docker

# 또는 dmg 직접 다운로드
# https://docs.docker.com/desktop/install/mac-install/
```

설치 후 `/Applications/Docker.app` 실행 → 첫 실행 시 권한 요청 (privileged helper) → 환영 화면.

### Windows

1. https://docs.docker.com/desktop/install/windows-install/ 에서 `Docker Desktop Installer.exe` 다운로드
2. 더블 클릭 → "Use WSL 2 instead of Hyper-V" 체크 (default)
3. 재부팅 후 자동 실행
4. WSL 2 backend 활성 확인 — `wsl --status` 가 Version 2 표시

### Linux

```bash
# Ubuntu 예
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-desktop
systemctl --user start docker-desktop
```

Linux 는 일반적으로 Docker Engine (CLI only) 만 사용 — Desktop GUI 는 선택.

## 3. 첫 실행 흐름

1. **로그인 / 계정 생성** (선택) — Docker Hub 계정 — image pull rate limit 완화 + Scout / Build Cloud / Dev Environments 통합
2. **튜토리얼 건너뛰기 또는 진행** — "Quick start guide" 가 hello-world 실행
3. **Dashboard 진입** — Containers / Images / Volumes / Builds / Dev Environments / Extensions
4. **smoke check**:
   ```bash
   docker --version              # Docker version 24.x ...
   docker compose version        # Docker Compose version v2.x ...
   docker run --rm hello-world   # "Hello from Docker!" 출력
   ```

## 4. 시스템 자원 (default)

| 자원 | macOS default | Windows default |
| --- | --- | --- |
| CPU | 2 cores | 사용 가능 코어 50% |
| RAM | 2GB (M1+) / 8GB | 사용 가능 RAM 50% |
| Swap | 1GB | 시스템 관리 |
| Disk image | 64GB sparse | 64GB sparse |

> 변경: Settings > Resources. 자세히 [[settings#resources]].

## 5. 데이터 위치

| 플랫폼 | 위치 |
| --- | --- |
| macOS | `~/Library/Containers/com.docker.docker/Data/vms/0/` (VM disk image) |
| Windows | `%LOCALAPPDATA%\Docker\wsl\` (WSL2 distro: `docker-desktop`, `docker-desktop-data`) |
| Linux | `~/.docker/desktop/vms/0/` |

> 디스크 정리 / 이전 / 백업 시 위 위치. 자세히 [[resource-usage#disk]].

## 6. 첫 화면 — Dashboard 한 줄 요약

| 메뉴 | 뭐 함 |
| --- | --- |
| Containers | 실행 중 / 정지 컨테이너 목록 + GUI 로 stop/start/exec/logs |
| Images | local image 목록 + run / pull / scout |
| Volumes | named volume / bind 경로 + 사용 컨테이너 |
| Builds | 진행 중 / 완료 build (buildx) — log 추적 |
| Dev Environments | clone 한 dev env workspace (VS Code Remote 통합) |
| Extensions | Marketplace + 설치된 extension |

자세히 → [[dashboard]]

## 7. 인증 / 로그인

- Sign in → Docker Hub 계정 (또는 GitHub / Google SSO)
- 로그인 후 효과:
  - **image pull rate limit**: anonymous 100/6h → authenticated 200/6h (Pro 무제한)
  - **private repo** 접근
  - **Docker Scout**: 무료 tier (월 한도)
  - **Build Cloud**: cloud builder (옵션 유료)
  - **Dev Environments**: clone share
  - **organization** seat (회사 계정)

## 8. 자동 시작 / 종료

- macOS: 메뉴바 🐳 → Settings → **Start Docker Desktop when you sign in to your computer**
- Windows: General → Start Docker Desktop when you log in
- 종료: `docker context use default && Docker Desktop > Quit` (또는 메뉴바)

CLI 통합 (모든 플랫폼):
```bash
# 시작
open -a Docker            # macOS
# Windows: PowerShell 에서 'Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"'

# 종료
osascript -e 'quit app "Docker"'   # macOS
```

CLI 자동화는 [[cli-integration]] 참조.

## 9. 첫 troubleshooting

| 증상 | 해결 |
| --- | --- |
| `Cannot connect to the Docker daemon` | Docker Desktop 실행 중인지 확인 (메뉴바 아이콘) |
| Apple Silicon 에서 x86 image 느림 | Settings > Features in development > **Use Rosetta for x86_64/amd64 emulation** 켜기 |
| WSL 2 backend 실패 (Windows) | `wsl --update` + Docker Desktop 재시작 |
| 디스크 용량 부족 | Settings > Resources > Disk image size 확장, 또는 `docker system prune -a --volumes` |
| 호스트 폴더 접근 거부 (macOS) | Settings > Resources > File sharing 에 경로 추가 |

자세히 → [[settings]], [[resource-usage]]

## 10. 다음에 볼 것

- ▶ **Dashboard 활용**: [[dashboard]]
- ▶ **세부 설정**: [[settings]]
- ▶ **Kubernetes 켜기**: [[kubernetes]]
- ▶ **OrbStack 과 비교**: [[../orbstack/vs-docker-desktop]]
- ▶ **첫 image build**: [[../practice/02-dockerfile-spring-boot]]
