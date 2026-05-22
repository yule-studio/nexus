---
title: "OrbStack — 설치 / 첫 실행"
kind: knowledge
status: current
created_at: 2026-05-22T12:30:00+09:00
tags: [devops, docker, orbstack, install, mac]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[vs-docker-desktop]]"
---

# OrbStack — 설치 / 첫 실행

> **macOS 한정 Docker / Kubernetes / Linux VM 통합 도구**. Docker Desktop 대체 — 가볍고 빠름.
> Apple Silicon native + Rosetta x86 emulation. Personal free / Commercial paid.

## 1. 무엇인가

OrbStack 은 macOS 에서:
- **Docker engine** (CLI + GUI dashboard)
- **Linux VM** (Ubuntu / Debian / Arch / Fedora / Alpine / NixOS — 다중)
- **Kubernetes** (single-node K3s 기반)

을 한 앱 + 작은 footprint 로 통합 제공. Apple Silicon 에 최적화 — Apple Virtualization framework
+ VirtioFS + Rosetta 2 (x86_64 emulation).

영상: [OrbStack — Docker Desktop killer? (Fireship 스타일 리뷰)](https://www.youtube.com/results?search_query=orbstack+vs+docker+desktop)

## 2. 시스템 요구사항

| 항목 | 요구 |
| --- | --- |
| OS | macOS 13+ (Ventura). 14+ 권장 |
| CPU | Apple Silicon (M1/M2/M3) 또는 Intel x86_64 |
| RAM | 4GB+ (8GB 권장) |
| Disk | 8GB+ (image / VM 따라 가변) |
| License | Personal free / Commercial / Enterprise — https://orbstack.dev/pricing |

> Linux / Windows 지원 없음 — Mac 전용.

## 3. 설치

### Homebrew (권장)
```bash
brew install --cask orbstack
```

### dmg 직접
https://orbstack.dev/download 에서 다운로드 → `/Applications` 으로 드래그.

### 첫 실행
1. **권한 요청** — privileged helper 설치 (필수, 한 번만)
2. **시스템 통합** — Finder / Spotlight / Files app 통합 옵션
3. **마이그레이션** — Docker Desktop 있으면 image / volume / container 자동 import 옵션
4. **welcome 화면** — Docker / Kubernetes / Linux Machine 중 시작점 선택

## 4. smoke check

```bash
# OrbStack 가 docker socket / kubectl context 자동 설정
docker --version
# Docker version 27.x ...

docker info | grep -i orbstack
# Operating System: OrbStack

docker run --rm hello-world
# "Hello from Docker!" — 즉시 (Docker Desktop 보다 시작 빠름)
```

GUI:
- 메뉴바 🐳 (OrbStack 아이콘) → Dashboard
- 좌측: Containers / Images / Volumes / Networks / Builds / Kubernetes / Linux Machines

## 5. 메뉴 / 데이터 위치

| 위치 | 용도 |
| --- | --- |
| `~/.orbstack/` | 설정 / 메타 |
| `~/OrbStack/` (Finder) | Linux machine 파일 시스템 직접 access |
| `~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data` | image / volume / VM disk |
| `/Applications/OrbStack.app` | 본체 |

> Linux machine 의 파일은 `~/OrbStack/<machine-name>/` 에 Mac Finder 로 그대로 접근. 양방향 sync.

## 6. Docker Desktop 과 충돌 회피

둘 다 설치된 상태:
- 동시 실행 시 docker socket 충돌
- **OrbStack 가 default 로 잡힘** — `docker context use orbstack` 자동
- Desktop 으로 전환: `docker context use desktop-linux`
- 영구 분리: Desktop 종료 + OrbStack 만 실행

```bash
docker context ls
# NAME           DESCRIPTION
# default*       Default
# desktop-linux  Docker Desktop
# orbstack       OrbStack (default for orbstack)
```

## 7. 자동 시작 / 종료

- Settings > General > **Launch at login** 토글
- 메뉴바 → Quit / Restart

CLI:
```bash
# 시작
open -a OrbStack

# 종료
osascript -e 'quit app "OrbStack"'
# 또는
orb stop
```

## 8. 라이선스

| Plan | 가격 (2026) | 사용 가능 |
| --- | --- | --- |
| Personal | 무료 | 개인 / 학생 / 비영리 / 비상업 |
| Pro | $8/월 | 상업 사용 + Pro 기능 |
| Business / Enterprise | per-seat | SSO / audit / large team |

> 회사 업무는 Pro+. License 모니터링이 dashboard 에서 표시.

## 9. 다음

- ▶ **Docker Desktop 과 비교**: [[vs-docker-desktop]]
- ▶ **Docker 사용**: [[docker-engine]]
- ▶ **Linux VM 띄우기**: [[linux-machines]]
- ▶ **Kubernetes**: [[k8s]]
- ▶ **자원 / 설정**: [[settings-and-resources]]
- ▶ **CLI**: [[cli]]

## 참고

- 공식: https://orbstack.dev/
- Docs: https://docs.orbstack.dev/
- Twitter / X: [@OrbStack](https://twitter.com/OrbStack)
- GitHub discussions: https://github.com/orbstack/orbstack/discussions
