---
title: "ECS 실습 2 — EC2 launch + Capacity Provider Managed Scaling"
kind: practice
status: current
created_at: 2026-05-22T14:22:00+09:00
updated_at: 2026-05-26T18:50:00+09:00
tags: [aws, ecs, practice, ec2, asg, capacity-provider]
home_hub: ecs
---

# 실습 2 — EC2 launch + Capacity Provider Managed Scaling

## 1. 목적

- EC2 launch type 의 **capacity management 모델** 체험 — Fargate microVM 과 다른 책임.
- ECS-optimized AMI 의 의의 — ecs-agent + Docker + SSM Agent 사전 설치.
- user-data 의 `ECS_CLUSTER` env 가 register 결정자.
- Capacity Provider Managed Scaling 의 **ECS-aware ASG 조정** mechanism.
- bridge network mode 의 dynamic port (32768-65535) + bin-packing.
- managedTerminationProtection 의 graceful drain.
- 45분 안 완료.

## 2. 범위

| 다룬다 | 다루지 않는다 |
| --- | --- |
| EC2 ECS-optimized AMI 사용 | Fargate → [[01-fargate-first-service]] |
| ASG + Launch Template + Capacity Provider | Blue-Green → [[03-blue-green-with-codedeploy]] |
| Managed Scaling 의 ECS-aware | Spring Boot → [[04-spring-boot-end-to-end]] |
| bridge mode + dynamic port | Auto Scaling 일반 → [[../../auto-scaling/auto-scaling]] |
| managedTerminationProtection | EC2 ASG 자체 → [[../../ec2/practice/03-asg-with-launch-template]] |

## 3. 사전 지식

- [[01-fargate-first-service]] — 첫 ECS service 흐름.
- [[../01-fundamentals/02-launch-type-and-capacity-provider]] §4.8 — Managed Scaling.
- [[../02-task-definition/03-network-mode]] §4.5 — bridge mode.
- [[../../ec2/practice/03-asg-with-launch-template]] — EC2 ASG 일반.

---

## 4. 작동 원리

### 4.1 EC2 launch vs Fargate — capacity 책임 차이

```
[Fargate]
- AWS 가 task 마다 microVM 자동 provision
- 사용자는 task spec 만
- capacity 부족 = AWS 가 더 만들어줌 (사실상 unlimited)

[EC2 launch]
- 사용자가 ASG 의 EC2 instance 직접 관리
- ECS scheduler 가 instance 의 남은 capacity 보고 placement
- capacity 부족 = ASG scale-out 필요 (사용자 책임)
- bin-packing 효율 (한 EC2 에 N task)
```

**왜 EC2 launch 선택**:

- 비용 절감 (bin-packing + Spot)
- GPU / 특수 instance type 필요
- Daemon process 필요
- bare metal 접근 필요 (rare)

### 4.2 ECS-optimized AMI 의 의의

```bash
AMI=$(aws ssm get-parameter \
  --name /aws/service/ecs/optimized-ami/amazon-linux-2023/recommended/image_id \
  --query Parameter.Value --output text)
```

**사전 설치**:

- **ecs-agent** — ECS cluster 와 통신 (long-polling)
- **Docker / containerd** — container runtime
- **SSM Agent** — ECS Exec / Run Command
- **CloudWatch Agent** — metric / log
- **awscli** — 운영

**왜 표준 AMI 안 쓰나**:

- 표준 Amazon Linux = 위 설치 manual
- ECS-optimized = AWS 가 최적 patch + tested
- 매주 새 AMI version → SSM parameter 가 latest 가리킴

### 4.3 user-data 의 `ECS_CLUSTER` env

```bash
USER_DATA=$(printf '#!/bin/bash\necho ECS_CLUSTER=ec2-demo >> /etc/ecs/ecs.config\n' | base64)
```

**왜 필수**:

- ecs-agent 가 어느 cluster 에 join 할지 결정
- `/etc/ecs/ecs.config` 의 `ECS_CLUSTER=<name>` 가 source of truth
- 없으면 default cluster 에 join (대부분 잘못된 결과)

**확인**:

```bash
# instance 안에서
cat /etc/ecs/ecs.config
# ECS_CLUSTER=ec2-demo
sudo systemctl status ecs
```

### 4.4 Managed Scaling — ECS-aware ASG

```bash
"managedScaling": {
  "status": "ENABLED",
  "targetCapacity": 100,
  "minimumScalingStepSize": 1,
  "maximumScalingStepSize": 5,
  "instanceWarmupPeriod": 300
}
```

**Mechanism — ECS-aware vs CloudWatch alarm-based**:

