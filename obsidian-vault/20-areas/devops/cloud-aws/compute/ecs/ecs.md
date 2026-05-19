---
title: "AWS ECS — Elastic Container Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:20:00+09:00
tags:
  - aws
  - compute
  - ecs
  - container
---

# AWS ECS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | ECS 개념 + Fargate + 사용 |

**[[compute|↑ Compute]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 의 **container orchestrator**. K8s 와 같은 의도 + AWS native + 단순.
**Fargate launch** 로 server 관리 자체 X (진정한 serverless container).

---

## 2. 왜

- **container 표준** — Docker / OCI
- K8s = 표준이지만 복잡 / 운영 부담
- ECS = AWS 통합 + 단순 + 학습 곡선 ↓
- Fargate = VM 관리 X — "container 만 신경"

대안:
- **EKS** — K8s 표준 (이식성, 멀티 cloud)
- **Lambda** — 짧은 / event-driven
- **App Runner** — git push → 자동 빌드 / 배포 (가장 단순)

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Cluster** | 논리 그룹 |
| **Task Definition** | container spec (image, CPU, RAM, env, mount, role) |
| **Task** | 실행 중인 container 인스턴스 |
| **Service** | desired count 유지 + LB 통합 |
| **Capacity Provider** | Fargate / EC2 / Spot |
| **Launch Type** | Fargate / EC2 |

---

## 4. Fargate vs EC2 launch

| | Fargate | EC2 launch |
| --- | --- | --- |
| Server 관리 | 없음 | EC2 직접 |
| 가격 | task 분 단위 (vCPU/RAM) | EC2 시간 + container 무료 |
| 빠른 시작 | 30-60s | 빠름 (이미 띄움) |
| GPU | 제한 | 가능 |
| 권장 | 일반 응용 | bin-packing / GPU |

→ 거의 모든 신규 = **Fargate**.

---

## 5. 설치 / 첫 service

### 5.1 ECR push (image)

```bash
# 인증
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS --password-stdin \
  111222333444.dkr.ecr.ap-northeast-2.amazonaws.com

# repo 생성
aws ecr create-repository --repository-name myapp

# build + push
docker build -t myapp .
docker tag myapp:latest 111222333444.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:v1
docker push 111222333444.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:v1
```

### 5.2 Task Definition

```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::111222333444:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "111222333444.dkr.ecr.ap-northeast-2.amazonaws.com/myapp:v1",
      "essential": true,
      "portMappings": [{ "containerPort": 8080, "protocol": "tcp" }],
      "environment": [
        { "name": "DB_HOST", "value": "rds-host" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "ap-northeast-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### 5.3 CLI

```bash
aws ecs register-task-definition --cli-input-json file://taskdef.json

aws ecs create-cluster --cluster-name prod

aws ecs create-service \
  --cluster prod \
  --service-name myapp \
  --task-definition myapp:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx]}" \
  --load-balancers "targetGroupArn=arn:...,containerName=app,containerPort=8080"
```

---

## 6. Service Auto Scaling

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/prod/myapp \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 20

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/prod/myapp \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    }
  }'
```

---

## 7. Deployment

| Strategy | 의미 |
| --- | --- |
| **Rolling Update** (default) | 점진 교체 |
| **Blue/Green** (CodeDeploy) | 두 환경 + traffic switch |
| **External (CI/CD)** | GitHub Actions / ArgoCD 가 직접 update |

```bash
# image 변경 → service update
aws ecs update-service --cluster prod --service myapp --force-new-deployment
```

---

## 8. 비용 (Seoul)

```
Fargate: $0.04048 / vCPU·시간 + $0.004445 / GB·시간

예: 0.5 vCPU + 1 GB × 24h × 30 = ~$18/월/task
+ data transfer + LB + ECR
```

Savings Plan 적용 시 ~ 50%. Spot Fargate 도 (75% 할인).

---

## 9. 사용 시나리오

- 웹 서버 / API (Spring / Node / Python)
- 백그라운드 worker
- batch job (Scheduled Task)
- 마이크로서비스

VS EKS — 표준 / 이식성 우선이면 EKS. AWS 통합 / 단순 = ECS.

---

## 10. 함정

### 10.1 task role vs execution role
- **executionRole** — ECS 가 image / log 권한 (기본)
- **taskRole** — container 안에서 AWS 호출 (S3 read 등)

### 10.2 awsvpc network mode
task 마다 ENI 1 개 → subnet IP 소모 ↑. 작은 subnet 에서 빠르게 고갈.

### 10.3 Fargate 의 ephemeral storage
20 GB 기본 (200 GB 까지). image + 임시. 영구 데이터는 EFS mount.

### 10.4 health check 까다로움
container health check + ALB target health 둘 다.

### 10.5 graceful shutdown
SIGTERM 후 default 30 s — `stopTimeout` 으로 늘림.

### 10.6 image pull 시간
큰 image (1 GB+) = task 시작 느림. multi-stage build / distroless.

### 10.7 cost — 작은 task 많이
task 마다 최소 0.25 vCPU 과금. 묶을 수 있으면 1 task 안에.

---

## 11. EKS vs ECS

| | ECS | EKS |
| --- | --- | --- |
| Orchestrator | AWS 자체 | Kubernetes |
| 학습 | 쉬움 | 복잡 |
| 이식성 | AWS only | 표준 K8s |
| AWS 통합 | 매우 강 | 좋음 |
| 운영 | 간단 | 복잡 |
| 생태계 | AWS | CNCF 전체 |
| 비용 | 작음 | EKS control plane $73/월 |

EKS 자세히 → [[kubernetes]]

---

## 12. 학습 자료

- AWS ECS docs
- **AWS Containers Roadmap** (github)
- **ECS Workshop** — ecsworkshop.com

---

## 13. 관련

- [[compute]] — Compute hub
- [[ec2]] — VM
- [[lambda]] — Serverless
- [[kubernetes]] — EKS
