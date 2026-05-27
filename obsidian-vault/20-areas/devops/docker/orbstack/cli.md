---
title: "OrbStack — orb / orbctl CLI 전수"
kind: knowledge
status: current
created_at: 2026-05-22T12:42:00+09:00
tags: [devops, docker, orbstack, cli]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[linux-machines]]"
  - "[[settings-and-resources]]"
---

# OrbStack CLI — orb / orbctl

> `orb` (사용자 명령) + `orbctl` (서비스 제어). Dashboard 의 모든 기능을 CLI 로.

## 1. 기본

| 명령 | 효과 |
| --- | --- |
| `orb` | help |
| `orb status` | 전체 상태 (VM / Docker / K8s 시작 여부, RAM / Disk) |
| `orb start` | OrbStack engine 시작 |
| `orb stop` | engine 종료 |
| `orb restart` | engine 재시작 |
| `orb quit` | 앱 완전 종료 |

## 2. machine 관리

```bash
# 생성
orb create ubuntu my-ubuntu                # Ubuntu 최신 LTS
orb create -a amd64 ubuntu:22.04 amd64-test
orb create alpine alpine-test
orb create -u myuser -p mypass debian dev-debian

# 목록
orb list
# 또는
orb list -v                                # verbose (IP / status)

# 시작 / 정지 / 재시작
orb start my-ubuntu
orb stop my-ubuntu
orb restart my-ubuntu

# 진입
orb shell my-ubuntu                        # bash / zsh 자동
orb -m my-ubuntu                           # 단축
orb -m my-ubuntu -- whoami                 # 한 명령
orb -m my-ubuntu -- apt update             # 여러 명령은 quotation

# 삭제
orb delete my-ubuntu
orb delete --keep-data my-ubuntu           # data 보존
```

## 3. snapshot

```bash
orb snapshot create my-ubuntu before-experiment
orb snapshot list my-ubuntu
orb snapshot restore my-ubuntu before-experiment
orb snapshot delete my-ubuntu before-experiment
```

## 4. clone

```bash
orb clone my-ubuntu my-ubuntu-2
# 또는 다른 distro 로 base
orb clone --distro debian my-ubuntu my-debian-from-ubuntu
```

## 5. Docker

OrbStack 가 docker context 자동 설정 — 일반 `docker` 명령 그대로.

```bash
docker context ls
docker run hello-world

# OrbStack 의 docker engine config 편집 (Settings GUI 동등)
orb config docker
# 또는 직접
vim ~/.orbstack/config/docker.json
orb restart
```

## 6. Kubernetes

```bash
# K8s 시작 / 정지
orb start k8s
orb stop k8s

# K8s 안의 모든 namespace pod
kubectl get pods -A

# K8s reset
orb reset k8s
```

## 7. SSH / 직접 접근

OrbStack 자동 `~/.ssh/config` 추가 (`Include ~/.orbstack/ssh/config`).

```bash
ssh my-ubuntu                              # 기본 user
ssh root@my-ubuntu                         # root
ssh -p 22 my-ubuntu                        # default port

# VS Code Remote-SSH
code --remote ssh-remote+my-ubuntu /home/myuser
```

config 직접:
```bash
cat ~/.orbstack/ssh/config
# Host my-ubuntu
#   User myuser
#   ProxyCommand /Applications/OrbStack.app/Contents/MacOS/orb ssh-stdio my-ubuntu
```

## 8. 파일 sync 확인

```bash
# Mac → VM
echo "from mac" > ~/OrbStack/my-ubuntu/tmp/test.txt
orb -m my-ubuntu -- cat /tmp/test.txt
# from mac

# VM → Mac
orb -m my-ubuntu -- bash -c 'echo "from vm" > /tmp/test2.txt'
cat ~/OrbStack/my-ubuntu/tmp/test2.txt
# from vm
```

## 9. log / 디버깅

```bash
# 전체 OrbStack log
orb logs

# 특정 machine log
orb logs my-ubuntu

# K8s log
orb logs k8s

# diagnostic 패키지 (지원 문의 용)
orb diagnostic export ~/Downloads/orb-diag.tgz
```

## 10. config

```bash
# 전체 보기
orb config get

# 특정 키
orb config get network.subnet

# 변경 (재시작 필요할 수도)
orb config set network.mtu 1450
orb config set docker.engine.registry-mirrors '["https://mirror.gcr.io"]'

# 재시작
orb restart
```

config 파일: `~/.orbstack/config/orbstack.conf` (TOML)

## 11. 자주 쓰는 alias / 함수

`.zshrc` / `.bashrc`:

```bash
# 빠른 machine 생성 + 진입
function orbnew() {
  orb create ubuntu "$1" && orb shell "$1"
}

# Mac 의 현재 폴더를 VM 안에서 열기
function orbcd() {
  cd "/mnt/mac$(pwd)"      # VM 안에서 사용
}

# port forward 확인
alias orbports='orb list -v'
```

## 12. 환경 변수 / 통합 명령

```bash
# Mac path 와 VM path 변환
orb path-convert /Users/masterway/code   # → /mnt/mac/Users/masterway/code
orb path-convert -r /mnt/mac/foo         # → /Users/masterway/foo

# Mac 명령 호출 (VM 안에서)
orb -m my-ubuntu
# 안에서:
mac open ~/Documents                     # Mac Finder 열기
mac pbcopy < /etc/hostname               # Mac clipboard 로 복사
mac say "build done"                     # Mac 알림
```

## 13. orbctl (서비스 제어 — 고급)

`orbctl` 은 OrbStack daemon / hypervisor 직접 제어. 일반 사용 시 `orb` 면 충분.

```bash
orbctl status
orbctl service list
orbctl service restart <name>
```

## 14. 자주 쓰는 cheat sheet

| 의도 | 명령 |
| --- | --- |
| 우분투 머신 빠르게 | `orb create ubuntu u && orb -m u` |
| 머신 안 명령 한 줄 | `orb -m u -- apt install -y curl` |
| 머신 reset | `orb delete u && orb create ubuntu u` |
| docker context 확인 | `docker context show` |
| K8s 시작 | `orb start k8s` |
| disk 사용 | `orb status` 또는 dashboard |
| log dump | `orb logs --since 1h` |

## 15. 다음

- ▶ Linux Machines: [[linux-machines]]
- ▶ Networking: [[networking]]
- ▶ K8s: [[k8s]]
- ▶ Integration (VS Code 등): [[integration]]

## 참고

- 공식 CLI ref: https://docs.orbstack.dev/cli
