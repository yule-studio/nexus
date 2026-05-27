---
title: "ECS 실습 1 — Fargate 첫 service (minimum viable production baseline)"
kind: practice
status: current
created_at: 2026-05-22T14:20:00+09:00
updated_at: 2026-05-26T18:45:00+09:00
tags: [aws, ecs, practice, fargate, alb]
home_hub: ecs
---

# 실습 1 — Fargate 첫 service

## 1. 목적

- 첫 ECS Fargate service 의 **minimum viable production baseline** 체험.
- 각 step 책임 분리 — ECR / Execution Role / Log group / Cluster / TD / ALB / Service.
- 학습용 `assignPublicIp=ENABLED` 와 production 의 private subnet + NAT 차이.
- ALB IP target group + awsvpc 결합 mechanism.
- enableExecuteCommand + Circuit Breaker + Container Insights 의 baseline.
- 30분 안 완료 가능한 흐름.

## 2. 범위

| 다룬다 | 다루지 않는다 |
| --- | --- |
| Fargate 첫 service 30분 실습 | EC2 launch type → [[02-ec2-with-asg]] |
| ECR / IAM / Cluster / TD / ALB / Service 흐름 | Blue-Green → [[03-blue-green-with-codedeploy]] |
| ARM64 image build / push | Spring Boot end-to-end → [[04-spring-boot-end-to-end]] |
| awsvpc + ALB IP target | Service Connect mesh → [[../03-service/04-service-discovery-and-connect]] |
| ECS Exec / Circuit Breaker | Auto scaling 깊이 → [[../03-service/03-auto-scaling]] |
| Clean up | |

## 3. 사전 지식

- [[../01-fundamentals/01-architecture]] §4.1 — 4 추상.
- [[../02-task-definition/01-overview]] §4.1 — TD 4 영역.
- [[../03-service/01-service-overview]] §4.4 — Service 옵션.
- [[../03-service/05-load-balancer-integration]] §4.2 — IP target.

---

## 4. 작동 원리 — 왜 각 step 인가

### 4.1 minimum viable production 의 7 요소

```
1. ECR repository       — image 보관 (IMMUTABLE)
2. Execution Role       — ECS agent 가 image pull + log put
3. Log Group            — CloudWatch Logs (retention 7일)
4. Cluster              — 논리 namespace + Container Insights
5. Task Definition      — container spec (Fargate awsvpc ARM64)
6. ALB + Target Group   — HTTPS endpoint + IP target
7. Service              — desiredCount + LB binding + Circuit Breaker
```

**왜 7 요소 모두 필수**:

| 빠지면 | 결과 |
| --- | --- |
| ECR | image 어디 보관? Docker Hub rate limit |
| Execution Role | pull 실패 (ResourceInitializationError) |
| Log Group | log 안 보임 (debugging 불가) |
| Cluster | 자원 그룹 0 — Service 등록 불가 |
| TD | container spec 없음 |
| ALB | 외부 접근 불가 (public IP 직접 안 권장) |
| Service | desiredCount 유지 안 됨 (task fail = down) |

→ **30분 실습이지만 production 모델 그대로**. 후속 실습은 위 baseline 위에 추가.

### 4.2 학습용 vs production 차이

```
[학습용 (이 실습)]
- assignPublicIp: ENABLED       (public subnet 의 task)
- HTTP 80 (HTTPS 없음)
- single AZ OK
- desiredCount 2 (HA 의 최소)

[production]
- assignPublicIp: DISABLED      (private subnet + NAT)
- HTTPS 443 + ACM Certificate
- multi-AZ 3 subnet
- desiredCount 3+ + Auto Scaling
- VPC Endpoint (NAT 비용 절감)
- WAF + Shield
- Secrets Manager + KMS
- Service Connect (internal mesh)
```

**왜 학습용은 단순화**:

- 30분 안 완료 — VPC Endpoint setup 만 30분
- learning curve 단축
- 개념 우선

