---
title: "OrbStack — Troubleshooting"
kind: knowledge
status: current
created_at: 2026-05-22T12:48:00+09:00
tags: [devops, docker, orbstack, troubleshooting]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[cli]]"
---

# OrbStack — Troubleshooting

> 흔한 문제 + 진단 명령. 처음 만나는 증상은 본 표 먼저 확인.

## 1. 진단 시작

```bash
# 전체 상태
orb status

# 자세한 status (JSON)
orb status -o json

# log dump
orb logs --since 1h > /tmp/orb.log

# diagnostic 패키지 (지원 문의 첨부용)
orb diagnostic export ~/Downloads/orb-diag.tgz
```

## 2. 시작 / 종료

| 증상 | 원인 / 해결 |
| --- | --- |
| `orb` 명령이 "OrbStack is not running" | `orb start` 또는 `open -a OrbStack` |
| 시작 후 메뉴바 아이콘 안 보임 | 시스템 환경설정 > 제어 센터 > **OrbStack** 아이콘 표시 |
| 시작이 매우 느림 (>30초) | Mac 의 RAM 부족 — 다른 앱 종료. 또는 디스크 95%+ 가득 |
| 시작 시 권한 요청 무한 반복 | OrbStack helper 재설치 — Preferences > Advanced > **Reinstall helper** |
| `orb start` 가 "Already running" 인데 응답 없음 | `orb stop -f` 후 재시작 |

## 3. Docker

| 증상 | 해결 |
| --- | --- |
| `docker: Cannot connect to the Docker daemon` | `docker context use orbstack` 또는 `orb start` |
| `docker run hello-world` 가 ImagePullError | network 확인 — `docker info` 에서 OrbStack 시작 여부 |
| `docker buildx build --platform linux/amd64` 가 느림 | Apple Silicon 의 Rosetta 활성 — Settings > Performance |
| port 충돌 | `lsof -i :8080` 으로 어느 프로세스 — Mac native vs OrbStack |
| disk 폭주 | `docker system prune -a --volumes` + dashboard 좌하단 disk usage 확인 |
| `docker compose watch` 가 file 변경 못 감지 | Settings > File sharing — VirtioFS 활성 |

## 4. Linux Machines

| 증상 | 해결 |
| --- | --- |
| `orb create` 가 "image not found" | distro 이름 확인 (`ubuntu` / `ubuntu:24.04` 등). `orb list-distros` |
| `orb shell` 가 hang | machine stopped — `orb start <name>` 또는 `orb restart <name>` |
| `apt update` 가 timeout | DNS — `cat /etc/resolv.conf` 안의 nameserver 확인. systemctl restart systemd-resolved |
| VM 의 file 변경이 Mac 에 안 보임 | VirtioFS 캐시 — 잠시 (1~2초) 대기 또는 Finder 새로고침 (⌘R) |
| Mac 의 file 이 VM 안 `/mnt/mac/` 에 안 보임 | OrbStack 종료 후 재시작 |
| machine disk 가 full | `orb -m <name> -- df -h` 확인 — 큰 file 정리 또는 `orb snapshot delete` |
| network 격리 — VM 들 끼리 통신 안 됨 | bridge mode 활성 — Settings > Network |

## 5. Kubernetes

| 증상 | 해결 |
| --- | --- |
| `kubectl` 가 "connection refused" | `orb start k8s` |
| Pod 가 `ImagePullBackOff` | local image 사용 시 `imagePullPolicy: Never` 명시 |
| LoadBalancer external-IP 가 `<pending>` | klipper-lb pod 상태 — `kubectl get pods -n kube-system`. 1-2분 대기 |
| K8s 전체 깨짐 | dashboard > Kubernetes > **Reset cluster** 또는 `orb reset k8s` |
| Helm install 실패 | RBAC — helm 의 service account 권한 확인 |
| 어떤 namespace pod 가 안 뜸 | `kubectl describe pod <name>` 의 Events 섹션 |

