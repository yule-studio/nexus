---
title: "Docker Desktop — Volumes / Bind mounts GUI"
kind: knowledge
status: current
created_at: 2026-05-22T14:02:00+09:00
tags: [devops, docker, docker-desktop, volume, bind-mount]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[dashboard]]"
  - "[[../fundamentals/volume]]"
---

# Docker Desktop — Volumes / Bind mounts (GUI)

> Dashboard > Volumes 에서 할 수 있는 모든 작업 + File sharing 설정 (macOS / Windows). CLI 와 GUI
> 1:1 매핑.

## 1. Volumes 화면 — 5 컬럼

| 컬럼 | 의미 |
| --- | --- |
| **Name** | named volume 이름 |
| **In use** | 사용 중인 컨테이너 (있으면 컨테이너 이름 클릭 가능) |
| **Created** | 생성 시각 |
| **Driver** | `local` (default) / custom plugin |
| **Mountpoint** | host 의 path (Mac/Win 은 VM 안) |

## 2. 액션 (각 row 우측 메뉴)

| 액션 | 결과 | CLI |
| --- | --- | --- |
| **View content** | volume 안 파일 트리 + 작은 파일 인라인 view | (`docker run --rm -v vol:/d alpine ls /d`) |
| **Export** | tar.gz 로 백업 다운로드 | (`docker run --rm -v vol:/d -v $PWD:/b alpine tar czf /b/vol.tgz /d`) |
| **Import** | tar.gz 업로드 → 새 volume | (`docker volume create new && docker run --rm -v new:/d -v $PWD:/b alpine tar xzf /b/vol.tgz -C /d`) |
| **Clone** | 같은 내용의 새 volume | (export + import) |
| **Delete** | volume 제거 (in-use 거부) | `docker volume rm <name>` |

> Export / Import 가 Desktop GUI 의 강점 — CLI 만으로는 tar 명령 조합 필요.

## 3. View content — 인라인 편집

작은 텍스트 파일 (e.g. config.yml) 은 직접 편집 가능:
- 더블 클릭 → editor 열림 (textarea)
- 저장 시 volume 안 갱신

큰 binary / 큰 텍스트는 download 만 가능.

## 4. Volume 생성 (Create new volume)

상단 **+ Create** 버튼:
- Volume name 입력
- Labels (선택)
- Driver (default `local`, custom plugin 있으면 선택)

CLI 동일:
```bash
docker volume create --label app=bkurs --label env=prod db-data
```

## 5. Prune — dangling 일괄 삭제

상단 ⋯ 메뉴 > **Prune** — 사용 안 하는 volume 정리.

```bash
docker volume prune    # 또는
docker volume prune -af    # 모든 (label 무관)
```

> ⚠️ 영구 데이터 손실 — 정리 전 confirm.

## 6. Bind mount 설정 — File sharing

`docker run -v $PWD:/app ...` 가 작동하려면 host 의 그 path 가 Docker 에 노출돼야 함.

### macOS / Windows

Settings > Resources > **File sharing**:
- default 노출 path: `/Users`, `/Volumes`, `/private`, `/tmp` (macOS)
- 추가하려면 + 버튼 → path 입력 → Apply

### 구현 backend

| Backend | 성능 | 특징 |
| --- | --- | --- |
| **VirtioFS** (default macOS 12.5+) | ★★★ | M1 native, 100MB+ 빠름 |
| **gRPC FUSE** | ★★ | 호환성 ↑, 약간 느림 |
| **Direct host mounts** (Linux) | ★★★ | native (VM 없음) |

Settings > Resources > File sharing > **Choose file sharing implementation** 에서 전환.

## 7. CLI inspect

```bash
# named volume
docker volume inspect db-data
# [{
#   "CreatedAt": "...",
#   "Driver": "local",
#   "Labels": null,
#   "Mountpoint": "/var/lib/docker/volumes/db-data/_data",
#   "Name": "db-data",
#   "Options": null,
#   "Scope": "local"
# }]

# 사용 중인 컨테이너
docker ps --filter volume=db-data
```

## 8. 일반적인 use case GUI 흐름

### 8.1 dev DB 백업
1. Volumes > `myapp_db-data` > **Export**
2. 저장 위치 선택 → tar.gz
3. 매일 cron 으로 또는 수동

### 8.2 다른 PC 로 데이터 이전
1. PC A: Volumes > Export
2. tar.gz 를 PC B 로 전송 (Dropbox / scp)
3. PC B: Volumes > Import → 같은 데이터

### 8.3 staging 복원
1. prod 의 `pg_dump` 또는 volume export 받기
2. Volumes > Import → `staging-db-data`
3. compose `volumes: [staging-db-data:/var/lib/postgresql/data]`

### 8.4 디스크 사용 확인
1. Settings > Resources > **Disk usage** (extension 필요)
2. 또는 CLI: `docker system df -v`

## 9. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| host 의 폴더가 File sharing 에 없어 mount 실패 | Settings > Resources > File sharing 추가 |
| Volume export 시 in-use 컨테이너 → 불일치 | DB 는 stop 또는 `pg_dump` 사용 |
| Disk image 위치 변경 | Settings > Resources > **Disk image location** |
| named volume 의 mountpoint 직접 편집 (Mac/Win) | VM 안 — `docker run --rm -v vol:/d alpine ...` 로 |
| compose 의 anonymous volume 누적 | `docker volume prune` 정기 |

## 10. 영상 / 자료

| 자료 |
| --- |
| [Docker Volume vs Bind Mount — TechWorld with Nana](https://www.youtube.com/watch?v=p2PH_YPCsis) |
| [Persistent Storage — Bret Fisher](https://www.youtube.com/watch?v=z0Vfomd_OUE) |
| [Volumes Backup & Share extension](https://hub.docker.com/extensions/docker/volumes-backup-extension) |

## 11. 다음

- ▶ [[../fundamentals/volume]] — 이론 (named / bind / tmpfs)
- ▶ [[../practice/08-volumes-and-bind-mounts]] — 실습
- ▶ [[dashboard]] — 전체 Dashboard
- ▶ [[settings]] — File sharing 등 설정

## 12. 참고

- 공식: https://docs.docker.com/storage/volumes/
- Volumes Backup ext: https://hub.docker.com/extensions/docker/volumes-backup-extension
