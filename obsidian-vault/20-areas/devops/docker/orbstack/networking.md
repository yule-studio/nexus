---
title: "OrbStack — Networking (orb.local / port / bridge)"
kind: knowledge
status: current
created_at: 2026-05-22T12:44:00+09:00
tags: [devops, docker, orbstack, networking]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[settings-and-resources]]"
  - "[[../fundamentals/networking]]"
---

# OrbStack — Networking

> OrbStack 가 자동 처리하는 network 통합. `*.orb.local` 도메인 + auto port forwarding + bridge.
> Docker Desktop 의 `host.docker.internal` 와 다른 모델.

## 1. 핵심 모델

```
┌──────────────────────────────────────────┐
│  Mac host                                │
│                                          │
│   ┌──────────────────────────────────┐   │
│   │  OrbStack VM (Apple VZ)          │   │
│   │  ┌─────────┐    ┌────────────┐   │   │
│   │  │ Docker  │    │ Linux machines │   │
│   │  │ engine  │    │ ubuntu / debian│   │
│   │  └─────────┘    └────────────┘   │   │
│   │      │              │             │   │
│   │      └──────┬───────┘             │   │
│   │             │                     │   │
│   │         NAT bridge (default)      │   │
│   │         192.168.215.0/24          │   │
│   └─────────────┬──────────────────────┘   │
│                 │                          │
│       ┌─────────┴─────────┐                │
│       │ Mac network stack │                │
│       │  (en0/utun/127.x) │                │
│       └───────────────────┘                │
└──────────────────────────────────────────┘
```

## 2. `*.orb.local` 도메인 — 자동 DNS

OrbStack 가 다음 도메인 자동 resolver:

| 도메인 | 가리키는 곳 |
| --- | --- |
| `<container>.orb.local` | docker container 이름 |
| `<machine>.orb.local` | Linux machine |
| `<service>.k8s.orb.local` | K8s service (default ns) |
| `<service>.<ns>.svc.k8s.orb.local` | K8s service (특정 ns) |

예:
```bash
# container
docker run -d --name web nginx
curl http://web.orb.local

# Linux machine
orb create ubuntu my-ubuntu
orb -m my-ubuntu -- sudo apt install -y nginx && sudo systemctl start nginx
curl http://my-ubuntu.orb.local

# K8s
kubectl run myapp --image=nginx --port=80 --expose
curl http://myapp.k8s.orb.local
```

> Docker Desktop 에는 없는 기능 — `localhost:port` 충돌 회피.

## 3. port forwarding — 자동

VM 안의 listen port 가 Mac localhost 에 자동 노출:

```bash
# VM 안 (nginx on :80)
orb -m my-ubuntu -- sudo systemctl start nginx

# Mac 에서 즉시
curl http://localhost:80                  # OK (자동 forward)
# 또는 도메인으로
curl http://my-ubuntu.orb.local
```

Docker container 의 port 도 자동:
```bash
docker run -d -p 8080:80 nginx           # 명시적 -p 도 OK
# 또는 -P (random) 도 자동 도메인으로 접근:
docker run -d -P --name web nginx
curl http://web.orb.local                # port mapping 불필요
```

## 4. host 측 service 접근 (VM → Mac)

| 방법 | 명령 |
| --- | --- |
| Mac IP 직접 | VM 안에서 `host.docker.internal` 또는 `host.containers.internal` |
| Mac localhost | `mac.internal` 도메인 |
| Mac 의 특정 서비스 (예 :3000) | `curl http://mac.internal:3000` (VM 안에서) |

> Docker Desktop 의 `host.docker.internal` 도 동작. OrbStack 는 추가로 `mac.internal` 제공.

## 5. bridge mode (advanced)

기본은 NAT — VM 이 Mac IP 뒤. **bridge mode** 활성 시 VM 이 LAN 의 직접 IP 가짐:

Settings > Network > **Bridge mode** → 권장 시점:
- 회사 VPN + VPN 내부 IP 가 VM 도 필요
- 같은 LAN 의 다른 device 가 VM 에 직접 접근 필요
- DHCP IP 가 필요한 case (예: K8s metallb)

bridge mode 부작용:
- VM 이 인터넷 직접 access (별도 firewall)
- 공유기 DHCP 가 VM 별 IP 할당

## 6. subnet / 충돌 회피

회사 VPN 의 subnet (예 `192.168.0.0/16`) 가 OrbStack 의 default (`192.168.215.0/24`) 와 충돌 시:

```bash
orb config set network.subnet 10.42.0.0/24
orb restart
```

또는 GUI: Settings > Network > Subnet.

## 7. 컨테이너 간 네트워크 (Docker 기본)

OrbStack 의 Docker engine 도 일반 Docker — bridge / user-defined network 동일:

```bash
docker network create mynet
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name api -e DB_HOST=db myapp

# api 컨테이너에서 db.orb.local 또는 db (network alias) 둘 다 가능
```

자세히 → [[../fundamentals/networking]]

## 8. K8s service 노출

```yaml
# Service of type LoadBalancer
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  type: LoadBalancer
  ports: [{ port: 80, targetPort: 80 }]
  selector: { app: web }
```

OrbStack 가 klipper-lb 로 자동 Mac port 매핑 + `web.k8s.orb.local` 도메인.

```bash
curl http://web.k8s.orb.local
# 또는 (LoadBalancer port 가 80 이면)
curl http://localhost:80
```

## 9. 포트 충돌 해결

`localhost:5432` 가 Mac native PostgreSQL 와 VM 안 컨테이너 PostgreSQL 가 동시 사용 시 → 충돌.

해결:
- 도메인 사용 — `curl http://db.orb.local:5432` (충돌 회피)
- 다른 port — `-p 5433:5432`
- bridge mode + VM 의 별도 IP

## 10. DNS / nslookup

```bash
# Mac 에서
dscacheutil -q host -a name my-ubuntu.orb.local
# 또는
host my-ubuntu.orb.local
# my-ubuntu.orb.local has address 192.168.215.5

# VM 안에서
orb -m my-ubuntu -- cat /etc/resolv.conf
# nameserver: OrbStack 의 DNS proxy
```

## 11. proxy / firewall

회사 HTTP proxy:
- Mac 의 system proxy 자동 상속
- VM 안 별도 설정: `/etc/environment` 의 `HTTP_PROXY` / `HTTPS_PROXY`
- Docker pull 도 자동 (OrbStack 의 engine 이 Mac proxy 따름)

## 12. troubleshooting

| 증상 | 해결 |
| --- | --- |
| `*.orb.local` 안 풀림 | OrbStack 종료 → 재시작. `/etc/resolver/orb.local` 확인 |
| port 충돌 | `lsof -i :8080` 으로 어느 프로세스 — Mac native vs OrbStack |
| VPN 켜면 컨테이너 통신 안 됨 | subnet 충돌 — `orb config set network.subnet ...` |
| bridge mode 후 IP 변경 | DHCP — 라우터 lease 확인 |
| VM 안에서 외부 통신 안 됨 | DNS — `/etc/resolv.conf` 또는 host firewall |

## 13. 다음

- ▶ Docker network 일반: [[../fundamentals/networking]]
- ▶ K8s: [[k8s]]
- ▶ Linux Machines: [[linux-machines]]
- ▶ Integration: [[integration]]

## 참고

- 공식 networking: https://docs.orbstack.dev/network/
