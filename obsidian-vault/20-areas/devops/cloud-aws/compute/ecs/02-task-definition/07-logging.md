---
title: "ECS 02-7 — Logging (awslogs / firelens / fluentbit / Loki)"
kind: knowledge
status: current
created_at: 2026-05-22T13:42:00+09:00
tags: [aws, ecs, task-definition, logging, cloudwatch, firelens, fluentbit]
home_hub: ecs
---

# Task Definition — Logging

> container 의 stdout/stderr 를 어디로 보낼지. **awslogs** (CloudWatch) 가 default + 가장 단순.
> **firelens** + fluentbit 가 Loki / Splunk / Datadog 같은 외부로 분산.

## 1. log driver 종류

| Driver | 출처 / 대상 |
| --- | --- |
| **awslogs** | CloudWatch Logs (가장 일반적) |
| **firelens** | fluentbit / fluentd 사이드카 → 임의 대상 (Loki / S3 / Splunk / Kinesis) |
| **splunk** | Splunk HTTP Event Collector 직접 |
| **awsfirelens** | firelens 의 다른 이름 |
| **gelf** | Graylog |
| **fluentd** | fluentd 직접 |
| **journald** / **syslog** | EC2 launch only |

## 2. awslogs — default

```json
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/bkurs-be",
      "awslogs-region": "ap-northeast-2",
      "awslogs-stream-prefix": "ecs",
      "awslogs-create-group": "true"
    }
  }
}
```

생성된 log group:
```
/ecs/bkurs-be/
├── ecs/api/abc123-task-id     ← 각 task 의 stream
├── ecs/api/def456-task-id
└── ...
```

### 자동 생성

`awslogs-create-group: true` — log group 없으면 자동 생성. Execution Role 에 `logs:CreateLogGroup` 추가:
```json
{
  "Effect": "Allow",
  "Action": ["logs:CreateLogGroup"],
  "Resource": "*"
}
```

### Retention 설정

```bash
aws logs put-retention-policy \
  --log-group-name /ecs/bkurs-be \
  --retention-in-days 30
```

`1 / 3 / 5 / 7 / 14 / 30 / 60 / 90 / 120 / 180 / 365 / 400 / 545 / 731 / 1827 / 3653` 일 또는 무제한 (비용 폭주).

## 3. JSON 구조화 로그

Spring Boot 의 `logback-spring.xml`:

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <customFields>{"app":"bkurs-be","env":"prod"}</customFields>
      <includeMdcKeyName>requestId</includeMdcKeyName>
      <includeMdcKeyName>userId</includeMdcKeyName>
    </encoder>
  </appender>
  <root level="INFO"><appender-ref ref="STDOUT"/></root>
</configuration>
```

stdout 한 줄 = JSON 한 객체:
```json
{"@timestamp":"2026-05-22T13:00:00Z","level":"INFO","logger":"...","message":"...","app":"bkurs-be","requestId":"abc-123"}
```

CloudWatch Logs Insights 에서 즉시 query 가능:
```
fields @timestamp, level, requestId, message
| filter level = "ERROR"
| sort @timestamp desc
| limit 50
```

## 4. firelens — 외부 시스템

`firelens` 가 task 안에 **sidecar fluentbit** container 띄움. main container 의 stdout/stderr 를 받아서 다양한 백엔드로 forward.

```json
{
  "containerDefinitions": [
    {
      "name": "log-router",
      "image": "amazon/aws-for-fluent-bit:stable",
      "essential": true,
      "firelensConfiguration": {
        "type": "fluentbit",
        "options": {
          "enable-ecs-log-metadata": "true",
          "config-file-type": "file",
          "config-file-value": "/fluent-bit/etc/extra.conf"
        }
      },
      "memoryReservation": 64
    },
    {
      "name": "api",
      "image": "ghcr.io/seum-running/bkurs-be:v1",
      "essential": true,
      "logConfiguration": {
        "logDriver": "awsfirelens",
        "options": {
          "Name": "loki",
          "Host": "loki.bkurs.internal",
          "Port": "3100",
          "Labels": "{app=\"bkurs-be\",env=\"prod\"}"
        }
      }
    }
  ]
}
```

### 다중 destination

`extra.conf` (사이드카 image 에 마운트):
```
[OUTPUT]
    Name        cloudwatch_logs
    Match       *
    region      ap-northeast-2
    log_group_name /ecs/bkurs-be
    log_stream_prefix ecs/

