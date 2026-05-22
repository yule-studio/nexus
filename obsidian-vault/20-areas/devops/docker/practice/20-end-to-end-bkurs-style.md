---
title: "Docker 실습 20 — End-to-End bkurs 스타일 종합 프로젝트"
kind: practice
status: current
created_at: 2026-05-22T14:55:00+09:00
tags: [devops, docker, practice, end-to-end, bkurs, observability]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../../10-projects/bkurs-be/bkurs-be]]"
---

# 실습 20 — End-to-End 종합 프로젝트 (bkurs 스타일)

> 시간: 2시간. 학습 포인트: **Spring Boot + MySQL + nginx reverse proxy + Prometheus + Grafana
> + Loki + Promtail** 의 통합 stack. 실습 1~19 의 모든 요소 한 자리에. bkurs-be 의 미니어처.

## 1. 목표 아키텍처

```mermaid
flowchart TD
  USER[User<br/>localhost:80] --> NGINX[nginx<br/>reverse proxy + TLS]
  NGINX --> API[Spring Boot API<br/>:8080]
  API --> DB[(MySQL 9<br/>:3306)]

  API -.metrics./actuator/prometheus.-> PROM[Prometheus]
  PROM --> GRAF[Grafana<br/>:3000]
  API -.logs json-file.-> PROMTAIL[Promtail] --> LOKI[Loki] --> GRAF
  CADV[cAdvisor] --> PROM
  NODE[node-exporter] --> PROM
```

## 2. 사전 준비

```bash
mkdir bkurs-stack && cd bkurs-stack
# 디렉토리 구조
# ├── compose.yaml
# ├── api/Dockerfile
# ├── api/src/...
# ├── nginx/nginx.conf
# ├── prometheus/prometheus.yml
# ├── promtail/config.yaml
# ├── loki/config.yaml
# ├── grafana/provisioning/
# └── .env
```

## 3. Dockerfile (Spring Boot — 실습 04 패턴)

`api/Dockerfile`:
```dockerfile
# syntax=docker/dockerfile:1.7
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS runtime
RUN addgroup -S app && adduser -S app -G app
USER app
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 --start-period=30s \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]
```

## 4. compose.yaml — 전체 stack

```yaml
services:
  # ─── Application ───
  api:
    build:
      context: ./api
    image: ghcr.io/seum-running/bkurs-be:dev
    restart: unless-stopped
    ports: ["8080:8080"]                # 내부 직접 접근 (옵션 — 보통 nginx 만 노출)
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/bkurs_dev
      SPRING_DATASOURCE_USERNAME: bkurs
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      SPRING_PROFILES_ACTIVE: dev
      MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: health,info,prometheus,metrics
      JAVA_OPTS: "-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"
    depends_on:
      db: { condition: service_healthy }
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 60s
    deploy:
      resources:
        limits:   { cpus: "2", memory: 2G, pids: 200 }
        reservations: { cpus: "0.5", memory: 512M }
    labels:
      - "logging=promtail"
      - "logging_jobname=api"
    develop:
      watch:
        - action: sync
          path: ./api/src
          target: /app/src
        - action: rebuild
          path: ./api/pom.xml

  # ─── DB ───
  db:
    image: mysql:9.0
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: bkurs_dev
      MYSQL_USER: bkurs
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 5s
      timeout: 5s
      retries: 10
    deploy:
      resources:
        limits: { cpus: "1", memory: 1G }

  # ─── Reverse proxy ───
  nginx:
    image: nginx:1.27-alpine
    restart: unless-stopped
    ports: ["80:80"]
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      api: { condition: service_healthy }
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost"]
      interval: 10s

  # ─── Metrics ───
  prometheus:
    image: prom/prometheus:v3.0
    restart: unless-stopped
    ports: ["9090:9090"]
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prom-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.0
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro

  node-exporter:
    image: prom/node-exporter:v1.8
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/rootfs

  # ─── Logs ───
  loki:
    image: grafana/loki:3.0
    ports: ["3100:3100"]
    volumes:
      - ./loki/config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:3.0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail/config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml

  # ─── Grafana ───
  grafana:
    image: grafana/grafana:11.0.0
    ports: ["3000:3000"]
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - grafana-data:/var/lib/grafana

volumes:
  db-data:
  prom-data:
  loki-data:
  grafana-data:

networks:
  default:
    name: bkurs-stack
```

## 5. `.env`

```dotenv
MYSQL_ROOT_PASSWORD=root-secret
DB_PASSWORD=bkurs-pwd
```

⚠️ `.env` 는 `.gitignore`.

## 6. `nginx/nginx.conf`

```nginx
upstream api_backend {
    server api:8080;
}

server {
    listen 80;
    server_name _;

    location /healthz {
        return 200 "ok";
        add_header Content-Type text/plain;
    }

    location / {
        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

## 7. `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  external_labels: { stack: bkurs-dev }

scrape_configs:
  - job_name: prometheus
    static_configs: [{ targets: ['localhost:9090'] }]

  - job_name: cadvisor
    static_configs: [{ targets: ['cadvisor:8080'] }]

  - job_name: node
    static_configs: [{ targets: ['node-exporter:9100'] }]

  - job_name: bkurs-api
    metrics_path: /actuator/prometheus
    static_configs: [{ targets: ['api:8080'] }]
