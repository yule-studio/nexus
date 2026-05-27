---
title: "ECS 04-2 — Container Insights (CloudWatch metric / 대시보드 / alarm)"
kind: knowledge
status: current
created_at: 2026-05-22T14:02:00+09:00
tags: [aws, ecs, operations, monitoring, cloudwatch]
home_hub: ecs
---

# Container Insights

> CloudWatch Container Insights — cluster / service / task / container 통합 metric. cluster 당 $1.50/일.

## 1. 활성

```bash
aws ecs update-cluster-settings --cluster bkurs-prod \
  --settings name=containerInsights,value=enabled
```

CDK: `containerInsights: true`.

## 2. 메트릭

### Cluster
- CPUUtilization / MemoryUtilization
- RunningTaskCount / PendingTaskCount

### Service
- CPUUtilization / MemoryUtilization
- RunningTaskCount / DesiredTaskCount
- DeploymentCount

### Task (Performance Mode 추가 비용)
- TaskId × Container
- NetworkRxBytes / NetworkTxBytes
- StorageReadBytes / StorageWriteBytes

## 3. CloudWatch 대시보드

자동 widget — Console > CloudWatch > Container Insights:
- CPU / Memory utilization
- Network IO / Disk IO
- Running tasks / Service health

## 4. CloudWatch Logs Insights query

```
# 5xx
fields @timestamp, @message
| filter @message like /5[0-9]{2}/
| stats count() by bin(5m)

# requestId 추적
fields @timestamp, requestId, level, message
| filter requestId = "abc-123"
| sort @timestamp

# OOM
fields @timestamp, @message
| filter @message like /OutOfMemoryError/

# slow
fields @timestamp, uri, status, took_ms
| filter took_ms > 2000
| sort took_ms desc | limit 20
```

## 5. Alarm — bkurs-be 표준

```bash
# 5xx
aws cloudwatch put-metric-alarm --alarm-name bkurs-be-5xx \
  --metric-name HTTPCode_Target_5XX_Count --namespace AWS/ApplicationELB \
  --statistic Sum --period 60 --evaluation-periods 5 \
  --threshold 10 --comparison-operator GreaterThanThreshold

# CPU 80%
aws cloudwatch put-metric-alarm --alarm-name bkurs-be-cpu-80 \
  --metric-name CPUUtilization --namespace AWS/ECS \
  --statistic Average --period 300 --evaluation-periods 3 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --dimensions Name=ServiceName,Value=api Name=ClusterName,Value=bkurs-prod
```

## 6. EventBridge

ECS task 비정상 종료 → Lambda → 알림:
```yaml
EventPattern:
  source: [aws.ecs]
  detail-type: ["ECS Task State Change"]
  detail:
    clusterArn: [arn:...:cluster/bkurs-prod]
    lastStatus: [STOPPED]
    stoppedReason: [{ anything-but: "Essential container in task exited" }]
Targets:
  - Arn: arn:lambda:...:function:notify-slack
```

## 7. App metric

### EMF (Embedded Metric Format)
log 한 줄 → 자동 metric:
```json
{
  "_aws": {
    "Timestamp": 1716365400000,
    "CloudWatchMetrics": [{
      "Namespace": "Bkurs",
      "Metrics": [{ "Name": "OrderProcessed", "Unit": "Count" }],
      "Dimensions": [["Service"]]
    }]
  },
  "Service": "api",
  "OrderProcessed": 1
}
```

### PutMetricData
```java
cw.putMetricData(...);  // Task Role 의 cloudwatch:PutMetricData
```

### ADOT (OpenTelemetry)
Spring Boot `/actuator/prometheus` → ADOT Collector sidecar → CloudWatch / AMP.

## 8. 비용

| 항목 | 가격 |
| --- | --- |
| Container Insights | $1.50/cluster/일 |
| Custom metric | $0.30/metric/월 |
| Logs ingestion | $0.50/GB |
| Logs storage | $0.03/GB·월 |

절감:
- Performance Mode 끔 (task metric 안 수집)
- Log retention 7~30일
- query 시 시간 범위 좁힘

## 9. Grafana 통합

```
ECS → awslogs → CloudWatch → Grafana data source
```

또는 firelens → Loki + Prometheus → Grafana.

## 10. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Container Insights OFF | cluster settings |
| 비용 폭주 | Performance Mode 끔 + retention |
| custom metric 권한 부족 | Task Role cloudwatch:PutMetricData |
| EventBridge rule trigger 안 됨 | event pattern 검증 |
| Alarm INSUFFICIENT_DATA | datapointsToAlarm 조정 |

## 11. 다음

- ▶ [[03-troubleshooting]]
- ▶ [[04-cost-optimization]]
- ▶ [[../../observability/cloudwatch/cloudwatch]]

## 12. 영상 / 자료

| 자료 |
| --- |
| [Container Insights — AWS](https://www.youtube.com/watch?v=LyKgIaH5fjQ) |
| [공식](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html) |

## 13. 참고

- [[../ecs]]
- [[../02-task-definition/07-logging]]