### 4.3 ALB IP target + awsvpc 결합

```
[awsvpc + IP target]
ALB Target Group type=IP
   │
   ▼
ECS Service register
   │ task ENI 의 IP 등록
   ▼
ALB → task IP (직접)

[bridge + Instance target (legacy)]
ALB Target Group type=Instance
   │
   ▼
EC2 instance + dynamic port (32768-65535)
   │
   ▼
ALB → EC2 IP + dynamic port → bridge NAT → container
```

→ awsvpc + IP target = 한 hop 적음, SG per task, 표준.

### 4.4 ECS Service ↔ Target Group 자동 binding

```
[Service create 시]
- loadBalancers: targetGroupArn + containerName + containerPort
- ECS 가 task launch 마다 IP register
- task stop = deregister + drain (30s)

[운영자 개입]
- Target Group 에 IP 직접 등록 X
- ALB rule 만 작성 (path/host routing)
```

### 4.5 enableExecuteCommand + Circuit Breaker

```bash
--enable-execute-command \
--deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true},maximumPercent=200,minimumHealthyPercent=100"
```

**왜 처음부터**:

- **ECS Exec** = production container shell 접근 (SSH 부재)
- **Circuit Breaker** = bad deployment 자동 rollback
- **200/100** = zero-downtime rolling (capacity 잠시 2x)

→ 나중에 추가 어려움 — task definition 다시 register + force-new-deployment.

### 4.6 Health Check + Grace Period

```json
{
  "healthCheck": {
    "command": ["CMD-SHELL","wget -q -O- http://localhost:8080/health || exit 1"],
    "interval": 30, "timeout": 5, "retries": 3
  }
}
```

```bash
--health-check-grace-period-seconds 60
```

**왜 둘 다**:

- Container healthCheck = task lifecycle (unhealthy → kill)
- ALB health check = traffic routing
- Grace period = cold start 보호 (nginx 60s 충분)

자세히 → [[../02-task-definition/08-healthcheck]] §4.1.

---

## 5. 안 쓰면 어떤 문제 / 대안 / 트레이드오프

### 5.1 ECR 안 쓰고 Docker Hub

- rate limit (100 pull/6h, IP 기반)
- 같은 region pull 안 됨 → 인터넷 latency
- 권장: ECR

### 5.2 Execution Role 없이

- task launch 실패 (image pull / log put)
- 권장: AmazonECSTaskExecutionRolePolicy 시작

### 5.3 ALB 없이 public IP 직접

- HTTPS 종료 따로 (Nginx in container?)
- multi-AZ 분배 안 됨
- 권장: ALB

### 5.4 Container Insights 비활성

- metric / dashboard 0
- alarm 없음
- 권장: 시작부터 활성 ($1.50/cluster/일)

### 5.5 Circuit Breaker 없이

- bad deployment 무한 retry
- 권장: 처음부터 활성

### 5.6 production 패턴 비교

- VPC Endpoint (NAT 비용 절감)
- HTTPS (ACM + ALB)
- multi-AZ subnet
- WAF + Shield
- Secrets Manager + KMS

→ 모두 [[04-spring-boot-end-to-end]] 의 패턴.

---

## 6. 구체 예시 — 실습 명령

### 6.1 사전 준비

- AWS CLI 설정 + region `ap-northeast-2`
- VPC 1개 (default OK) + public subnet 2개 이상
- session-manager-plugin

### 6.2 ECR repository

```bash
aws ecr create-repository --repository-name echo-app \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE
```

### 6.3 Image build + push