[OUTPUT]
    Name        loki
    Match       *
    host        loki.bkurs.internal
    port        3100
    labels      app=bkurs-be,env=prod

[OUTPUT]
    Name        s3
    Match       *
    bucket      bkurs-logs
    region      ap-northeast-2
    total_file_size 50M
    upload_timeout 10m
```

→ 같은 log 가 CloudWatch + Loki + S3 동시 전송.

## 5. firelens 활용 예

### Loki + Grafana
on-prem Loki 또는 Grafana Cloud — Promtail 대신 fluentbit.

### Datadog
```
[OUTPUT]
    Name        datadog
    Match       *
    Host        http-intake.logs.datadoghq.com
    TLS         on
    apikey      ${DD_API_KEY}
    dd_service  bkurs-be
    dd_source   ecs
    dd_tags     env:prod
```

### Splunk
```
[OUTPUT]
    Name        splunk
    Match       *
    Host        splunk.bkurs.internal
    Port        8088
    Splunk_Token ${SPLUNK_TOKEN}
```

### Kinesis Data Streams → S3 / Redshift
대용량 log pipeline.

## 6. log 비용

CloudWatch Logs:
- **Ingestion**: $0.50/GB
- **Storage**: $0.03/GB·월
- **Insights query**: $0.005/GB scan
- **Data transfer**: regional 무료

비용 절감:
- Retention 30일 (또는 짧게)
- 불필요한 DEBUG 로그 회피
- Loki / S3 로 분산 (firelens)
- log filter pattern 으로 sampling

## 7. log-group 명명 컨벤션

```
/ecs/{environment}/{service}
```

예:
- `/ecs/prod/bkurs-be`
- `/ecs/staging/bkurs-be`
- `/ecs/dev/bkurs-be`

→ CloudWatch Logs Insights cross-group query 시 명확.

## 8. multiline (Java stack trace)

기본 awslogs 는 한 줄 = 한 event. Java exception 의 multi-line 이 분리됨:

```
2026-05-22 INFO Starting...        ← event 1
Caused by: NullPointerException     ← event 2 (분리됨!)
    at com.bkurs.api...              ← event 3
```

해결:
- JSON encoder 사용 (Logback) — 한 event 안에 stack 포함
- 또는 firelens + fluentbit multiline parser:
  ```
  [INPUT]
      Name tail
      Multiline.parser java
  ```

## 9. ECS task 의 logRetention CDK

```typescript
new ecs.ContainerDefinition(this, 'Api', {
  image: ...,
  logging: ecs.LogDrivers.awsLogs({
    streamPrefix: 'ecs',
    logRetention: logs.RetentionDays.ONE_MONTH,
  }),
});
```

또는 firelens:
```typescript
logging: ecs.LogDrivers.firelens({
  options: { Name: 'loki', Host: '...', Port: '3100' },
});
```

## 10. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Execution Role 에 logs:* 권한 없음 | `AmazonECSTaskExecutionRolePolicy` 또는 명시 |
| log group 자동 생성 안 됨 | `awslogs-create-group: true` 또는 사전 생성 |
| Java stack trace 가 N event 로 분리 | JSON encoder 또는 multiline parser |
| log 비용 폭주 | retention 단축 + level 조정 |
| firelens sidecar 가 OOM | memoryReservation 64MB 이상 |
| log 안 보임 (CloudWatch agent) | awslogs 는 daemon 불필요 — agent 와 다름 |

## 11. 다음

- ▶ [[08-healthcheck]]
- ▶ [[../04-operations/02-container-insights]] — metric / 통합 view
- ▶ [[../../observability/cloudwatch/cloudwatch]] — CloudWatch 일반
- ▶ [[../../../../docker/practice/16-logging-and-monitoring]] — Docker 측 Loki + Promtail

## 12. 영상 / 자료

| 자료 |
| --- |
| [FireLens + Fluent Bit — AWS](https://www.youtube.com/watch?v=PLM43JZJsmI) |
| [CloudWatch Logs Insights — AWS](https://www.youtube.com/watch?v=eXRZ8c9PpXM) |
| [Fluent Bit 공식](https://fluentbit.io/) |

## 13. 참고

- [[../ecs]]
- 공식 logConfiguration: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_awslogs.html
- aws-for-fluent-bit: https://github.com/aws/aws-for-fluent-bit
