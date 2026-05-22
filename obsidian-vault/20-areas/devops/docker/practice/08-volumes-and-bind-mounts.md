---
title: "Docker 실습 08 — Volumes / Bind mounts / tmpfs / 백업"
kind: practice
status: current
created_at: 2026-05-22T13:05:00+09:00
tags: [devops, docker, practice, volume, persistence]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../fundamentals/volume]]"
---

# 실습 08 — Volumes / Bind mounts / tmpfs / 백업

> 시간: 30분. 학습 포인트: named volume / bind mount / tmpfs 차이 / 데이터 영구성 / 백업 /
> 복원 / 권한 / 성능 / driver.

## 1. 3 종 mount 비교

| 종류 | 위치 | 용도 | 영구성 |
| --- | --- | --- | --- |
| **named volume** | Docker managed (`/var/lib/docker/volumes/`) | DB / 영구 데이터 | ✓ 컨테이너 삭제 후 보존 |
| **bind mount** | host 의 임의 경로 | dev (소스 sync) / config | ✓ host 파일 그대로 |
| **tmpfs** | RAM | 임시 / 보안 (secret) | ✗ 컨테이너 stop 시 사라짐 |

영상: [Docker Volume vs Bind Mount — TechWorld with Nana](https://www.youtube.com/watch?v=p2PH_YPCsis) (10분)

## 2. named volume — 권장 (DB / 영구 데이터)

```bash
# 생성 (옵션 — 자동 생성도 됨)
docker volume create db-data
docker volume ls
# DRIVER    VOLUME NAME
# local     db-data

# 사용
docker run -d --name db -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret postgres:16

# 컨테이너 삭제 후에도 보존
docker rm -f db
docker run -d --name db2 -v db-data:/var/lib/postgresql/data postgres:16
# 이전 DB 그대로
```

inspect:
```bash
docker volume inspect db-data
# Mountpoint: /var/lib/docker/volumes/db-data/_data
```

> **macOS / Windows**: 위 path 는 VM 안. host 에서 직접 access 불가. `docker run --rm -v db-data:/d alpine ls /d` 처럼.

## 3. bind mount — dev / config

```bash
# 현재 폴더를 컨테이너 안에
docker run -d --name dev -v $PWD:/app -w /app -p 8080:8080 \
  eclipse-temurin:21 ./mvnw spring-boot:run

# 단일 file
docker run -v $PWD/config.yml:/etc/myapp/config.yml:ro myapp
```

**옵션**:
- `:ro` — read-only
- `:rw` (default) — read-write
- `:z` / `:Z` — SELinux label (Linux only)

## 4. tmpfs — RAM 임시

```bash
docker run -d --tmpfs /tmp:size=100m,mode=1777 myapp
# /tmp 가 100MB RAM. 컨테이너 stop 시 사라짐
```

활용: secret / 큰 buffer / RAM 의 IO 속도 활용.

## 5. 새 syntax — `--mount` (권장)

`-v` 는 압축 syntax — `--mount` 가 명시적:

```bash
docker run -d --name db \
  --mount type=volume,source=db-data,target=/var/lib/postgresql/data \
  postgres:16

docker run -d --name dev \
  --mount type=bind,source="$PWD",target=/app,readonly \
  myapp

docker run --tmpfs /tmp:size=100m alpine
# 또는
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=100m alpine
```

## 6. 권한 / UID 매핑

**문제**: 컨테이너의 root (UID 0) 가 host 의 file 을 root 권한으로 변경 → host 에서 sudo 필요.

해결:
- `--user $(id -u):$(id -g)` — host user UID 그대로
- Dockerfile 의 `USER` directive
- `chown` initContainer 패턴

```bash
docker run --rm -u 1000:1000 -v $PWD:/app alpine touch /app/test
# host 에서 host user 소유로 생성
```

## 7. 실습 — Postgres 데이터 영구화 + 컨테이너 교체

```bash
# 1. 첫 컨테이너
docker run -d --name db -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret postgres:16
docker exec db psql -U postgres -c "CREATE TABLE foo (id int)"
docker exec db psql -U postgres -c "INSERT INTO foo VALUES (42)"

# 2. 삭제
docker rm -f db

# 3. 다른 버전으로 재시작
docker run -d --name db2 -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret postgres:17     # 17 로 업그레이드

# 4. 데이터 확인
docker exec db2 psql -U postgres -c "SELECT * FROM foo"
#  id
# ----
#  42
```

## 8. 백업 / 복원

### tar 방식 (가장 일반적)

```bash
# 백업 — volume 의 내용을 tar.gz 로
docker run --rm \
  -v db-data:/data \
  -v $PWD:/backup \
  alpine tar czf /backup/db-data-$(date +%Y%m%d).tar.gz -C /data .

# 복원
docker run --rm \
  -v db-data-new:/data \
  -v $PWD:/backup \
  alpine tar xzf /backup/db-data-20260522.tar.gz -C /data
```

### Postgres / MySQL dump (DB 전용)

DB 는 logical dump 권장 (raw file 보다 안전):
```bash
# Postgres
docker exec db pg_dump -U postgres mydb | gzip > backup.sql.gz

# 복원
gunzip -c backup.sql.gz | docker exec -i db2 psql -U postgres mydb
```

### Docker Desktop / OrbStack GUI

- Docker Desktop: Volumes Backup & Share extension — 1-click tar / S3
- OrbStack: `~/OrbStack/<machine>/var/lib/docker/volumes/` 직접 접근

## 9. compose 의 volume

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data    # named volume
      - ./initdb:/docker-entrypoint-initdb.d:ro   # bind mount

volumes:
  db-data:
    # driver: local (default)
    # external: true                         # 외부에서 미리 생성된 volume
```

```bash
docker compose up -d
docker volume ls
# myapp_db-data (compose project prefix)

# compose down 후에도 volume 보존
docker compose down                          # volume 보존
docker compose down -v                       # ⚠️ volume 도 삭제
```

## 10. volume driver — local 외

```bash
# NFS
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/data \
  nfs-data

# 또는 cifs / glusterfs / S3 plugin
docker plugin install rclone/docker-volume-rclone --grant-all-permissions
docker volume create -d rclone/docker-volume-rclone --opt remote=mys3:bucket s3-vol
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `docker compose down -v` 로 실수로 prod data 삭제 | prod 는 `external: true` named volume |
| host 의 absolute path 가 아닌 relative path bind | absolute path 사용 또는 `$(pwd)` |
| bind 한 폴더 권한이 root 인데 컨테이너 가 non-root | `--user` 또는 Dockerfile USER |
| named volume 의 mountpoint 직접 편집 | `docker run --rm -v vol:/d alpine sh` 로 접근 |
| Mac/Win 에서 bind mount 의 file IO 느림 | VirtioFS 활성, 또는 named volume 사용 |
| volume mount 시 mount target 의 기존 file 사라짐 | mount 가 그 폴더 통째 덮음 — 의도된 동작 |

## 12. 검증

```bash
# 사용 중인 volume
docker volume ls
docker system df -v       # volume 별 size + 컨테이너

# unused 정리 (주의)
docker volume prune

# 강제 prune (in-use 까지)
docker volume rm $(docker volume ls -q) -f
```

## 13. 다음

- ▶ [[09-compose-watch-and-profiles]] — compose 고급
- ▶ [[../fundamentals/volume]] — driver / plugin
- ▶ [[../desktop/dashboard#3]] — GUI volume export/import

## 참고

- 공식 volumes: https://docs.docker.com/storage/volumes/
- bind vs volume: https://docs.docker.com/storage/bind-mounts/
- TechWorld with Nana: https://www.youtube.com/watch?v=p2PH_YPCsis