```bash
mkdir echo && cd echo

cat > Dockerfile <<'EOF'
FROM nginxinc/nginx-unprivileged:1.27-alpine
COPY default.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
EOF

cat > default.conf <<'EOF'
server {
  listen 8080;
  location /health { return 200 'ok'; add_header Content-Type text/plain; }
  location / { return 200 'hello from ecs\n'; add_header Content-Type text/plain; }
}
EOF

ACCT=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-2

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCT.dkr.ecr.$REGION.amazonaws.com

docker buildx build --platform linux/arm64 \
  -t $ACCT.dkr.ecr.$REGION.amazonaws.com/echo-app:v1 --push .
```

### 6.4 IAM Execution Role

```bash
aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### 6.5 Log Group

```bash
aws logs create-log-group --log-group-name /ecs/echo-app
aws logs put-retention-policy --log-group-name /ecs/echo-app --retention-in-days 7
```

### 6.6 Cluster + Container Insights

```bash
aws ecs create-cluster --cluster-name demo \
  --capacity-providers FARGATE FARGATE_SPOT \
  --settings name=containerInsights,value=enabled
```

### 6.7 Task Definition (ARM64 Fargate)

```bash
cat > taskdef.json <<EOF
{
  "family": "echo-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "runtimePlatform": { "cpuArchitecture": "ARM64", "operatingSystemFamily": "LINUX" },
  "cpu": "256", "memory": "512",
  "executionRoleArn": "arn:aws:iam::$ACCT:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "app",
    "image": "$ACCT.dkr.ecr.$REGION.amazonaws.com/echo-app:v1",
    "essential": true,
    "portMappings": [{"containerPort": 8080, "protocol": "tcp", "name": "http"}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/echo-app",
        "awslogs-region": "$REGION",
        "awslogs-stream-prefix": "app"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL","wget -q -O- http://localhost:8080/health || exit 1"],
      "interval": 30, "timeout": 5, "retries": 3
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file://taskdef.json
```

### 6.8 ALB + Target Group + Listener

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC --query 'Subnets[].SubnetId' --output text)

SG_ALB=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name alb-sg --description "alb" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ALB \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

SG_TASK=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name task-sg --description "task" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_TASK \
  --protocol tcp --port 8080 --source-group $SG_ALB

ALB=$(aws elbv2 create-load-balancer --name echo-alb \
  --subnets $SUBNETS --security-groups $SG_ALB --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

TG=$(aws elbv2 create-target-group --name echo-tg \
  --protocol HTTP --port 8080 --vpc-id $VPC --target-type ip \
  --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 create-listener --load-balancer-arn $ALB \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG
```

### 6.9 Service

```bash
SUBNET_LIST=$(echo $SUBNETS | tr ' ' ',')

aws ecs create-service --cluster demo --service-name echo \
  --task-definition echo-app \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_LIST],securityGroups=[$SG_TASK],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG,containerName=app,containerPort=8080" \
  --health-check-grace-period-seconds 60 \
  --enable-execute-command \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true},maximumPercent=200,minimumHealthyPercent=100"
```

→ 실습이라 `assignPublicIp=ENABLED`. production 은 private subnet + NAT.

### 6.10 검증

```bash
DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB \
  --query 'LoadBalancers[0].DNSName' --output text)

sleep 90  # task starting + ALB registration

curl http://$DNS/
# > hello from ecs

aws logs tail /ecs/echo-app --follow --since 5m
```

### 6.11 ECS Exec 시도

```bash
TASK=$(aws ecs list-tasks --cluster demo --service-name echo \
  --query 'taskArns[0]' --output text)

aws ecs execute-command --cluster demo --task $TASK --container app \
  --command "/bin/sh" --interactive
```

### 6.12 Clean up

```bash
aws ecs update-service --cluster demo --service echo --desired-count 0
aws ecs delete-service --cluster demo --service echo --force
aws elbv2 delete-listener --listener-arn $(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB --query 'Listeners[0].ListenerArn' --output text)
aws elbv2 delete-target-group --target-group-arn $TG
aws elbv2 delete-load-balancer --load-balancer-arn $ALB
aws ecs delete-cluster --cluster demo
aws logs delete-log-group --log-group-name /ecs/echo-app
aws ecr delete-repository --repository-name echo-app --force
aws ec2 delete-security-group --group-id $SG_TASK
aws ec2 delete-security-group --group-id $SG_ALB
aws iam detach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ecsTaskExecutionRole
```

---

## 7. 운영 가이드 (요약)

| 의도 | 패턴 |
| --- | --- |
| 첫 service | 7 요소 (ECR/Role/Logs/Cluster/TD/ALB/Service) |
| Image | ECR + ARM64 + IMMUTABLE |
| Container Insights | 시작부터 활성 |
| Circuit Breaker | 시작부터 enable + rollback |
| ECS Exec | enable-execute-command |
| Multi-AZ | subnet 3개 |
| Production 차이 | private + NAT + HTTPS + WAF + Secrets |

---

## 8. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Docker Hub rate limit | ECR |
| Execution Role 없음 | AmazonECSTaskExecutionRolePolicy |
| Log retention 무제한 | 7일 (실습) / 30일 (production) |
| assignPublicIp ENABLED 그대로 production | private subnet |
| Container Insights 안 활성 | 시작부터 |
| TD IMMUTABLE revision 모름 | register 마다 새 revision |
| Service 가 latest revision 안 따라감 | --force-new-deployment |
| Health check grace 짧음 | runtime 별 (nginx 60, Java 120) |

---

## 부록 A — 7 요소

| 요소 | 책임 |
| --- | --- |
| ECR | image storage (IMMUTABLE) |
| Execution Role | pull / log / secret fetch |
| Log Group | CloudWatch (retention) |
| Cluster | 자원 그룹 + Insights |
| TD | container spec |
| ALB + TG | HTTPS / IP target |
| Service | desiredCount / LB / Circuit Breaker |

## 부록 B — 학습용 vs production

| | 학습용 | Production |
| --- | --- | --- |
| Subnet | public | private + NAT |
| Public IP | ENABLED | DISABLED |
| Protocol | HTTP 80 | HTTPS 443 (ACM) |
| AZ | single OK | multi-AZ 3 |
| HA | desired 2 | desired 3+ + AS |
| Endpoint | NAT only | VPC Endpoint |
| WAF | 없음 | WAF + Shield |

## 부록 C — 핵심 명령 sequence

```bash
# 1. ECR
aws ecr create-repository --repository-name echo-app

# 2. Image push
docker buildx build --platform linux/arm64 -t $ECR/echo-app:v1 --push .

# 3. IAM (Execution Role)
aws iam create-role --role-name ecsTaskExecutionRole ...

# 4. Logs
aws logs create-log-group --log-group-name /ecs/echo-app

# 5. Cluster
aws ecs create-cluster --cluster-name demo --settings containerInsights=enabled

# 6. TD register
aws ecs register-task-definition --cli-input-json file://taskdef.json

# 7. ALB + TG + Listener
aws elbv2 create-load-balancer ...
aws elbv2 create-target-group --target-type ip ...
aws elbv2 create-listener ...

# 8. Service
aws ecs create-service --enable-execute-command --circuit-breaker ...
```

---

## 9. 다음

- ▶ [[02-ec2-with-asg]] — EC2 launch + Managed Scaling
- ▶ [[03-blue-green-with-codedeploy]] — Blue-Green
- ▶ [[04-spring-boot-end-to-end]] — production stack

## 10. 영상 / 자료

| 자료 |
| --- |
| [ECS Tutorial — AWS](https://www.youtube.com/watch?v=esISkPlnxL0) |
| [Fargate Workshop](https://ecsworkshop.com/) |

## 11. 관련

- [[../ecs]]
- [[../03-service/01-service-overview]] — Service options
- [[../03-service/05-load-balancer-integration]] — ALB
- [[../02-task-definition/01-overview]] — TD
- [[../04-operations/01-ecs-exec]] — ECS Exec
