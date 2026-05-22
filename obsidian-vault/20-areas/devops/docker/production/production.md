---
title: "Docker production — 운영 / 패턴 hub"
kind: knowledge
status: current
created_at: 2026-05-22T13:48:00+09:00
tags: [devops, docker, production, hub]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[../practice/practice]]"
---

# Docker production — 운영 / 패턴 hub

**[[../docker|↑ docker]]**

> 컨테이너 production 배포 시 챙겨야 하는 패턴 / 함정. dev 단계의 "작동" 과 prod 의 "안정 +
> 관측" 은 매우 다름.

## 1. 학습 순서

| 노트 | 내용 |
| --- | --- |
| [[patterns]] ★ | sidecar / init container / healthcheck / probe / restart policy / graceful shutdown |
| [[pitfalls]] | 흔한 함정 (PID 1 / signal / log flooding / time / DNS / OOM) |

## 2. 운영 체크리스트 (요약)

| 항목 | 권장 |
| --- | --- |
| **HEALTHCHECK** | Dockerfile 에 명시 (또는 compose `healthcheck`) |
| **non-root USER** | `USER 1000:1000` 또는 distroless `:nonroot` |
| **read-only filesystem** | `--read-only` + tmpfs (`/tmp` 등) |
| **resource limits** | `--cpus`, `--memory`, `--pids-limit` |
| **restart policy** | `on-failure:5` / `unless-stopped` |
| **log driver** | `json-file` + rotation (`max-size: 50m`, `max-file: 3`) — 또는 fluentd / awslogs / Loki |
| **graceful shutdown** | SIGTERM 수신 → 진행 중 요청 완료 → exit (timeout 10s default) |
| **secret 주입** | env var (단순) / docker secret (swarm) / external (SSM / Vault) |
| **image pinning** | `latest` 금지 — semver 또는 digest |
| **scan + sign** | 매 build 시 — [[../practice/13-image-scan-scout-and-trivy]] / [[../practice/14-sign-and-verify-cosign]] |
| **observability** | metric (Prometheus) + log (Loki) + trace (Tempo) |

영상: [Docker Production Patterns — Bret Fisher](https://www.youtube.com/watch?v=GwbjRqK01-c)

## 3. 실습 매핑

| 실습 | doc |
| --- | --- |
| 15 healthcheck / restart / resources | [[../practice/15-healthcheck-restart-resources]] |
| 16 logging / monitoring | [[../practice/16-logging-and-monitoring]] |
| 17 debug / troubleshoot | [[../practice/17-debug-and-troubleshoot]] |

## 4. 학습 영상

| 영상 | 길이 |
| --- | --- |
| [Docker Production Best Practices — Bret Fisher](https://www.youtube.com/watch?v=GwbjRqK01-c) | 45분 |
| [Signals + PID 1 — DevOps Toolkit](https://www.youtube.com/watch?v=ZcRoFEC6Xeo) | 12분 |
| [Container Resource Limits — Liz Rice](https://www.youtube.com/watch?v=sK5i-N34im8) | 20분 |
| [Observability — Honeycomb](https://www.youtube.com/watch?v=R3K0xCnWE5g) | 30분 |

## 5. 흔한 함정 목록 (요약 — [[pitfalls]] 가 SSoT)

- **PID 1 문제**: shell script ENTRYPOINT 가 child process 의 signal 안 받음 → `tini` / `dumb-init` 사용
- **OOMKilled**: `--memory` 미설정 시 host OOM Killer 가 다른 프로세스 죽임
- **log 폭주**: rotation 안 하면 디스크 폭주 — daemon.json `log-opts`
- **time skew**: 컨테이너 시간이 host 와 다름 — NTP / 호스트 timezone mount
- **DNS 캐시**: 짧은 TTL 의 ALB endpoint → 새 IP 못 받음
- **graceful shutdown**: HTTP server 의 SIGTERM handler 누락 → 진행 중 요청 drop

## 6. 참고

- 공식 production checklist: https://docs.docker.com/engine/install/linux-postinstall/
- 12-Factor App: https://12factor.net/
- OWASP Docker Top 10: https://github.com/OWASP/Docker-Security
- Liz Rice book: Container Security (O'Reilly)

## 7. 관련

- [[../docker]]
- [[../fundamentals/security]]
- [[../desktop/scout]] (예정) — Scout / CVE
- [[../practice/practice]] — Level 6 운영 실습
