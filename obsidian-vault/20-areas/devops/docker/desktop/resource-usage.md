---
title: "Docker Desktop — Resource Usage / Disk / Memory / Cleanup"
kind: knowledge
status: current
created_at: 2026-05-22T14:12:00+09:00
tags: [devops, docker, docker-desktop, resource, cleanup]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[settings]]"
---

# Docker Desktop — Resource Usage / Cleanup

> CPU / Memory / Disk / Network 모니터링 + 정기 정리. Disk usage extension 권장.

## 1. 실시간 자원 표시

### Dashboard 좌하단

```
RAM: 2.1 GB / 8 GB        ▮▮▮▮▯▯▯▯
CPU: 12%                  ▮▮▯▯▯▯▯▯
Disk: 35 GB / 64 GB       ▮▮▮▮▮▯▯▯
```

활성화: Settings > General > **Show resource usage**.

### `docker stats` — terminal

```bash
docker stats
# CONTAINER ID  NAME      CPU %    MEM USAGE / LIMIT     NET I/O    BLOCK I/O
# abc123        myapp     2.3%     150MiB / 2GiB         100KB/MB   ...
# def456        db        0.1%     50MiB / 1GiB          50KB/100KB ...
```

`docker stats --no-stream` — 한 번만 출력.

## 2. Disk usage

### CLI

```bash
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          25        10        4.5GB     2.8GB (62%)
# Containers      8         3         200MB     150MB (75%)
# Local Volumes   12        4         8.2GB     3.1GB (37%)
# Build Cache     50        0         3.5GB     3.5GB

docker system df -v       # verbose — 각 image / volume 별
```

### GUI — Disk usage extension

```
Dashboard > Extensions > Marketplace > "Disk usage" 설치
```

시각화:
- image 별 크기 (layer 공유 고려)
- volume 별 크기 + 마지막 접근
- build cache sub-tree
- 1-click 정리 버튼

## 3. 정리 명령 (level 별)

### 3.1 가벼운 정리

```bash
# 정지된 컨테이너만
docker container prune

# dangling image 만 (untagged + 의존 없음)
docker image prune

# unused network
docker network prune
```

### 3.2 보통 정리

```bash
docker system prune
# - 정지 컨테이너
# - dangling image
# - unused network
# - build cache
# (volume / tagged image 보존)
```

### 3.3 강력 정리 (주의)

```bash
docker system prune -a
# 위 + tagged image 중 의존 컨테이너 없는 것

docker system prune -a --volumes
# 위 + unused volume 까지 (⚠️ 데이터 손실)
```

### 3.4 build cache 만

```bash
docker buildx prune
docker buildx prune -af               # 전체
docker buildx prune --keep-storage 10GB    # 10GB 만 남기고
```

## 4. 자동 정리 — daemon.json

```json
{
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "20GB",
      "policy": [
        { "keepStorage": "10GB", "filter": ["unused-for=24h"] }
      ]
    }
  },
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  }
}
```

## 5. Disk image — 위치 / 크기 조정

Settings > Resources > **Advanced**:
- **Disk image size**: 64GB (default sparse) → 200GB 등
- **Disk image location**: 외장 SSD 옵션 (macOS)

CLI 등가:
```bash
# 위치 확인
docker info | grep -i "Docker Root Dir"

# macOS — diskutil 로 sparse 파일 크기 보기
ls -lh ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw
```

> Disk image 는 sparse — 표시 크기와 실제 사용량 다름. `du -sh` 가 실제.

## 6. CPU / RAM limit 변경

### container 단위

```bash
docker run --cpus=2 --memory=1g --memory-swap=1g myapp
docker run --cpus=0.5 --memory=512m myapp

# 실행 중 변경
docker update --cpus=2 --memory=2g <id>
```

### compose

```yaml
services:
  api:
    deploy:
      resources:
        limits: { cpus: "2", memory: 2G }
        reservations: { cpus: "0.5", memory: 256M }
```

### Desktop VM 전체

Settings > Resources > Advanced → CPUs / Memory / Swap. Apply & restart.

## 7. log rotation

기본 `json-file` 의 default 무제한 — 디스크 폭주.

권장:
```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
```

→ 컨테이너당 150MB 한도 (50MB × 3 file).

또는 외부 log driver:
- `local` (rotation 자동, Docker 20.10+)
- `loki` / `fluentd` / `awslogs`

```bash
docker run --log-driver=local --log-opt max-size=50m myapp
```

## 8. 메모리 OOM 방지

```bash
# 컨테이너 메모리 추적
docker stats <id>

# OOM 발생 확인
docker inspect <id> | jq .State.OOMKilled

# 해결
docker update --memory 2g --memory-swap 2g <id>
```

또는 JVM 메모리 (Spring Boot 등):
```bash
docker run -e JAVA_OPTS="-Xmx1g -XX:MaxRAMPercentage=75" myapp
```

## 9. CPU throttle 진단

```bash
# cgroup throttle 통계
docker exec <id> cat /sys/fs/cgroup/cpu.stat
# nr_throttled: ...
# throttled_usec: ...
```

throttle 많이 발생 → cpus limit 증가 또는 affinity 조정.

## 10. 모니터링 통합 (Prometheus)

Docker daemon 에 metric endpoint 활성:

```json
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

```bash
curl http://localhost:9323/metrics
# engine_daemon_container_states_containers ...
```

Prometheus scrape config:
```yaml
- job_name: docker
  static_configs:
    - targets: ['host.docker.internal:9323']
```

자세히 → [[../practice/16-logging-and-monitoring]] (예정)

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `system prune -a --volumes` 실수 | volume 은 별도 명령 권장 |
| log 파일 폭주 | log-opts 의 max-size + max-file |
| RAM 부족 (Desktop VM) | Settings > Resources > Memory ↑ |
| Disk image 가 호스트 디스크 가득 | size 줄임 + prune + 외장 SSD 이전 |
| stats 가 0 표시 | container 가 paused 또는 0 CPU |

## 12. 영상 / 자료

| 자료 |
| --- |
| [Docker Resource Limits — Bret Fisher](https://www.youtube.com/watch?v=Vyp5_F42NGs) |
| [JVM memory in containers — Aboullaite](https://www.youtube.com/watch?v=KvBakaTBV-c) |
| [system df / prune docs](https://docs.docker.com/engine/reference/commandline/system/) |

## 13. 다음

- ▶ [[debug-and-shell]] — stats / inspect / logs
- ▶ [[settings]] — Resources 패널
- ▶ [[../practice/15-healthcheck-restart-resources]] (예정) — 실습
- ▶ [[../production/patterns]] — production resource limit

## 14. 참고

- 공식: https://docs.docker.com/engine/reference/commandline/system_df/
- Disk usage extension: https://hub.docker.com/extensions/docker/disk-usage-extension
