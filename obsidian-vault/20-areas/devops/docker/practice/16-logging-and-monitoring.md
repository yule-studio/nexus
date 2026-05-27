---
title: "Docker 실습 16 — Logging / Monitoring (Loki / Prometheus / Grafana)"
kind: practice
status: current
created_at: 2026-05-22T14:30:00+09:00
tags: [devops, docker, practice, logging, monitoring, prometheus, grafana, loki]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../production/patterns]]"
---

# 실습 16 — Logging / Monitoring

> 시간: 45분. 학습 포인트: log driver (json-file / local / loki / fluentd / awslogs) / Promtail
> → Loki / cAdvisor + node-exporter + Prometheus / Grafana 대시보드 / alert rule.

## 1. log driver

```bash
# 현재 default
docker info | grep "Logging Driver"
# Logging Driver: json-file
```

| driver | 특징 |
| --- | --- |
| **json-file** (default) | host 의 `/var/lib/docker/containers/<id>/<id>-json.log` |
| **local** | binary 압축, 자동 rotation (20.10+) |
| **journald** | systemd journal |
| **syslog** | rsyslog / syslog-ng |
| **fluentd** | fluentd / fluent-bit |
| **loki** | Grafana Loki |
| **awslogs** | CloudWatch Logs |
| **gcplogs** | Google Cloud Logging |
| **splunk** | Splunk HEC |
| **none** | log 안 함 |

### daemon-wide

```json
// daemon.json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

### container 단위

```bash
docker run --log-driver=loki \
  --log-opt loki-url="http://loki:3100/loki/api/v1/push" \
  --log-opt loki-retries=5 \
  --log-opt loki-batch-size=400 \
  myapp
```

## 2. Promtail + Loki + Grafana — 권장 stack

`compose.yaml`:

```yaml
services:
  api:
    image: myapp:v1
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "3"

  loki:
    image: grafana/loki:3.0
    ports: ["3100:3100"]
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:3.0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml

  grafana:
    image: grafana/grafana:11.0.0
    ports: ["3000:3000"]
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  loki-data:
  grafana-data:
```

`promtail-config.yaml`:
```yaml
server: { http_listen_port: 9080 }
positions: { filename: /tmp/positions.yaml }
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
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: stream
```

`loki-config.yaml`:
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
limits_config: { retention_period: 720h }
```

```bash
docker compose up -d
# Grafana: http://localhost:3000
```

Grafana 에서 Loki data source 추가 (URL `http://loki:3100`) → Explore → `{container="myapp"}` query.

영상: [Loki + Promtail 종합 — TechWorld with Nana](https://www.youtube.com/watch?v=h_GGd7HfKQ8)

## 3. Prometheus + cAdvisor + node-exporter

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.0
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
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
      - /dev/disk:/dev/disk:ro
    ports: ["8085:8080"]

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
    ports: ["9100:9100"]

volumes:
  prom-data:
```

`prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: cadvisor
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: node
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: docker
    static_configs:
      - targets: ['host.docker.internal:9323']

  - job_name: bkurs-be
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ['api:8080']
```

> Docker daemon metric — `daemon.json` 에 `"metrics-addr": "0.0.0.0:9323"` 추가 필요.

## 4. Spring Boot metric

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure: { include: health, info, prometheus }
  metrics:
    tags: { application: bkurs-be, env: prod }
```

```bash
curl http://api:8080/actuator/prometheus
# http_server_requests_seconds_count{...} 12345
# jvm_memory_used_bytes{...} 1234567890
# hikaricp_connections_active{...} 5
```

## 5. Grafana 대시보드 import

기존 대시보드 — Grafana > + > Import > ID:

| ID | 대시보드 |
| --- | --- |
| **893** | Docker / cAdvisor |
| **1860** | Node Exporter Full |
| **4701** | JVM (Micrometer) |
| **10619** | Spring Boot Statistics |
| **15705** | Loki — Log Explorer |
| **893** | Docker monitoring |

## 6. Alert rule (Prometheus + Alertmanager)

```yaml
# alerts.yml
groups:
  - name: bkurs
    rules:
      - alert: BkursHigh5xx
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "5xx 비율 5% 초과 5분"

      - alert: BkursHighLatency
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 10m
        labels: { severity: warning }

      - alert: BkursJvmHeapPressure
        expr: |
          jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 5m
        labels: { severity: critical }

      - alert: BkursDown
        expr: up{job="bkurs-be"} == 0
        for: 2m
        labels: { severity: critical }
```

Alertmanager → webhook (Slack / Discord / 카카오):

```yaml
# alertmanager.yml
route:
  receiver: slack
receivers:
  - name: slack
    slack_configs:
      - api_url: https://hooks.slack.com/services/...
        channel: '#alerts'
```

## 7. application log → 구조화 (JSON)

Logback (`logback-spring.xml`):
```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <customFields>{"app":"bkurs-be","env":"prod"}</customFields>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

→ stdout 이 JSON → Promtail 가 parse 후 Loki 의 label / field 로 indexing.

## 8. 실습 — 종합 흐름

1. compose stack 띄움 (api + db + cache + Loki + Promtail + Prometheus + cAdvisor + Grafana)
2. 부하 (`ab` / `hey` / `k6`)
3. Grafana 에서 메트릭 / 로그 확인
4. alert rule 정의 → trigger 시험 (cpu 폭주 / 5xx 강제)
5. webhook 으로 Slack 알림

## 9. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Loki volume 폭주 | retention_period 설정 + log level 조정 |
| cAdvisor privileged 필요 | true 또는 cgroup mount 만 명시 |
| Prometheus rule reload | `kill -HUP <pid>` 또는 API `/-/reload` |
| Spring metric 비공개 | `exposure.include: prometheus` |
| Alertmanager 의 receiver typo | `amtool` 으로 validate |

## 10. 영상 / 자료

| 자료 |
| --- |
| [Loki + Promtail — TechWorld with Nana](https://www.youtube.com/watch?v=h_GGd7HfKQ8) |
| [Prometheus + Grafana — Bret Fisher](https://www.youtube.com/watch?v=h4Sl21AKiDg) |
| [cAdvisor docs](https://github.com/google/cadvisor) |
| [Grafana Dashboard library](https://grafana.com/grafana/dashboards/) |

## 11. 다음

- ▶ [[17-debug-and-troubleshoot]]
- ▶ [[20-end-to-end-bkurs-style]] — 종합 프로젝트
- ▶ [[../desktop/resource-usage]] — Desktop 측 모니터링

## 12. 참고

- 공식 logging: https://docs.docker.com/config/containers/logging/
- Loki: https://grafana.com/docs/loki/
- Prometheus: https://prometheus.io/docs/
- cAdvisor: https://github.com/google/cadvisor
