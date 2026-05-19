---
title: "AWS CloudWatch — Metrics / Logs / Alarms"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:45:00+09:00
tags:
  - aws
  - observability
  - monitoring
  - cloudwatch
---

# AWS CloudWatch

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | CloudWatch 개념 + 사용 |

**[[observability|↑ Observability]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 의 **종합 모니터링** — metrics / logs / alarms / dashboards / events / traces (X-Ray 별도).

---

## 2. 구성

| 영역 | 의미 |
| --- | --- |
| **Metrics** | 수치 시계열 (CPU / 요청 수 / latency / queue depth) |
| **Logs** | 텍스트 / JSON / 응용 로그 |
| **Alarms** | metric threshold → 알림 |
| **Dashboard** | metric / log widget 합성 |
| **Events / EventBridge** | event-driven (별도) |
| **Insights** | log query (SQL 비슷) |
| **Synthetics** | canary 테스트 |
| **RUM** | Real User Monitoring (브라우저) |
| **Container Insights** | ECS / EKS |

---

## 3. Metrics

### 3.1 자동 metrics (AWS 자동 발생)
- EC2 — CPU, Network, Disk
- RDS — DB connections, CPU
- Lambda — Invocations, Errors, Duration
- ALB — RequestCount, TargetResponseTime
- ECS / Fargate — CPUUtilization, MemoryUtilization
- ...수백 서비스 모두

### 3.2 Custom metric

```python
import boto3
cw = boto3.client("cloudwatch")

cw.put_metric_data(
    Namespace="MyApp",
    MetricData=[{
        "MetricName": "OrdersProcessed",
        "Value": 1,
        "Unit": "Count",
        "Dimensions": [{"Name":"Service","Value":"checkout"}]
    }]
)
```

→ 응용 / 비즈니스 metric.

### 3.3 EMF (Embedded Metric Format)
JSON log 에 임베드 → 자동 metric.

```python
print(json.dumps({
    "_aws": {
        "Timestamp": int(time.time()*1000),
        "CloudWatchMetrics": [{
            "Namespace": "MyApp",
            "Dimensions": [["Service"]],
            "Metrics": [{"Name":"OrdersProcessed","Unit":"Count"}]
        }]
    },
    "Service": "checkout",
    "OrdersProcessed": 1
}))
```

→ Lambda / ECS 의 표준 — 별도 API call X.

---

## 4. Logs

### 4.1 Log Group / Stream

```
Log Group: /aws/lambda/myapp                 (log 묶음 + retention)
Log Stream: 2026/05/14/[$LATEST]xxxx          (1 source 의 stream)
```

### 4.2 응용 로그 보내기

| 방법 | |
| --- | --- |
| **stdout** | Lambda / ECS / EKS — 자동 |
| **CloudWatch Agent** | EC2 |
| **EKS** — Fluent Bit | DaemonSet |
| **SDK** | 응용에서 직접 (지양) |

### 4.3 Retention 설정 (필수!)

```hcl
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/myapp"
  retention_in_days = 30
}
```

→ default = 영구. 비용 폭증.

### 4.4 Logs Insights — query

```sql
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)
| sort @timestamp desc
| limit 100
```

→ ad-hoc 분석. Postgres-like + 시간 함수.

---

## 5. Alarms

### 5.1 Static threshold

```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "myapp-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "EC2 CPU > 80% for 2 minutes"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  dimensions = {
    InstanceId = aws_instance.app.id
  }
}
```

### 5.2 Composite alarm
여러 alarm 의 logical 조합.

### 5.3 Anomaly detection
ML 기반 동적 threshold.

```hcl
metric_query {
  id          = "anomaly"
  return_data = true
  expression  = "ANOMALY_DETECTION_BAND(m1, 2)"
}
```

→ 계절 / 패턴 인식. 정적 threshold 어려운 metric.

---

## 6. Alarm 통합

| Action | 의미 |
| --- | --- |
| **SNS** | email / SMS / Slack / Lambda |
| **EC2 Auto Recovery** | 자동 재부팅 |
| **Auto Scaling** | scale up/down |
| **Systems Manager** | runbook 자동 실행 |

```
Alarm → SNS topic → Lambda → Slack webhook
                  → email
                  → PagerDuty (외부)
```

---

## 7. Dashboard

```hcl
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "myapp-prod"
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0; y = 0; width = 12; height = 6
        properties = {
          metrics = [["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/myapp/abc"]]
          period  = 60
          stat    = "Sum"
          region  = "ap-northeast-2"
          title   = "ALB Requests"
        }
      },
      ...
    ]
  })
}
```

또는 Grafana 가 CloudWatch data source 로 더 강력.

---

## 8. Container Insights (ECS / EKS)

```bash
aws ecs update-cluster-settings --cluster prod --settings name=containerInsights,value=enabled
```

→ task / container 레벨 metric 자동 + log group `/aws/ecs/containerinsights/...`.

EKS:
```bash
helm install aws-cloudwatch-metrics aws/aws-cloudwatch-metrics --namespace amazon-cloudwatch
```

---

## 9. Synthetics — Canary

```python
import urllib3, json
def handler(event, context):
    response = urllib3.PoolManager().request("GET", "https://example.com/health")
    assert response.status == 200, f"Status: {response.status}"
```

→ 5 분마다 외부에서 endpoint check. uptime 모니터링.

---

## 10. 비용 (Seoul, 대략)

```
Custom Metric:    $0.30 / metric·월
Standard Logs:    $0.76 / GB ingest + $0.033 / GB·월 storage
Vended Logs:      더 저렴
Logs Insights query: $0.0076 / GB scanned
Alarm:            $0.10 / alarm·월 (Standard) / $0.30 (Composite)
Synthetics:       $0.0012 / canary run
```

함정:
- log retention 무제한 = storage 비용 폭증
- custom metric 너무 많이 ($/metric) = 비용
- high-resolution metric (1초) = 비싸짐

---

## 11. 외부 / 보완

- **Grafana** + CloudWatch data source — 더 좋은 시각화
- **Prometheus on EKS** + Amazon Managed Prometheus
- **Datadog / NewRelic** — APM + log 통합 (비싸지만 강력)
- **OpenTelemetry** — vendor-neutral
- **PagerDuty / Opsgenie** — on-call

---

## 12. 사용 시나리오

- 모든 AWS 자원 (기본)
- Lambda / ECS log
- 비즈니스 metric (custom)
- alerting (SNS → email/slack/PagerDuty)
- Auto Scaling (metric 기반)
- canary test
- 보안 alarm (root login, failed auth, ...)

---

## 13. 함정

### 13.1 log retention 무제한
storage 비용. 모든 log group 에 retention 설정.

### 13.2 custom metric 폭증
metric 마다 $0.30. dimension 조합도 별도. cardinality 통제.

### 13.3 alarm "INSUFFICIENT_DATA"
metric 안 흐를 때. `treat_missing_data` 옵션.

### 13.4 1초 resolution
보통 X (5/15초). 명시할 때만 + 더 비쌈.

### 13.5 logs 의 cost
ingest 가 가장 큰 비용. 응용 로그 — 디버그 logs filter / sample.

### 13.6 alarm 의 evaluation period
너무 짧음 = false alarm. 너무 김 = 늦은 대응. 1-5 분 이상 권장.

### 13.7 dashboard 의 한계
복잡 → Grafana.

---

## 14. 학습 자료

- AWS CloudWatch docs
- **Brendan Gregg — Observability** (AWS 무관도 좋음)
- **Datadog / Grafana** 도 학습 추천

---

## 15. 관련

- [[observability]] — Observability hub
- [[cloudtrail]] — audit log
- [[lambda]] — 자동 log
- [[sns]] — Alarm action
- [[monitoring|↗ Linux 모니터링]]