## 6. Network

| 증상 | 해결 |
| --- | --- |
| `*.orb.local` 안 풀림 | `/etc/resolver/orb.local` 파일 존재? — OrbStack 재시작 |
| VPN 켜면 컨테이너 통신 안 됨 | OrbStack subnet 이 VPN subnet 와 충돌 — `orb config set network.subnet 10.42.0.0/24` |
| `localhost:port` 가 응답 없음 | container 가 `0.0.0.0` 가 아닌 `127.0.0.1` listen — bind 변경 |
| VM 안에서 `host.docker.internal` 안 됨 | `mac.internal` 도메인 시도 |
| bridge mode 후 IP 안 잡힘 | 라우터 DHCP — lease 확인 / 갱신 |

## 7. Integration / VS Code

| 증상 | 해결 |
| --- | --- |
| VS Code Remote-SSH 가 machine 못 찾음 | `~/.ssh/config` 의 `Include ~/.orbstack/ssh/config` 확인 |
| SSH agent 가 forward 안 됨 | Mac 의 `ssh-add` 후 재시도 |
| Finder 에서 OrbStack 항목 없음 | Settings > Integration > Finder 토글 다시 |
| 1Password SSH key 안 됨 | 1Password Developer 설정 확인 + Mac shell 의 `SSH_AUTH_SOCK` 환경변수 |

## 8. Performance

| 증상 | 해결 |
| --- | --- |
| Mac 발열 / 팬 소음 | idle 컨테이너 정지 — `docker ps -q | xargs docker stop` |
| 배터리 빨리 소모 | Settings > Performance > Low Power mode |
| build 느림 | Rosetta 활성 (M1+) — Settings > Performance |
| 메모리 부족 알림 | Settings > Resources > Memory limit ↑ 또는 idle 머신 stop |

## 9. 데이터 백업 / 복구

```bash
# 전체 OrbStack 데이터 location
ls ~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack/data/

# Time Machine — ~/OrbStack/ symlink 가 보호됨
# 단, image file 자체는 별도 backup (큰 용량)

# 특정 machine snapshot 백업
orb snapshot create my-ubuntu daily-$(date +%Y%m%d)

# 외부 export (실험적)
orb export my-ubuntu ~/Downloads/my-ubuntu.tar
orb import ~/Downloads/my-ubuntu.tar new-machine
```

## 10. 완전 reset

마지막 수단:

```bash
# 1. OrbStack 종료
osascript -e 'quit app "OrbStack"'

# 2. 데이터 / 캐시 모두 삭제 (주의 — 모든 image / volume / machine / K8s 사라짐)
rm -rf ~/Library/Group\ Containers/HUAQ24HBR6.dev.orbstack
rm -rf ~/.orbstack
rm -rf ~/OrbStack

# 3. OrbStack 재실행 — 처음부터
open -a OrbStack
```

## 11. log 위치

| 종류 | 경로 |
| --- | --- |
| OrbStack app log | `~/.orbstack/logs/orbstack.log` |
| VM kernel log | `orb logs` (또는 dashboard) |
| Docker daemon log | `orb logs docker` |
| K8s system log | `kubectl logs -n kube-system <pod>` |
| Crash report | `~/Library/Logs/DiagnosticReports/OrbStack*` |

## 12. 지원 채널

- **공식 GitHub Discussions**: https://github.com/orbstack/orbstack/discussions
- **Email**: support@orbstack.dev (Pro+ tier)
- **Twitter**: [@OrbStack](https://twitter.com/OrbStack)
- **Docs**: https://docs.orbstack.dev/

## 13. 다음

- ▶ 시작: [[getting-started]]
- ▶ CLI: [[cli]]
- ▶ Settings: [[settings-and-resources]]
- ▶ Docker Desktop 으로 전환?: [[vs-docker-desktop]]
