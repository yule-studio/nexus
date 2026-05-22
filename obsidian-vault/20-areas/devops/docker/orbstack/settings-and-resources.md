---
title: "OrbStack — Settings / Resources"
kind: knowledge
status: current
created_at: 2026-05-22T12:40:00+09:00
tags: [devops, docker, orbstack, settings]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[../desktop/settings]]"
---

# OrbStack — Settings / Resources

> 모든 설정 패널 + 각 옵션 의미 + 권장값. CLI 로도 동등하게 수정 가능.

## 1. Settings 진입

- Dashboard > 우상단 ⚙️ → Preferences
- 또는 메뉴바 OrbStack → Preferences
- 또는 CLI: `~/.orbstack/config/orbstack.conf`

## 2. General

| 옵션 | 의미 |
| --- | --- |
| **Launch at login** | 부팅 시 자동 실행 |
| **Quit OrbStack when last window closes** | 마지막 window 닫으면 quit |
| **Theme** | Light / Dark / System |
| **Update channel** | Stable / Beta |
| **Allow telemetry** | 사용 통계 (anonymized) |
| **Finder integration** | sidebar OrbStack 표시 |
| **Spotlight integration** | machine 검색 가능 |

## 3. Resources

### 3.1 System

| 옵션 | default | 권장 |
| --- | --- | --- |
| **CPU limit** | dynamic | dynamic (idle 시 자동 0) |
| **Memory limit** | dynamic / 8GB max | 4~8GB (host 의 절반) |
| **Disk size** | dynamic | dynamic (사용한 만큼) |

OrbStack 의 강점 — **dynamic resource**. idle 시 RAM 자동 반환. Docker Desktop 의 static 할당과 차이.

### 3.2 Performance

| 옵션 | 의미 |
| --- | --- |
| **Rosetta 2** | x86_64 emulation (Apple Silicon) — 자동 |
| **VirtioFS** | file sync backend (default) |
| **Power mode** | Performance / Balanced / Low Power |

## 4. Network

| 옵션 | 의미 |
| --- | --- |
| **Subnet** | OrbStack 의 NAT subnet (default 192.168.215.0/24) |
| **Bridge mode** | machine 이 host network 직접 access (회사 VPN 충돌 시 유용) |
| **Auto port forwarding** | VM 의 listen port 자동 Mac 노출 |
| **orb.local domain** | 자동 DNS (`.orb.local` 자동 resolver) |
| **MTU** | network packet size (default 1500) |

자세히 → [[networking]]

## 5. Docker

| 옵션 | 의미 |
| --- | --- |
| **Enable Docker** | docker engine 활성 (default true) |
| **Engine config** | `daemon.json` 동등 JSON 편집 |
| **Auto-prune** | builds cache / image GC 자동 |
| **Resource limits per container** | cgroup default (cpu/mem) |

예: registry mirror 설정
```json
{
  "registry-mirrors": ["https://mirror.gcr.io"],
  "insecure-registries": ["registry.local:5000"]
}
```

## 6. Kubernetes

| 옵션 | 의미 |
| --- | --- |
| **Enable Kubernetes** | K3s cluster 활성 |
| **Version** | K3s minor version |
| **CNI** | flannel / calico (옵션) |
| **Ingress controller** | Traefik (default) / nginx |

자세히 → [[k8s]]

## 7. Linux Machines

| 옵션 | 의미 |
| --- | --- |
| **Default username** | 신규 머신 생성 시 기본 user (예: `mac` 또는 `ubuntu`) |
| **Default password** | sudo password (선택) |
| **SSH config injection** | `~/.ssh/config` 에 자동 `Include orbstack` |
| **Boot timeout** | machine 시작 대기 시간 |
| **NFS-based home mount** | `/mnt/mac/` 의 backend (VirtioFS vs NFS) |

자세히 → [[linux-machines]]

## 8. Integrations

| 옵션 | 의미 |
| --- | --- |
| **Add machines to Finder sidebar** | Finder 에 OrbStack 항목 |
| **Add machines to Spotlight** | Spotlight 검색 |
| **VS Code Remote** | `~/.ssh/config` 자동 (Remote-SSH 통합) |
| **1Password CLI integration** | 1Password 가 OrbStack 안 SSH key 인식 |

자세히 → [[integration]]

## 9. Advanced

| 옵션 | 의미 |
| --- | --- |
| **Diagnostic mode** | 추가 log + crash report |
| **Reset cluster** | 모든 데이터 wipe (주의) |
| **Export logs** | `/tmp/orbstack-logs.tgz` 생성 (지원 문의 시) |
| **Debug shell** | hypervisor 진입 (지원 case only) |

## 10. CLI 로 같은 설정

```bash
orb config get
# 전체 설정 출력

orb config set network.mtu 1450
orb config set docker.engine '{"registry-mirrors":["..."]}'
orb config set k8s.version 1.31

# 재시작 필요할 수도
orb restart
```

config 파일 직접 편집: `~/.orbstack/config/orbstack.conf` (TOML)

## 11. 권장 setup — 첫 사용자

1. **Theme: System** (light/dark 자동)
2. **Launch at login: OFF** (수동 실행)
3. **CPU limit: dynamic / Memory limit: dynamic** (default)
4. **Rosetta: ON** (자동)
5. **Finder integration: ON** + Spotlight: ON
6. **K8s: 처음엔 OFF** (필요할 때만 켜기 — RAM 절약)
7. **Docker engine config**: registry-mirrors 추가 (회사 환경 시)

## 12. troubleshooting

| 증상 | 해결 |
| --- | --- |
| 설정 안 저장됨 | OrbStack 가 quit 상태 — 실행 후 재시도 |
| `orb config set ...` 실패 | 권한 — `sudo` 필요 X (user config) |
| VM 시작 느림 | Performance mode > Performance |
| 메뉴바 아이콘 안 보임 | macOS 설정 > Control Center > OrbStack 활성 |

## 13. 다음

- ▶ CLI: [[cli]]
- ▶ Network: [[networking]]
- ▶ Integration: [[integration]]
- ▶ Troubleshoot: [[troubleshooting]]

## 참고

- 공식: https://docs.orbstack.dev/settings