```
[without Managed Scaling]
1. task PROVISIONING (capacity 부족)
2. CloudWatch metric (CPU/Memory) 측정 (1-2분)
3. Alarm 발생
4. ASG scale-out trigger
5. EC2 launch (3-5분)
6. ECS register
→ 총 6-8분 lag

[with Managed Scaling]
1. task PROVISIONING (capacity 부족)
2. ECS scheduler 가 pending count 즉시 계산
3. 직접 ASG desiredCount ↑
4. EC2 launch (3분)
5. ECS register
→ 총 3-4분 (lag 절반)
```

**`targetCapacity=100` 의 의미**:

- ASG capacity 의 100% 활용 (bin-packing 최대)
- 100 = 낭비 0 / 80-90 = 헤드룸 (안전)
- 권장: 80-90 (Service auto-scale 시 잠시 burst capacity 필요)

→ 실습은 100 으로 간단히. production 은 85.

### 4.5 bridge mode + dynamic port

```json
{
  "networkMode": "bridge",
  "containerDefinitions": [{
    "portMappings": [{ "containerPort": 80, "hostPort": 0 }]
  }]
}
```

**`hostPort: 0` 의미**:

- ECS 가 자동 random port (32768-65535) 할당
- 같은 EC2 에 같은 image task N개 packing 가능 (port 충돌 없음)
- ALB target group type=Instance 가 자동 등록

**왜 SG ingress 32768-65535**:

```bash
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 32768-65535 --cidr 10.0.0.0/8
```

- ALB 가 dynamic port 로 접근
- 정확한 port 는 ECS 가 결정 → range 전체 허용
- CIDR 은 VPC 안 (외부 차단)

→ awsvpc 가 새 권장 ([[../02-task-definition/03-network-mode]] §4.2). bridge 는 legacy bin-packing 용.

### 4.6 managedTerminationProtection — graceful drain

```json
{
  "managedTerminationProtection": "ENABLED"
}
```

**Mechanism**:

```
ASG scale-in 결정 (instance 줄임)
   │
   ▼
ASG 가 termination 후보 instance 선택
   │
   ▼
ECS scheduler 가 instance 의 task 확인
   ├─ task 있음: ASG terminate 차단 (drain 완료까지)
   │      → task 종료 시점에 다시 시도
   └─ task 없음: ASG terminate 허용
```

**왜 의미**:

- task 살아있는 instance 갑자기 terminate 차단
- graceful drain (ALB deregister + stopTimeout)
- production 필수

→ ASG 의 `new-instances-protected-from-scale-in` 도 결합 (새 instance 도 보호 후 ECS 가 unprotect).

### 4.7 IAM 두 Role + 1 Instance Profile

```
1. ecsTaskExecutionRole   — Fargate / EC2 task 가 사용 (image pull / log put)
2. ecsTaskRole            — application 권한 (S3 / DDB)
3. ecsInstanceRole + Instance Profile — EC2 host 가 사용 (cluster register)
```

**왜 3 분리** (자세히 [[../05-security/01-iam-best-practices]] §4.1):

| Role | 누가 | 권한 |
| --- | --- | --- |
| Execution | ECS agent | ECR / Logs / SSM |
| Task | application | S3 / DDB |
| Instance | EC2 host | cluster register + Container Insights |

**Instance Profile**:

```bash
aws iam create-instance-profile --instance-profile-name ecsInstanceProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name ecsInstanceProfile --role-name ecsInstanceRole
```

→ EC2 launch 시 attach. `AmazonEC2ContainerServiceforEC2Role` managed policy 사용.

### 4.8 Service + Capacity Provider Strategy

```bash
--capacity-provider-strategy capacityProvider=ec2-cp,weight=1
```

→ default strategy 따름 (cluster 의 put-cluster-capacity-providers). 

**확인 — task 추가 → instance scale-out**:

- desired 3 task → 1 EC2 (t3.small) 으로 부족 → ASG scale-out → 새 instance launch → task placement
- desired 10 task → 추가 scale → 여러 instance

---

## 5. 안 쓰면 어떤 문제 / 대안 / 트레이드오프

### 5.1 EC2 launch 인데 ECS-optimized AMI 안 씀

- ecs-agent / Docker / SSM 수동 설치
- patch 관리 부담
- 권장: SSM parameter 의 latest AMI

### 5.2 `ECS_CLUSTER` env 누락

- instance 가 default cluster 에 join → 의도와 다름
- 권장: user-data 에 명시

### 5.3 Managed Scaling OFF

- CloudWatch alarm lag 5분+ → task placement 지연
- 권장: ENABLED + targetCapacity 80-90

### 5.4 managedTerminationProtection OFF

- task 살아있는 instance terminate
- 권장: ENABLED

### 5.5 bridge mode SG 32768-65535 누락

- ALB target health check fail
- 권장: range 전체 + VPC CIDR

