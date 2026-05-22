---
title: "Docker 핵심 개념"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:08:00+09:00
tags: [devops, docker, concepts]
---

# Docker 핵심 개념

**[[docker|↑ docker]]**

---

## 1. 5 가지 개념

| 개념 | 무엇 |
| --- | --- |
| **Image** | 읽기 전용 template (filesystem + metadata) |
| **Container** | image 의 running instance (process + r/w layer) |
| **Layer** | image 의 변경 단위 (cache 됨) |
| **Volume** | 컨테이너 밖 영속 storage |
| **Network** | 컨테이너 간 통신 |

---

## 2. Image / Container

```
[Image]                            [Container]
├── base layer (Ubuntu)            (image 의 instance)
├── layer 1 (apt install)          + writable layer
├── layer 2 (COPY app.jar)         + network namespace
└── layer 3 (CMD java -jar)        + process namespace
                                   + cgroup limits
```

- image = "class".
- container = "object" (instance).
- 1 image → N container (각자 독립 process).

---

## 3. Layer 와 cache

```dockerfile
FROM ubuntu:22.04             # layer 1
RUN apt-get update            # layer 2
RUN apt-get install -y nginx  # layer 3 (cache hit if 1,2 변경 X)
COPY index.html /var/www/     # layer 4 (cache hit if file 동일)
CMD ["nginx", "-g", "daemon off;"]
```

→ 위에서 변경 시 아래 모든 layer rebuild. **자주 변경되는 건 아래**.

---

## 4. Volume vs Bind mount vs tmpfs

| Type | 위치 | 사용 |
| --- | --- | --- |
| **Volume** | Docker 관리 | DB 데이터 / production |
| **Bind mount** | host 임의 경로 | dev (소스 코드) |
| **tmpfs** | RAM | 캐시 / 민감 데이터 |

```bash
docker run -v mydata:/data ...           # volume
docker run -v $(pwd):/app ...            # bind
docker run --tmpfs /tmp ...              # tmpfs
```

자세히: [[volume]].

---

## 5. Network mode

| Mode | 사용 |
| --- | --- |
| **bridge** (default) | 컨테이너 격리 + port mapping |
| **host** | host network 직접 사용 (성능 ↑) |
| **none** | network X (보안) |
| **overlay** | Swarm / k8s 의 cross-node |

자세히: [[networking]].

---

## 6. 6 가지 자주 헷갈리는 것

| 헷갈림 | 정답 |
| --- | --- |
| `docker stop` vs `docker kill` | stop=SIGTERM (15s grace), kill=SIGKILL 즉시 |
| `docker rm` vs `docker rmi` | rm=container, rmi=image |
| `docker exec` vs `docker run` | exec=실행 중 container 안, run=새 container |
| `ENTRYPOINT` vs `CMD` | ENTRYPOINT=항상 실행, CMD=default 인자 (덮어쓰기) |
| `EXPOSE` vs `-p` | EXPOSE=문서 only, -p=실제 binding |
| `VOLUME` (Dockerfile) vs `-v` | VOLUME=anonymous (껐다켜면 사라짐), -v=named |

---

## 7. 5 개념 관계도

```mermaid
flowchart TD
  REG[(Container Registry<br/>Docker Hub / GHCR / ECR)]
  IMG[Image<br/>+ layers + metadata]
  REG -->|pull| IMG
  REG <--|push| IMG

  IMG -->|docker run| C1[Container 1]
  IMG -->|docker run| C2[Container 2]

  V1[(Volume<br/>named)]
  V2[(Bind mount<br/>host path)]
  C1 -->|mount /var/lib/...| V1
  C2 -->|mount /app/src| V2

  NET[Network bridge<br/>app-net]
  C1 --- NET
  C2 --- NET

  HOST[(Host kernel + filesystem)]
  C1 -.->|namespace / cgroup| HOST
  C2 -.->|namespace / cgroup| HOST
```

> container 는 **host 의 process** — kernel 공유. VM 아님. image 는 filesystem 의 immutable
> snapshot. 같은 image 가 N container 로 spawn.

## 8. 더 깊이 → [[mental-models]]

- 컨테이너 = process (PID 1 / signal / OOM)
- layer = content-addressable (sha256)
- OCI image-spec / runtime-spec / distribution-spec
- BuildKit DAG / cache backend

## 9. 학습 영상

| 영상 | 길이 | 추천 |
| --- | --- | --- |
| [Docker explained — Fireship](https://www.youtube.com/watch?v=Gjnup-PuquQ) | 8분 | 가장 빠른 입문 |
| [What is a container? — Liz Rice](https://www.youtube.com/watch?v=8fi7uSYlOdc) | 45분 | namespace/cgroup deep |
| [Docker Tutorial — TechWorld with Nana](https://www.youtube.com/watch?v=3c-iBn73dDE) | 2h | 종합 입문 ★ |

## 10. 실습 매핑

| 개념 | 실습 |
| --- | --- |
| image / container 기본 | [[../practice/01-hello-world]] |
| Dockerfile / layer | [[../practice/04-multi-stage-build]] |
| network DNS-by-name | [[../practice/07-networks-and-dns]] |
| volume / bind / tmpfs | [[../practice/08-volumes-and-bind-mounts]] |
| compose 다중 컨테이너 | [[../practice/03-compose-full-stack]] |

## 11. 관련

- [[fundamentals]] — fundamentals hub
- [[mental-models]] — 깊이 노트 (process / layer / OCI)
- [[networking]] / [[volume]] / [[registry]] / [[security]] / [[runtime-comparison]]
- [[../dockerfile/best-practices]] — Dockerfile 작성
- [[../docker]] — root hub
