---
title: "Docker pitfalls"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:26:00+09:00
tags: [devops, docker, pitfalls]
---

# Docker pitfalls

**[[docker|↑ docker]]**

---

## 함정 모음

1. **`latest` tag** — 재현 X. semver + git sha 사용.
2. **root user** — break out 시 host root.
3. **secret 을 ARG / ENV 로** — image layer 영구. BuildKit secret.
4. **`-v /:/host`** — host 전체 mount, 매우 위험.
5. **`--privileged`** — 모든 capability.
6. **default bridge 사용** → DNS X. custom bridge.
7. **HEALTHCHECK 없음** → 죽은 컨테이너에 traffic.
8. **PID 1 = shell** → SIGTERM 못 받음 → `exec`.
9. **resource limit 없음** → OOM 한 컨테이너가 host 영향.
10. **anonymous volume** → 데이터 손실.
11. **bind mount + DB** → 권한 / UID 문제.
12. **`docker compose down` (down -v 없이)** → volume 남음.
13. **layer cache 무시** (자주 변경 → 위) → 매번 rebuild 1 분.
14. **`.dockerignore` 없음** → context 1GB → build 느림.
15. **`apt-get update` only** (no install + clean) → 캐시 layer 큼.
16. **same `RUN` 으로 합쳐야 layer 1개** → multiple RUN = multiple layer.
17. **multi-stage 안 함** → 1.5GB image (JDK 포함).
18. **log driver default** → 디스크 무한 누적.
19. **restart: always 의 crash loop** → backoff 없이 무한 재시작.
20. **CVE scan 안 함** → image 의 알려진 취약점.

---

## 관련

- [[docker|↑ docker]]
- [[dockerfile-best-practices]]
- [[security]]
- [[production-patterns]]
