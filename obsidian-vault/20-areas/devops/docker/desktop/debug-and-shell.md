---
title: "Docker Desktop — Debug / Shell / Logs / Inspect / Files"
kind: knowledge
status: current
created_at: 2026-05-22T14:10:00+09:00
tags: [devops, docker, docker-desktop, debug, troubleshoot]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[dashboard]]"
---

# Docker Desktop — Debug / Shell / Logs / Inspect

> 컨테이너 안에 진입 / 로그 확인 / 파일 browse / network inspect / stats 모니터링. Desktop GUI
> 의 강점 — CLI 만으로는 N 개 명령 조합 필요.

## 1. Container detail panel — 5 탭

Dashboard > Containers > {id} 클릭:

### 1.1 Logs

- 실시간 stdout / stderr tail
- 검색 (⌘F) + 강조
- Wrap text / Auto-scroll 토글
- Clear / Save to file
- Filter (예: `level=ERROR`)

CLI 동등:
```bash
docker logs -f --since 1h --tail 100 <id>
docker logs <id> 2>&1 | grep ERROR
```

### 1.2 Inspect

JSON inspect 통째 — network / mount / env / labels / state / config 등.

CLI:
```bash
docker inspect <id>
docker inspect --format '{{.State.Health.Status}}' <id>
docker inspect --format '{{json .NetworkSettings.Networks}}' <id> | jq
```

### 1.3 Bind mounts

host ↔ container path 매핑 — bind 한 모든 path 표시. 클릭 시 host Finder 열기 (macOS).

### 1.4 Exec — 컨테이너 안 shell

내장 terminal 진입:
- 자동 `bash` 또는 `sh` (alpine 등) 시도
- ⌘+T 로 새 terminal 탭

CLI:
```bash
docker exec -it <id> bash
docker exec -it <id> sh             # alpine
docker exec --user 0 -it <id> bash  # root 강제
docker exec -e DEBUG=true -it <id> bash
```

### 1.5 Files — 파일 시스템 browse

트리 view:
- 우클릭 → save (host 로 다운로드) / open (편집기) / view (인라인)
- ⌘F 검색
- 작은 텍스트 파일은 인라인 편집 가능

CLI:
```bash
docker cp <id>:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./fix.conf <id>:/etc/nginx/nginx.conf
docker exec <id> ls -la /etc/nginx/
```

### 1.6 Stats (실시간 그래프)

- CPU %
- Memory MB / %
- Network IO (in / out)
- Block IO (read / write)

CLI:
```bash
docker stats <id>
# 또는 모든
docker stats
```

## 2. Debug pane (Docker Desktop 4.30+)

Containers > {id} > **Debug** (실험적, 일부 versions):
- Java thread dump (jstack)
- Heap dump 다운로드
- profiler 통합 (특정 image 한정)

## 3. Container 상태 한눈에

| 상태 | 의미 |
| --- | --- |
| Running | 실행 중 |
| Restarting | restart policy 의 반복 |
| Paused | `docker pause` (process freeze) |
| Exited (0) | 정상 종료 |
| Exited (N) | exit code N — 비정상 (`docker inspect` 의 `State.Error`) |
| Dead | rm 못 한 상태 (filesystem 문제) |

## 4. Healthcheck 결과

Dockerfile `HEALTHCHECK` 정의 시:

```bash
docker inspect --format='{{json .State.Health}}' <id>
# {
#   "Status": "healthy",
#   "FailingStreak": 0,
#   "Log": [{ "Start": "...", "End": "...", "ExitCode": 0, "Output": "..." }]
# }
```

Dashboard 의 컨테이너 row 에 ✓ healthy / ✗ unhealthy / ... starting 아이콘.

## 5. 디버깅 시나리오

### 5.1 컨테이너가 시작 직후 죽음

```bash
docker logs <id>                     # crash log
docker inspect <id> | jq .State      # ExitCode / Error
docker run --entrypoint sh -it <image>   # entrypoint 우회 + 안에서 직접 실행
```

GUI: Containers > {id} > Logs

### 5.2 healthcheck 실패

```bash
docker inspect --format='{{json .State.Health}}' <id> | jq
docker exec <id> /healthcheck-script.sh   # 직접 실행
```

GUI: Logs / Inspect 탭

### 5.3 network 통신 안 됨

```bash
docker exec <id> ping db
docker exec <id> nslookup db
docker exec <id> wget -O- http://db:5432   # plaintext check
docker network inspect <network-name>
```

### 5.4 권한 오류 (Permission denied)

```bash
docker exec <id> id              # uid / gid 확인
docker exec <id> ls -la /app     # 파일 owner
docker exec -u 0 -it <id> bash   # root 로 진입 후 chmod
```

### 5.5 memory 부족 (OOMKilled)

```bash
docker inspect <id> | jq '.State.OOMKilled'   # true?
docker stats <id>                              # 사용량 확인

# 해결
docker update --memory 2g <id>
# 또는 compose
deploy:
  resources:
    limits: { memory: 2G }
```

## 6. `docker events` — 실시간 이벤트

```bash
docker events --filter container=myapp
# {start, stop, die, kill, oom, restart, health_status}

# 모든 이벤트
docker events --filter event=die
docker events --since 1h
```

GUI 의 Dashboard 가 본 이벤트를 시각적으로 표시.

## 7. `docker top` — 컨테이너 안 process

```bash
docker top <id>
# UID    PID    PPID   CMD
# 0      1234   1233   nginx: master
# 0      1235   1234   nginx: worker
```

GUI: Stats 탭에 process count.

## 8. log driver — production 권장

기본 `json-file` 은 디스크 폭주 위험. 권장 daemon.json:

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
```

또는 외부 (Loki / fluentd / awslogs):
```bash
docker run --log-driver=loki \
  --log-opt loki-url="http://loki:3100/loki/api/v1/push" \
  myapp
```

## 9. dive — image layer 분석 (외부)

```bash
brew install dive
dive myapp:v1
```

각 layer 의 변경 파일 / 크기 / 효율 (% efficiency). GUI 보다 자세.

자세히 → [[../practice/10-image-tags-and-layers]]

## 10. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Exec 가 connection lost | container stopped — start 후 재시도 |
| Files browse 가 빈 트리 | 마찬가지 stopped — start |
| Logs 없음 | entrypoint 가 stdout 안 보냄 — TTY / detach |
| Stats 가 0 표시 | 짧은 lived task — `docker stats --no-stream` |
| OOMKilled 모름 | `docker events --filter event=oom` |

## 11. 영상 / 자료

| 자료 |
| --- |
| [Container Debug 101 — Bret Fisher](https://www.youtube.com/watch?v=YkW2EBu7nlk) |
| [docker events / stats / inspect docs](https://docs.docker.com/reference/cli/docker/) |
| [dive](https://github.com/wagoodman/dive) |

## 12. 다음

- ▶ [[resource-usage]] — CPU / Mem / Disk 모니터링
- ▶ [[../practice/17-debug-and-troubleshoot]] (예정) — 실습
- ▶ [[../production/pitfalls]] — 운영 함정

## 13. 참고

- 공식 docker logs / exec / inspect: https://docs.docker.com/engine/reference/commandline/
