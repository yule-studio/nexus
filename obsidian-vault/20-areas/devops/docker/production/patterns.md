---
title: "Production patterns — healthcheck / sidecar / init"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:24:00+09:00
tags: [devops, docker, production, pattern]
---

# Production patterns — healthcheck / sidecar / init

**[[docker|↑ docker]]**

---

## 1. HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1
```

→ unhealthy 시 LB / k8s 가 traffic 빠짐.

---

## 2. Graceful shutdown (SIGTERM)

```java
// Spring Boot — auto handled
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

```dockerfile
# Java 는 SIGTERM 안 받음 (PID 1 X) 시
ENTRYPOINT ["sh", "-c", "exec java -jar app.jar"]   # exec 키워드 = PID 1
```

→ k8s preStop hook 도 활용 가능.

---

## 3. Init container (k8s 영향)

```yaml
# Compose 도 depends_on + healthcheck 로 비슷한 효과
services:
  migrate:
    image: myapp:1.0
    command: ["./gradlew", "flywayMigrate"]
    depends_on:
      postgres:
        condition: service_healthy
  app:
    image: myapp:1.0
    depends_on:
      migrate:
        condition: service_completed_successfully
```

---

## 4. Sidecar (log shipper)

```yaml
services:
  app:
    image: myapp:1.0
    volumes:
      - logs:/app/logs
  log-shipper:
    image: fluent/fluent-bit
    volumes:
      - logs:/logs:ro
      - ./fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
volumes:
  logs:
```

→ k8s 도 같은 pattern (같은 Pod 안 컨테이너).

---

## 5. Resource limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
```

→ JVM 의 경우 `-XX:MaxRAMPercentage=75` 와 함께.

---

## 6. Restart policy

```yaml
services:
  app:
    restart: unless-stopped   # always | on-failure | unless-stopped | no
```

| Policy | 동작 |
| --- | --- |
| `no` | 수동 |
| `on-failure` | exit code 0 아닐 때 만 |
| `always` | 항상 |
| `unless-stopped` ★ | 사용자 수동 stop 외 |

---

## 7. logging driver

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

→ default 의 무한 log 방지. production = fluent-bit / loki 직접 전송.

---

## 8. 함정

1. **HEALTHCHECK 없음** → LB 가 죽은 컨테이너 로 traffic.
2. **PID 1 = shell** → SIGTERM 안 받음 → `exec` 추가.
3. **Resource limit 없음** → OOM kill 다른 컨테이너 영향.
4. **log driver default** → disk 무한 누적.
5. **restart: always + crash loop** → 무한 재시작 (다음 시도 backoff X).

---

## 9. 관련

- [[docker|↑ docker]]
- [[compose]]
- [[../kubernetes/kubernetes|↗ kubernetes — 같은 패턴]]