### 5.6 Instance Profile 누락

- instance 가 cluster register 못 함
- 권장: ecsInstanceProfile + AmazonEC2ContainerServiceforEC2Role

### 5.7 Fargate vs EC2 결정

- Fargate = 운영 부담 0, 비싸
- EC2 = bin-packing + Spot 절감, 운영 부담
- 권장: 시작은 Fargate, scale 커지면 EC2 mix

---

## 6. 구체 예시 — 실습 명령

### 6.1 ECS-optimized AMI

```bash
REGION=ap-northeast-2
AMI=$(aws ssm get-parameter \
  --name /aws/service/ecs/optimized-ami/amazon-linux-2023/recommended/image_id \
  --query Parameter.Value --output text)
echo $AMI
```

### 6.2 IAM (Instance Profile)

```bash
aws iam create-role --role-name ecsInstanceRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name ecsInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

aws iam create-instance-profile --instance-profile-name ecsInstanceProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name ecsInstanceProfile --role-name ecsInstanceRole
```

### 6.3 Cluster

```bash
aws ecs create-cluster --cluster-name ec2-demo \
  --settings name=containerInsights,value=enabled
```

### 6.4 Launch Template

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' --output text)

SG=$(aws ec2 create-security-group --group-name ecs-ec2-sg \
  --description "ECS EC2" --vpc-id $VPC --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 32768-65535 --cidr 10.0.0.0/8

USER_DATA=$(printf '#!/bin/bash\necho ECS_CLUSTER=ec2-demo >> /etc/ecs/ecs.config\n' | base64)

aws ec2 create-launch-template --launch-template-name ecs-lt \
  --launch-template-data "{
    \"ImageId\": \"$AMI\",
    \"InstanceType\": \"t3.small\",
    \"IamInstanceProfile\": { \"Name\": \"ecsInstanceProfile\" },
    \"SecurityGroupIds\": [\"$SG\"],
    \"UserData\": \"$USER_DATA\"
  }"
```

### 6.5 Auto Scaling Group

```bash
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC \
  --query 'Subnets[].SubnetId' --output text | tr '\t' ',')

aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name ecs-asg \
  --launch-template LaunchTemplateName=ecs-lt \
  --min-size 1 --max-size 5 --desired-capacity 1 \
  --vpc-zone-identifier "$SUBNETS" \
  --new-instances-protected-from-scale-in
```

### 6.6 Capacity Provider + Managed Scaling

```bash
ASG_ARN=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names ecs-asg \
  --query 'AutoScalingGroups[0].AutoScalingGroupARN' --output text)

aws ecs create-capacity-provider \
  --name ec2-cp \
  --auto-scaling-group-provider "{
    \"autoScalingGroupArn\": \"$ASG_ARN\",
    \"managedScaling\": {
      \"status\": \"ENABLED\",
      \"targetCapacity\": 100,
      \"minimumScalingStepSize\": 1,
      \"maximumScalingStepSize\": 5,
      \"instanceWarmupPeriod\": 300
    },
    \"managedTerminationProtection\": \"ENABLED\"
  }"

aws ecs put-cluster-capacity-providers \
  --cluster ec2-demo \
  --capacity-providers ec2-cp \
  --default-capacity-provider-strategy capacityProvider=ec2-cp,weight=1,base=1
```

### 6.7 Task Definition (bridge mode)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)

cat > td.json <<EOF
{
  "family": "ec2-demo",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "executionRoleArn": "arn:aws:iam::$ACCT:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "app",
    "image": "nginxdemos/hello",
    "essential": true,
    "memory": 128,
    "portMappings": [{ "containerPort": 80, "hostPort": 0 }],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/ec2-demo",
        "awslogs-region": "$REGION",
        "awslogs-stream-prefix": "app",
        "awslogs-create-group": "true"
      }
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file://td.json
```

### 6.8 Service

```bash
aws ecs create-service --cluster ec2-demo --service-name web \
  --task-definition ec2-demo --desired-count 3 \
  --capacity-provider-strategy capacityProvider=ec2-cp,weight=1
```

### 6.9 검증 — Managed Scaling 동작

```bash
# 3 task → 1 EC2 부족 → ASG scale out 자동
watch -n 5 'aws ecs describe-services --cluster ec2-demo --services web \
  --query "services[0].{desired:desiredCount,running:runningCount,pending:pendingCount}"'

# instance 증가
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ecs-asg \
  --query 'AutoScalingGroups[0].{desired:DesiredCapacity,instances:Instances[].InstanceId}'

# desired 10 → 추가 scale
aws ecs update-service --cluster ec2-demo --service web --desired-count 10
```

→ Managed Scaling 이 자동으로 ASG desiredCount 조정하는 것 관찰.

