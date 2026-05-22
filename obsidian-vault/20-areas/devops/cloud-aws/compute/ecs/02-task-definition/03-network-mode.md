---
title: "ECS 02-3 — Network Mode (awsvpc / bridge / host / none)"
kind: knowledge
status: current
created_at: 2026-05-22T13:34:00+09:00
tags: [aws, ecs, task-definition, network]
home_hub: ecs
---

# Task Definition — Network Mode

> task 의 networking 모델 4 종. Fargate 는 awsvpc 강제. EC2 launch 는 4 종 모두 가능.
> **awsvpc** 가 표준 + ALB IP target + 보안 그룹 가능.

## 1. 4 종 비교

| Mode | Fargate | EC2 | IP 모델 | 권장 |
| --- | --- | --- | --- | --- |
| **awsvpc** | ✓ (강제) | ✓ | task 마다 ENI + IP | ★ 표준 |
| **bridge** | ✗ | ✓ | host port → container port NAT | legacy / port mapping 동적 |
| **host** | ✗ | ✓ | host network 직접 공유 | 성능 critical |
| **none** | ✗ | ✓ | network 없음 | 격리 / 빌드 task |

## 2. awsvpc — 표준

```json
{
  "networkMode": "awsvpc",
  "containerDefinitions": [{
    "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }]
  }]
}
```

- task 마다 별도 **ENI** (Elastic Network Interface) — 별도 IP / SG
- VPC subnet 의 IP 1개 소비
- ALB / NLB 가 IP target 으로 직접 등록
- 보안 그룹 task 단위 — fine-grained

Service 의 network configuration:
```json
{
  "awsvpcConfiguration": {
    "subnets": ["subnet-aaa", "subnet-bbb"],
    "securityGroups": ["sg-ccc"],
    "assignPublicIp": "DISABLED"     // private subnet 일 때 (권장)
  }
}
```

### ENI trunking (EC2 only)

EC2 instance 의 ENI 한도 (보통 type 별 5~15) 가 task 수 한도가 됨. Trunking 활성 시 더 많은 task:

```bash
aws ec2 modify-instance-attribute --instance-id i-xxx \
  --no-source-dest-check    # 별도 활성 절차
```

자세히: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-eni.html

## 3. bridge — EC2 legacy

```json
{
  "networkMode": "bridge",
  "containerDefinitions": [{
    "portMappings": [
      { "containerPort": 8080, "hostPort": 0 }       // 0 = 동적 (32768~65535)
    ]
  }]
}
```

- container 가 `docker0` bridge 에서 IP 받음 (172.17.x.x)
- host port → container port NAT
- `hostPort: 0` → ECS 가 자동 random port — bin-packing 용이
- ALB 가 instance target — 동적 port 자동 등록

장점: 한 EC2 에 같은 image 의 task 여러개 packing 가능 (host port 충돌 없음).

> 단점: SG 가 host 단위 — task 단위 격리 X. 대부분 신규는 awsvpc 권장.

## 4. host — 직접 공유

```json
{
  "networkMode": "host"
}
```

- container 가 host network namespace 직접 사용
- `containerPort = hostPort` (mapping 없음, NAT 없음)
- 성능 ★★★ (NAT overhead 0)
- 한 host 에 같은 port 의 task 1개만

> 활용: 성능 critical, low-latency. 격리 ↓ — 보안 주의.

## 5. none — 격리

```json
{
  "networkMode": "none"
}
```

- network 없음 (lo 만)
- 활용: build / compute / batch task — 외부 통신 불필요

## 6. Fargate 의 awsvpc 강제

```json
{
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc"       // 다른 값이면 register 실패
}
```

Fargate microVM 의 격리 모델이 awsvpc 만 지원. 나머지 mode 사용 시 `register-task-definition` 거부.

## 7. service network configuration

Service create / update 시:
```bash
aws ecs create-service \
  --cluster bkurs-prod \
  --service-name api \
  --task-definition bkurs-be:42 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
      subnets=[subnet-aaa,subnet-bbb,subnet-ccc],
      securityGroups=[sg-ddd],
      assignPublicIp=DISABLED
  }"
```

- **subnets** — 다중 AZ 권장 (3개)
- **securityGroups** — 1+
- **assignPublicIp** — private subnet + NAT Gateway 가 표준 (ENABLED 는 비용 + 보안 위험)

## 8. private subnet + NAT Gateway

```
Internet
   │
   ▼
Public Subnet ─ NAT Gateway
   │
   ▼ (outbound only)
Private Subnet ─ ECS task (awsvpc)
                    │
                    ▼
                  RDS / S3 / SSM
```

ECS task 가 ECR pull / SSM 조회 / 외부 API 호출 가능 (NAT outbound).

**VPC Endpoints** 사용 시 NAT 비용 ↓:
- `com.amazonaws.{region}.ecr.api`, `.ecr.dkr` — ECR
- `com.amazonaws.{region}.s3` (Gateway endpoint) — S3
- `com.amazonaws.{region}.ssm`, `.ssmmessages`, `.ec2messages` — ECS Exec
- `com.amazonaws.{region}.logs` — CloudWatch Logs

## 9. 보안 그룹 — task vs LB

```
Internet
   │
   ▼
ALB SG (sg-alb) ─ Inbound 443 from 0.0.0.0/0
   │
   ▼
Task SG (sg-task) ─ Inbound 8080 from sg-alb only
```

> Task SG 의 Inbound 가 **ALB SG** 만 허용 (CIDR 아님) — task IP 변경에도 정책 유지.

## 10. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Fargate 의 networkMode = bridge | awsvpc 만 가능 |
| Subnet IP 부족 — task 시작 fail | subnet 크기 / multi-AZ |
| ENI 한도 (account / region) | quota 증액 |
| ALB Target Group type = instance 인데 awsvpc | IP target group 으로 |
| SG inbound CIDR 0.0.0.0/0 | LB SG 만 허용 — 보안 ↓ |
| public IP 켰는데 NAT 없음 | private subnet 권장 |
| ECR pull fail (private subnet) | VPC Endpoint 또는 NAT |

## 11. 다음

- ▶ [[04-volumes-and-efs]] — 영구 storage
- ▶ [[06-iam-roles]]
- ▶ [[../03-service/05-load-balancer-integration]] — ALB / NLB IP target
- ▶ [[../05-security/02-network-isolation]] — 보안 / SG 깊이
- ▶ [[../../network/vpc/vpc]] — VPC 일반

## 12. 영상 / 자료

| 자료 |
| --- |
| [ECS networking — re:Invent](https://www.youtube.com/watch?v=icQUg6JnICQ) |
| [awsvpc + ENI trunking — AWS blog](https://aws.amazon.com/blogs/containers/maximize-your-amazon-elastic-container-service-cluster-density-with-amazon-ecs-managed-instances/) |

## 13. 참고

- [[../ecs]]
- 공식: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-networking.html
