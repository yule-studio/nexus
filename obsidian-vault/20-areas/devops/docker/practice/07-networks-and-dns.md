---
title: "Docker 실습 07 — Networks / DNS / 격리"
kind: practice
status: current
created_at: 2026-05-22T13:00:00+09:00
tags: [devops, docker, practice, network, dns]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../fundamentals/networking]]"
---

# 실습 07 — Networks / DNS / 격리

> 시간: 30분. 학습 포인트: bridge / host / overlay / macvlan / none / user-defined network /
> service DNS (`container name 으로 resolve`) / network alias / 격리 / publish vs expose.

## 1. network driver 5종

| driver | 용도 |
| --- | --- |
| **bridge** (default) | 단일 host. 컨테이너간 통신 + NAT |
| **host** | 컨테이너가 host network namespace 공유 (port 격리 없음) |
| **overlay** | 다중 host (Swarm / k8s). VXLAN tunnel |
| **macvlan** | 컨테이너가 LAN 의 직접 MAC + IP (예: 라우터 / IoT) |
| **none** | network 없음 (격리 / 보안 테스트) |

영상: [Docker Networking — NetworkChuck](https://www.youtube.com/watch?v=bKFMS5C4CG0) (18분, 시각화 우수)

## 2. user-defined bridge — 권장 패턴

default `bridge` (docker0) 는 **DNS 없음** + 모든 컨테이너 연결. **user-defined bridge** 가
DNS-by-name + 격리 + IP 자동.

```bash
docker network create mynet
docker network ls
# NAME      DRIVER    SCOPE
# bridge    bridge    local      ← default (사용 비추)
# mynet     bridge    local      ← user-defined (권장)
```

## 3. 실습 A — 같은 network 의 컨테이너 통신

```bash
docker network create app-net

# DB 먼저
docker run -d --name db --network app-net \
  -e POSTGRES_PASSWORD=secret postgres:16

# API — DB 컨테이너 이름으로 resolve (DNS-by-name)
docker run -d --name api --network app-net \
  -e DB_URL=jdbc:postgresql://db:5432/postgres \
  -p 8080:8080 \
  myapp:v1

# api 안에서 db 통신 확인
docker exec api ping -c 1 db
# PING db (172.18.0.2): 56 data bytes
docker exec api nc -zv db 5432
# Connection to db 5432 port [tcp/postgresql] succeeded!
```

> 핵심: 같은 user-defined network 안에서 **container name = DNS name**. 별도 `--link` 불필요.

## 4. network alias — 여러 이름

```bash
docker run -d --name db --network app-net \
  --network-alias postgres --network-alias primary-db \
  -e POSTGRES_PASSWORD=secret postgres:16

# api 가 db / postgres / primary-db 모두 resolve
docker exec api nslookup postgres
```

활용: 호환성 (legacy app 이 다른 이름 expect) / blue-green (alias 만 갈아끼움).

## 5. 격리 — 다른 network = 통신 불가

```bash
docker network create web-net
docker network create db-net

docker run -d --name web --network web-net nginx
docker run -d --name db --network db-net postgres:16

docker exec web ping db
# bad address 'db' — 다른 network, DNS 안 풂
```

해결 — 양쪽 network 에 연결 (proxy 같은 역할):
```bash
docker network connect db-net web
docker exec web ping db
# OK (web 이 web-net + db-net 둘 다)
```

## 6. publish (`-p`) vs expose (`EXPOSE`)

| 항목 | publish (`-p`) | expose (`EXPOSE`) |
| --- | --- | --- |
| 효과 | host port → container port mapping | 메타데이터만 (문서화 + `--link`) |
| 외부 접근 | 가능 | 불가 |
| 사용 | runtime 시점 결정 | Dockerfile 에 기록 |

```bash
# publish — host:8080 → container:80
docker run -d -p 8080:80 nginx

# random host port
docker run -d -P nginx
docker port <id>
# 80/tcp -> 0.0.0.0:32769
```

> `-p 127.0.0.1:8080:80` — 외부 LAN 차단, 본인만 access.

## 7. host network — port 격리 없음

```bash
docker run -d --network host nginx
# nginx 가 host 의 :80 직접 점유 — port mapping 불필요
curl http://localhost:80
```

**주의** (macOS / Windows): host network 가 VM 안. Docker Desktop 의 `Enable host networking on Mac (beta)` 또는 OrbStack 의 자동 통합 필요.

활용: 성능 critical (latency / throughput), 다중 port 동적.

## 8. macvlan — LAN 의 직접 IP

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=en0 lan-net

docker run -d --network lan-net --ip=192.168.1.100 --name iot nginx
# 192.168.1.100 — 라우터에서 직접 보이는 device
```

활용: IoT 시뮬레이션 / 다른 host 의 LAN device 처럼 보이기.

## 9. none — 완전 격리

```bash
docker run --rm --network none alpine ip a
# lo 만 존재. 외부 network 없음
```

활용: 격리 빌드 / 보안 테스트 / network 없이 컴퓨테이션.

## 10. DNS resolver 설정

```bash
# 컨테이너 specific
docker run --dns 8.8.8.8 --dns 1.1.1.1 alpine nslookup google.com

# search domain
docker run --dns-search corp.local alpine ...

# /etc/hosts 추가
docker run --add-host db:192.168.1.50 alpine ping db
```

daemon 단위 (default):
```json
// /etc/docker/daemon.json 또는 Desktop > Settings > Docker Engine
{
  "dns": ["1.1.1.1", "8.8.8.8"],
  "dns-search": ["corp.local"]
}
```

## 11. inspect / diagnose

```bash
docker network inspect app-net
# IPAM / connected containers / driver options 등 JSON

docker exec api cat /etc/resolv.conf
# nameserver: Docker embedded DNS (127.0.0.11)

# host 의 bridge 확인
ip addr show docker0
ip addr show br-<network-id>
```

## 12. compose 의 network

`compose.yaml` 의 모든 service 가 자동 같은 default network — DNS-by-service-name:

```yaml
services:
  api:
    image: myapp
  db:
    image: postgres:16
networks:
  default: { name: myapp-net }
```

multi-network:
```yaml
services:
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]
  web:
    networks: [frontend]
networks:
  frontend:
  backend:
```

`api` 만 frontend + backend 양쪽 — proxy 역할.

## 13. 검증 / cleanup

```bash
# 모든 network
docker network ls

# unused 정리
docker network prune

# 특정 network 의 컨테이너 강제 분리
docker network disconnect -f mynet container-name
```

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| default `bridge` 에서 DNS-by-name 시도 | user-defined network 사용 |
| `localhost` 가 다른 컨테이너 가리킬 거라 기대 | container 마다 자체 loopback — service name 사용 |
| `-p` 안 했는데 외부 접근 기대 | publish 필수 |
| host network on macOS | beta 옵션 활성 또는 OrbStack |
| macvlan 의 parent 인터페이스 typo | `ip link show` 로 정확한 이름 |

## 15. 다음

- ▶ [[08-volumes-and-bind-mounts]] — 데이터 영구화
- ▶ [[09-compose-watch-and-profiles]] — compose 고급
- ▶ [[../fundamentals/networking]] — bridge/overlay/macvlan 이론

## 참고

- 공식 networking: https://docs.docker.com/network/
- NetworkChuck Docker networking: https://www.youtube.com/watch?v=bKFMS5C4CG0
- iximiuz Labs Network deep: https://labs.iximiuz.com/courses/docker-networking
