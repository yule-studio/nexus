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

## 7. 관련

- [[docker|↑ docker]]
- [[dockerfile-best-practices]]
- [[volume]]
- [[networking]]