### 6.10 Clean up

```bash
aws ecs update-service --cluster ec2-demo --service web --desired-count 0
aws ecs delete-service --cluster ec2-demo --service web --force
aws ecs put-cluster-capacity-providers --cluster ec2-demo \
  --capacity-providers --default-capacity-provider-strategy
aws ecs delete-capacity-provider --capacity-provider ec2-cp
aws autoscaling update-auto-scaling-group --auto-scaling-group-name ecs-asg \
  --min-size 0 --desired-capacity 0
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name ecs-asg --force-delete
aws ec2 delete-launch-template --launch-template-name ecs-lt
aws ec2 delete-security-group --group-id $SG
aws ecs delete-cluster --cluster ec2-demo
aws logs delete-log-group --log-group-name /ecs/ec2-demo
aws iam remove-role-from-instance-profile \
  --instance-profile-name ecsInstanceProfile --role-name ecsInstanceRole
aws iam delete-instance-profile --instance-profile-name ecsInstanceProfile
aws iam detach-role-policy --role-name ecsInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role
aws iam delete-role --role-name ecsInstanceRole
```

---

## 7. 운영 가이드 (요약)

| 의도 | 패턴 |
| --- | --- |
| EC2 ECS cluster | ECS-optimized AMI (SSM parameter latest) |
| User-data | ECS_CLUSTER env 명시 |
| Capacity 조정 | Managed Scaling ENABLED + targetCapacity 80-90 |
| Graceful drain | managedTerminationProtection ENABLED |
| Bin-packing | bridge mode + dynamic port |
| SG | 32768-65535 from VPC CIDR |
| Multi-AZ | ASG vpc-zone-identifier 3 subnet |
| IAM | Instance Profile + AmazonEC2ContainerServiceforEC2Role |
| Spot mix | Mixed Instance Policy ([[../../ec2/practice/03-asg-with-launch-template]] §4.7) |

---

## 8. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Standard Amazon Linux AMI | ECS-optimized AMI |
| ECS_CLUSTER 누락 | user-data 명시 |
| Managed Scaling OFF | ENABLED |
| managedTerminationProtection OFF | ENABLED |
| bridge SG 32768-65535 누락 | range 전체 |
| Instance Profile 없음 | ecsInstanceProfile |
| awsvpc 신규 | bridge legacy — awsvpc 권장 |
| targetCapacity 100 (헤드룸 0) | 80-90 (production) |

---

## 부록 A — Fargate vs EC2 책임

| | Fargate | EC2 |
| --- | --- | --- |
| Capacity | AWS 자동 | 사용자 ASG |
| OS patch | AWS | 사용자 (AMI 갱신) |
| Bin-packing | task 마다 microVM | 여러 task per host |
| Spot | FARGATE_SPOT | EC2 Spot |
| Cost | 비싸 | 절감 |

## 부록 B — ECS-optimized AMI

```bash
aws ssm get-parameter \
  --name /aws/service/ecs/optimized-ami/amazon-linux-2023/recommended/image_id
```

사전 설치: ecs-agent / Docker / SSM Agent / CloudWatch Agent / awscli.

## 부록 C — user-data

```bash
#!/bin/bash
echo ECS_CLUSTER=<cluster-name> >> /etc/ecs/ecs.config
```

## 부록 D — Capacity Provider Managed Scaling

```json
{
  "managedScaling": {
    "status": "ENABLED",
    "targetCapacity": 85,
    "minimumScalingStepSize": 1,
    "maximumScalingStepSize": 10,
    "instanceWarmupPeriod": 300
  },
  "managedTerminationProtection": "ENABLED"
}
```

## 부록 E — bridge mode + dynamic port

```json
"portMappings": [{ "containerPort": 80, "hostPort": 0 }]
```

```bash
aws ec2 authorize-security-group-ingress --group-id $SG \
  --protocol tcp --port 32768-65535 --cidr 10.0.0.0/8
```

---

## 9. 다음

- ▶ [[03-blue-green-with-codedeploy]] — Blue-Green
- ▶ [[04-spring-boot-end-to-end]] — production stack
- ▶ [[../03-service/03-auto-scaling]] — Service Auto Scaling
- ▶ [[../../ec2/practice/03-asg-with-launch-template]] — EC2 ASG deep

## 10. 영상 / 자료

| 자료 |
| --- |
| [ECS EC2 with ASG](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/asg-capacity-providers.html) |
| [ECS-optimized AMI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html) |

## 11. 관련

- [[../ecs]]
- [[../01-fundamentals/02-launch-type-and-capacity-provider]] — CP Strategy
- [[01-fargate-first-service]] — Fargate 비교
- [[../02-task-definition/03-network-mode]] §4.5 — bridge
