---
title: "troubleshooting · AWS VPC — task 가 public internet 못 나감"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, aws, vpc, nat, vpc-endpoint, private-subnet]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[../devops/ecs-task-failed-to-start]]"]
---

# Private subnet 의 task 가 외부 / AWS API 도달 못함

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

- private subnet 의 EC2 / ECS task 에서 `curl` 가 timeout
- `apt update` / `pip install` / `aws s3 ls` 실패
- `aws sts get-caller-identity` 도 timeout

## 원인

1. NAT Gateway 없음 / route table 잘못
2. VPC Endpoint 없음 (AWS API 접근)
3. Security Group outbound block

## 해결

### NAT Gateway (외부 internet 전체)

```hcl
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_a.id   # public subnet 에 NAT
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
}

resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private.id
}
```

NAT 가 비쌈 ($0.045/h + $0.045/GB) — AZ 별 NAT 하나면 single AZ failure 영향.

### VPC Endpoint (AWS API 만 — 더 저렴)

```hcl
# Interface Endpoint
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-northeast-2.ecr.api"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.ap-northeast-2.ecr.dkr"
  vpc_endpoint_type = "Interface"
  # ...
}

# Gateway Endpoint (S3, DynamoDB — 무료!)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.ap-northeast-2.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = aws_route_table.private[*].id
}
```

**자주 필요한 endpoint**:
- `ecr.api`, `ecr.dkr` — image pull
- `s3` (Gateway) — ECR layer / Lambda code
- `logs` — CloudWatch
- `secretsmanager` — secrets
- `ssm`, `ssmmessages`, `ec2messages` — Session Manager
- `kms` — encrypt/decrypt

각 endpoint $0.01/h × subnet 수.

### Security Group outbound

default 는 all outbound allow. 명시적 deny 있으면 검토:

```hcl
egress {
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

VPC Endpoint 의 SG 도 별도:
```hcl
resource "aws_security_group" "vpc_endpoint" {
  vpc_id = aws_vpc.main.id
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]   # VPC 내부만
  }
}
```

### Route table 확인

```bash
aws ec2 describe-route-tables --filters Name=vpc-id,Values=vpc-xxx
# 각 subnet 의 라우팅:
# - 10.0.0.0/16 → local
# - 0.0.0.0/0 → nat-gw or igw
```

private subnet 의 0.0.0.0/0 가 `igw-*` (internet gateway) 면 안 되고 `nat-*` 또는 `vpce-*` 여야 함.

### Public subnet vs private 구분
```
public: route 가 IGW 로
private: route 가 NAT 또는 endpoint 로만
```

## 진단

```bash
# Instance 의 routing
# (EC2 안에서)
ip route
curl -s http://checkip.amazonaws.com   # 외부 IP — NAT 작동 여부

# AWS API 도달
aws sts get-caller-identity
aws ecr describe-repositories --max-items 1

# VPC Endpoint DNS
nslookup ecr.ap-northeast-2.amazonaws.com
# privateDNS enabled 면 VPC 내부 IP (10.x.x.x) 반환
```

## NAT 비용 절감

- VPC Endpoint 우선 (Gateway endpoint 는 무료)
- 단일 NAT (vs AZ 별) — 비용 절감 vs HA tradeoff
- NAT Instance (EC2 직접) — 작은 환경

## 재발 방지

- Production private subnet 표준: NAT Gateway + 표준 VPC Endpoint set
- CDK / Terraform module 화 — 누락 방지
- 모니터링: NAT bytes / VPC Endpoint connection count

## 관련

- [[network]] / [[connection-refused]] / [[../devops/ecs-task-failed-to-start]] / [[../devops/k8s-imagepullbackoff]]
