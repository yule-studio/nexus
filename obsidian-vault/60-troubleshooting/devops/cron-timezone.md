---
title: "troubleshooting · cron / EventBridge TZ 불일치"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, cron, eventbridge, timezone, schedule]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[../backend/timezone-bug]]"]
---

# cron 가 한국 시간 기준이 아닌 UTC 로 실행

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

- cron `0 9 * * *` 인데 한국 18시에 실행
- 일일 정산 batch 가 다음날 새벽 실행
- AWS EventBridge cron 가 의도와 9시간 차이

## 원인

container / EventBridge cron 의 기본 TZ 가 UTC. 한국 기준으로 작성한 cron 이 9시간 어긋남.

```
container TZ = UTC
cron "0 9 * * *" = UTC 09:00 = KST 18:00
```

## 해결

### Container TZ 변경 (Docker)
```dockerfile
RUN apt-get install -y tzdata && \
    ln -snf /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone
ENV TZ=Asia/Seoul
```

또는 Alpine:
```dockerfile
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone
```

### crontab 직접 TZ
```cron
CRON_TZ=Asia/Seoul
0 9 * * * /scripts/daily-report.sh
```

→ 해당 line 만 KST 기준.

### Kubernetes CronJob — timeZone field
```yaml
apiVersion: batch/v1
kind: CronJob
spec:
  schedule: "0 9 * * *"
  timeZone: "Asia/Seoul"   # K8s 1.27+ stable
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: report
              image: myapp:latest
```

### AWS EventBridge — UTC 통일 권장
```
# KST 매일 09:00 = UTC 매일 00:00
cron(0 0 * * ? *)

# KST 평일 09:00 = UTC 평일 00:00
cron(0 0 ? * MON-FRI *)
```

→ EventBridge 는 TZ 옵션 없음 (UTC fixed). 변환 표 만들기:

| KST | UTC | EventBridge cron |
| --- | --- | --- |
| 09:00 | 00:00 | `cron(0 0 * * ? *)` |
| 12:00 | 03:00 | `cron(0 3 * * ? *)` |
| 18:00 | 09:00 | `cron(0 9 * * ? *)` |
| 00:00 | 15:00 (전날) | `cron(0 15 * * ? *)` |

→ KST 자정 정산은 UTC 15:00 (전날 기준). 헷갈리기 쉬움.

### Application 측 일일 정산 — TZ 명시
```java
@Scheduled(cron = "0 0 9 * * *", zone = "Asia/Seoul")
public void dailyReport() { ... }
```

```python
from apscheduler.schedulers.blocking import BlockingScheduler
sched = BlockingScheduler(timezone='Asia/Seoul')
@sched.scheduled_job('cron', hour=9)
def daily_report(): ...
```

### Spring `@Scheduled` zone
```yaml
spring:
  task:
    scheduling:
      timezone: Asia/Seoul
```

## DST (Daylight Saving Time) 함정

한국 / 일본은 DST 없음. 그러나 미국 / 유럽 사용자 / 서버 운영 시:
- DST 전환 시점 (3월 / 11월) 의 "spring forward" 시간은 cron 실행 안 되거나 2번 실행
- KST 가 안전 — TZ change 없음

## 진단

```bash
# Container 의 시간 / TZ
date                     # 현재 시간
date +%Z                 # TZ 약어 (KST / UTC)
cat /etc/timezone
ls -l /etc/localtime

# cron 의 다음 실행 (croniter)
python -c "from croniter import croniter; from datetime import datetime; print(croniter('0 9 * * *', datetime.now()).get_next(datetime))"

# AWS EventBridge — 다음 실행
aws events list-rules
aws events describe-rule --name myrule
```

## 재발 방지

- **container TZ 표준 (UTC 또는 Asia/Seoul) 명시** — base image 에서 시작
- EventBridge 는 UTC + 변환 표 PR template 첨부
- Application scheduler 의 `zone` 명시 의무 (lint)
- 모니터링: 정산 job 의 실행 시간 / 실행 횟수 검증

## 관련

- [[devops]] / [[../backend/timezone-bug]]