```

## 8. `loki/config.yaml`

```yaml
auth_enabled: false
server: { http_listen_port: 3100 }
common:
  ring: { kvstore: { store: inmemory } }
  replication_factor: 1
  path_prefix: /loki
schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index: { prefix: index_, period: 24h }
limits_config:
  retention_period: 168h     # 7일
```

## 9. `promtail/config.yaml`

```yaml
server:
  http_listen_port: 9080
positions:
  filename: /tmp/positions.yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: container
      - source_labels: ['__meta_docker_container_label_logging_jobname']
        target_label: job
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: stream
```

## 10. `grafana/provisioning/datasources/datasources.yaml`

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

`grafana/provisioning/dashboards/dashboards.yaml`:
```yaml
apiVersion: 1
providers:
  - name: default
    folder: bkurs
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

dashboard 파일 (`bkurs.json`) — Grafana 의 dashboard library 에서 import 후 export.

## 11. 실행

```bash
docker compose up -d --build
docker compose ps
# 모두 healthy 확인

# 접근
curl http://localhost/actuator/health         # via nginx
curl http://localhost:9090                    # Prometheus
open http://localhost:3000                    # Grafana

# 부하
ab -n 10000 -c 50 http://localhost/api/v1/health

# log 확인
docker compose logs -f api
# 또는 Grafana > Explore > Loki > {container="bkurs-stack-api-1"}
```

## 12. 검증 체크리스트

- [ ] api 가 healthy (compose ps)
- [ ] nginx → api proxy 작동 (`curl http://localhost`)
- [ ] Prometheus 가 bkurs-api 의 metric scrape 성공 (Status > Targets)
- [ ] Grafana 에서 `http_server_requests_seconds_count` query 가 데이터 반환
- [ ] Loki 에서 `{container=~"bkurs.*"}` log 보임
- [ ] cAdvisor metric (container_memory_usage_bytes) Grafana 에서 보임
- [ ] OOM 시뮬 — api 의 메모리 한도 도달 시 자동 restart

## 13. dev mode (watch)

```bash
docker compose watch
# api/src/ 변경 → 컨테이너 자동 sync (Spring DevTools hot reload)
# api/pom.xml 변경 → 자동 rebuild
```

자세히 → [[09-compose-watch-and-profiles]]

## 14. scan + sign (CI 통합)

`.github/workflows/build.yml` 의 패턴 — 실습 19. 본 stack 의 api 이미지를 GHCR push + Cosign sign +
Trivy scan + SBOM 자동.

## 15. K8s 배포 (옵션)

```bash
docker compose bridge convert
kubectl apply -f ./out/
```

→ Desktop / OrbStack K8s 위에 동일 stack. 자세히 → [[18-kubernetes-with-docker-desktop]]

## 16. tear down

```bash
docker compose down            # 컨테이너 + network
docker compose down -v         # ⚠️ + volume (DB / Grafana data 삭제)
docker compose down --rmi all  # + image
```

## 17. 다음 단계 — 실제 prod 로

| 항목 | 가이드 |
| --- | --- |
| AWS ECS on EC2 배포 | [[../../10-projects/bkurs-be/handover/operations/env-deployment]] |
| 백업 (mysqldump + EBS snapshot) | 같은 doc §4 |
| SSM Parameter Store 의 secret | 같은 doc §5 |
| Alertmanager → 카카오/SMS/Discord | 같은 doc §2.5 |
| Cosign + policy enforcement | [[14-sign-and-verify-cosign]] |

## 18. 영상 / 자료

| 자료 |
| --- |
| [Full Stack Docker — Bret Fisher](https://www.youtube.com/watch?v=GwbjRqK01-c) |
| [Loki + Grafana — TechWorld with Nana](https://www.youtube.com/watch?v=h_GGd7HfKQ8) |
| [Awesome Compose 예제](https://github.com/docker/awesome-compose) |
| [Grafana Dashboard library](https://grafana.com/grafana/dashboards/) |

## 19. 졸업 🎓

본 실습 (1~20) 완수 시:
- ✅ Docker / Dockerfile / Compose 마스터
- ✅ Network / Volume / Registry 깊이
- ✅ Security (scan / sign / policy)
- ✅ 운영 (healthcheck / restart / resource / observability)
- ✅ K8s 통합
- ✅ CI/CD (GitHub Actions + scan + sign)
- ✅ End-to-End 종합 stack

**다음 학습 영역**:
- [[../../kubernetes/kubernetes|↗ Kubernetes 본격]]
- [[../../cicd/cicd|↗ CI/CD 일반]]
- [[../../observability/observability|↗ Observability]]
- [[../../security-ops/security-ops|↗ Security Ops]]
- [[../production/patterns]] — production 패턴

## 20. 참고

- bkurs-be 운영 가이드: [[../../10-projects/bkurs-be/handover/operations/env-deployment]]
- 12-Factor App: https://12factor.net/
- Awesome Compose: https://github.com/docker/awesome-compose
- Grafana docs: https://grafana.com/docs/
