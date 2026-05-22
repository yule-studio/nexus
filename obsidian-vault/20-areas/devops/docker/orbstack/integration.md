---
title: "OrbStack — Mac 통합 (Finder / Spotlight / VS Code / SSH / 1Password)"
kind: knowledge
status: current
created_at: 2026-05-22T12:46:00+09:00
tags: [devops, docker, orbstack, integration, macos]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[linux-machines]]"
  - "[[networking]]"
---

# OrbStack — Mac 통합

> OrbStack 의 진가는 통합. Finder / Spotlight / VS Code Remote / SSH agent / 1Password 가
> "VM 인지 의식 안 하게" 동작. 다른 도구 (Multipass / Lima) 에 비해 강점.

## 1. Finder 통합

### sidebar
좌측 sidebar 에 **OrbStack** 항목 — 각 Linux machine 의 root 가 폴더로 보임.

```
Finder sidebar
└── OrbStack
    ├── my-ubuntu/   ← / (VM root)
    │   ├── etc/
    │   ├── home/
    │   ├── var/
    │   └── ...
    └── my-debian/
        └── ...
```

Finder 에서:
- 파일 더블 클릭 → Mac app 으로 열기 (VS Code / Preview / TextEdit 등)
- 드래그 앤 드롭으로 Mac ↔ VM 복사 (양방향)
- 우클릭 → "Open in Terminal" → VM shell 진입

### activation
Settings > General > **Add machines to Finder sidebar** 토글.

수동 mount:
```bash
ls ~/OrbStack/                       # 모든 machine 의 root
ls ~/OrbStack/my-ubuntu/home/        # 특정 machine
```

## 2. Spotlight 통합

⌘Space → "my-ubuntu" 검색 → Enter → terminal 자동 진입.

또는 "OrbStack" → dashboard 열기.

activation: Settings > Integration > **Spotlight**.

## 3. VS Code Remote-SSH

OrbStack 가 SSH config 자동 추가 — Remote-SSH extension 이 즉시 인식.

```bash
# 1. VS Code 의 Command Palette (⌘⇧P) → "Remote-SSH: Connect to Host"
# 2. 목록에서 my-ubuntu 선택
# 3. VS Code 가 새 창에서 VM 에 attach
# 4. VM 의 file 직접 편집 + terminal + extension 모두 VM 안에서 실행
```

또는 CLI:
```bash
code --remote ssh-remote+my-ubuntu /home/myuser
```

### Dev Container

VM 안의 `devcontainer.json` 자동 인식:
```bash
orb -m my-ubuntu
cd /home/myuser/myproject
# VS Code 가 .devcontainer/devcontainer.json 발견 시
# "Reopen in Container" 자동 prompt
```

VS Code 의 Dev Containers extension + OrbStack 의 Docker engine 통합.

## 4. SSH agent 통합

Mac 의 ssh-agent (1Password / Keychain / 직접 add) 가 VM 안에서 동작:

```bash
# Mac 에서 키 추가
ssh-add ~/.ssh/id_ed25519

# VM 안에서 동작
orb -m my-ubuntu
ssh git@github.com                 # Mac 의 agent forward 작동
git clone git@github.com:org/repo  # OK
```

ForwardAgent 가 OrbStack SSH config 에 자동 활성.

## 5. 1Password 통합

1Password 8+ 에 SSH agent 활성한 경우:

```bash
# Mac 1Password 의 SSH key 가
# OrbStack 안에서도 사용 가능
orb -m my-ubuntu
ssh git@github.com                  # 1Password 가 Touch ID 프롬프트
```

설정:
1. 1Password Settings > Developer > **Use the SSH agent** 토글
2. 1Password 에 SSH key 저장
3. Mac shell config 에 `SSH_AUTH_SOCK` 설정 (1Password 가 자동 안내)
4. OrbStack 의 ForwardAgent 자동 작동

## 6. Mac CLI ↔ VM

### VM 에서 Mac 명령 호출

```bash
orb -m my-ubuntu

# Mac 앱 / 시스템 명령
mac open ~/Documents              # Finder
mac open https://google.com       # 기본 브라우저
mac pbcopy < /etc/hostname        # clipboard 복사
mac pbpaste                       # clipboard 붙여넣기
mac say "build complete"          # 음성 알림
mac notify "build complete"       # 알림 센터
mac path-of-mac /home/myuser/code # VM path → Mac path
```

### Mac 에서 VM 명령 호출

```bash
# bash function
function vmrun() {
  orb -m "$1" -- "${@:2}"
}
vmrun my-ubuntu apt list --installed
```

## 7. path translation

Mac 과 VM 의 path 가 다름 — OrbStack 가 자동 변환 도구 제공:

```bash
# Mac path → VM path
orb path-convert /Users/masterway/code/myapp
# /mnt/mac/Users/masterway/code/myapp

# VM path → Mac path
orb path-convert -r /mnt/mac/Users/masterway/code/myapp
# /Users/masterway/code/myapp
```

활용: shell script 에서 Mac/VM 양쪽 동작:
```bash
#!/bin/bash
SRC=$(orb path-convert "$PWD")
orb -m my-ubuntu -- bash -c "cd $SRC && make build"
```

## 8. Docker context 통합

```bash
docker context ls
# default        — Mac native (없음)
# orbstack*      — OrbStack docker engine
# desktop-linux  — Docker Desktop (있으면)
```

전환:
```bash
docker context use orbstack
docker context use desktop-linux
```

## 9. K8s context 통합

```bash
kubectl config get-contexts
# *  orbstack          orbstack
#    docker-desktop    docker-desktop
#    arn:aws:eks:...   (회사 EKS)
```

OrbStack 가 cluster 시작 시 자동 등록 / 삭제.

## 10. iTerm / Terminal.app 통합

OrbStack 의 Linux machine 진입 시 어떤 terminal 사용할지:

Settings > General > **Default terminal** → iTerm / Terminal / Warp / Wezterm.

CLI:
```bash
orb shell my-ubuntu               # 기본 terminal
orb shell --terminal iterm my-ubuntu   # 특정
```

## 11. Time Machine

Mac 의 Time Machine 가 `~/OrbStack/` 자동 backup — VM 의 모든 file 보호.

> 단, OrbStack VM disk 자체 (image file) 는 Time Machine 제외 권장 (큰 용량 + 자체 snapshot 사용).
> `~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/` 는 Time Machine 에서 exclude 추가.

## 12. menubar / quick actions

OrbStack 메뉴바 🐳 클릭:
- **Open Dashboard**
- **Quit**
- **Restart**
- **Linux Machines** — 빠른 진입 list
- **Settings**

## 13. 다음

- ▶ Linux Machines: [[linux-machines]]
- ▶ K8s: [[k8s]]
- ▶ Network: [[networking]]
- ▶ Troubleshoot: [[troubleshooting]]

## 참고

- 공식 integration: https://docs.orbstack.dev/integration
- 1Password SSH agent: https://developer.1password.com/docs/ssh/
