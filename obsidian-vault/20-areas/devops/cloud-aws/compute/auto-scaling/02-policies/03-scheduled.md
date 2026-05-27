---
title: "Auto Scaling 02-3 — Scheduled Actions"
kind: knowledge
status: current
created_at: 2026-05-22T18:14:00+09:00
tags: [aws, auto-scaling, scheduled, cron, peak-hour]
home_hub: auto-scaling
---

# Scheduled Actions

> cron / one-time. 평일 peak 시간 미리 늘리기 / 야간 줄이기 / 이벤트 대응.

## 1. EC2 ASG Scheduled

```bash
# 매일 09:00 KST → min 10
aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name lab-asg \
  --scheduled-action-name morning-peak \
  --recurrence "0 9 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 10 --max-size 30 --desired-capacity 10

# 매일 21:00 KST → min 2
aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name lab-asg \
  --scheduled-action-name night-rest \
  --recurrence "0 21 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 2 --max-size 10 --desired-capacity 2
```

| 옵션 | 의미 |
| --- | --- |
| `recurrence` | cron 형식 (UTC default, time-zone 지정) |
| `start-time` / `end-time` | one-time 일정 |
| `min-size` / `max-size` / `desired-capacity` | 각 시점의 값 |

## 2. Application Auto Scaling (ECS / Lambda / DDB)

```bash
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/bkurs-prod/api \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name peak \
  --schedule "cron(0 9 ? * MON-FRI *)" \
  --timezone "Asia/Seoul" \
  --scalable-target-action "MinCapacity=10,MaxCapacity=30"

aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/bkurs-prod/api \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name rest \
  --schedule "cron(0 21 ? * MON-FRI *)" \
  --timezone "Asia/Seoul" \
  --scalable-target-action "MinCapacity=3,MaxCapacity=10"
```

## 3. cron 형식 (EventBridge)

```
cron(분 시 일 월 요일 년)

cron(0 9 ? * MON-FRI *)       매주 평일 09:00
cron(0 21 ? * MON-FRI *)      매주 평일 21:00
cron(0 0 1 * ? *)             매월 1일 00:00
cron(0 0/30 * * ? *)          30분마다
cron(0 9 * * SUN *)            매주 일요일 09:00
rate(5 minutes)                매 5분
```

> 주의 — `?` 는 day-of-week 또는 day-of-month 중 하나만 사용.

## 4. One-time Action

```bash
# 2026-12-31 18:00 KST 새해 이벤트
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/bkurs-prod/api \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name new-year-event \
  --schedule "at(2026-12-31T09:00:00)" \
  --timezone "UTC" \
  --scalable-target-action "MinCapacity=50,MaxCapacity=100"
```

## 5. dev/staging shutdown

```bash
# 매일 야간 22:00 → 0 (off)
aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name dev-asg \
  --scheduled-action-name night-off \
  --recurrence "0 22 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 0 --max-size 0 --desired-capacity 0

# 매일 08:00 → resume
aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name dev-asg \
  --scheduled-action-name morning-resume \
  --recurrence "0 8 * * MON-FRI" --time-zone "Asia/Seoul" \
  --min-size 1 --max-size 3 --desired-capacity 1
```

→ dev/staging 야간+주말 비용 ~67% 절감.

## 6. Scheduled + Target Tracking 결합

```
Scheduled (09:00): min=10, max=30          ← baseline 변경
Target Tracking:   CPU 60% target          ← desired 자동 (10-30 안)
```

→ baseline 미리 늘리고, 실제 부하 따라 reactive 보정.

## 7. List / Update / Delete

```bash
# List
aws autoscaling describe-scheduled-actions --auto-scaling-group-name lab-asg

# Update
aws autoscaling put-scheduled-update-group-action ...  # 같은 이름 = update

# Delete
aws autoscaling delete-scheduled-action --auto-scaling-group-name lab-asg \
  --scheduled-action-name night-off
```

## 8. CloudWatch alarm + Scheduled

Scheduled action 의 결과 / 실패 alarm:
```
ASG metric: GroupDesiredCapacity vs GroupInServiceInstances
```

→ Scheduled 실행 후 capacity 가 안 따라오면 alarm.

## 9. 타임존 주의

```
recurrence:  "0 9 * * MON-FRI"
time-zone:  Asia/Seoul         (default UTC)

= KST 09:00 = UTC 00:00
```

> time-zone 안 명시 시 UTC. 한국 작업 시 항상 KST 명시.

## 10. EventBridge Scheduler (modern 대안)

EventBridge Scheduler — Scheduled Action 의 후속:
```bash
aws scheduler create-schedule --name bkurs-peak \
  --schedule-expression "cron(0 9 ? * MON-FRI *)" \
  --schedule-expression-timezone "Asia/Seoul" \
  --target '{
    "Arn": "arn:aws:scheduler:::aws-sdk:applicationautoscaling:registerScalableTarget",
    "RoleArn": "arn:iam::...:role/scheduler-role",
    "Input": "{\"ServiceNamespace\":\"ecs\",\"ResourceId\":\"service/bkurs-prod/api\",\"ScalableDimension\":\"ecs:service:DesiredCount\",\"MinCapacity\":10,\"MaxCapacity\":30}"
  }' \
  --flexible-time-window 'Mode=OFF'
```

장점:
- 1M schedule / month free
- 모든 AWS API trigger 가능
- one-time + recurring

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| time-zone UTC 그대로 → KST 9h 늦음 | --time-zone Asia/Seoul |
| Scheduled 가 Target Tracking 위에 충돌 | Scheduled = min/max / TT = desired |
| desired 명시 안 함 → TT 가 결정 | desired 명시 또는 생략 (TT 만) |
| One-time 후 미정리 | start-time end-time 명시 |
| Peak 시간 직전 부족 → cold start | SchedulingBufferTime |

## 12. 다음

- ▶ [[../03-operations/01-lifecycle-hooks-and-warm-pool]]
- ▶ [[../03-operations/02-troubleshooting]]
- ▶ [[../practice/01-asg-with-mixed-instances]]

## 13. 영상 / 자료

| 자료 |
| --- |
| [Scheduled Scaling 공식](https://docs.aws.amazon.com/autoscaling/application/userguide/application-auto-scaling-scheduled-scaling.html) |
| [EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/) |

## 14. 참고

- [[../auto-scaling]]
- [[01-target-tracking]]
- [[02-step-and-simple]]
