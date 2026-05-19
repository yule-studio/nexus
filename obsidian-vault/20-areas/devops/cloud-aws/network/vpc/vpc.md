---
title: "AWS VPC — Virtual Private Cloud"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:00:00+09:00
tags:
  - aws
  - network
  - vpc
---

# AWS VPC

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | VPC 개념 + 토폴로지 |

**[[network|↑ Network]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 안의 **격리된 가상 네트워크**. 모든 자원의 base. CIDR / subnet / route / SG 의 모든 것.

---

## 2. 왜

- **격리** — 다른 AWS 사용자와 분리
- **subnet 으로 보안 zone 분리** (public / private)
- **IP 제어** — 같은 LAN 의 사설 IP
- **on-premise 연결** (VPN / Direct Connect)

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **VPC** | 격리 네트워크 (CIDR 블록, /16 권장) |
| **Subnet** | AZ 별 IP 범위 (/24 등) |
| **Route Table** | subnet 별 라우팅 |
| **Internet Gateway (IGW)** | VPC ↔ 인터넷 |
| **NAT Gateway** | private → 인터넷 outbound |
| **Security Group (SG)** | stateful 인스턴스 방화벽 |
| **NACL** | stateless subnet 방화벽 |
| **VPC Peering** | VPC 간 연결 |
| **Transit Gateway** | 다수 VPC + on-premise hub |
| **VPC Endpoint** | S3/DynamoDB 등 privately |
| **PrivateLink** | 다른 VPC 서비스 private 접근 |

---

## 4. 표준 토폴로지 (3-tier)

```
                    Internet
                       ↓
                Internet Gateway
                       ↓
              ┌────────┴────────┐
              ↓                 ↓
   [Public subnet AZ-A]  [Public subnet AZ-B]
     - ALB                - ALB
     - NAT GW             - NAT GW
              ↓                 ↓
   [Private subnet AZ-A] [Private subnet AZ-B]
     - EC2 / ECS / EKS    - 같음
              ↓                 ↓
   [DB subnet AZ-A]      [DB subnet AZ-B]
     - RDS                - RDS standby
```

CIDR 예:
```
VPC:        10.0.0.0/16
Public-A:   10.0.1.0/24
Public-B:   10.0.2.0/24
Private-A:  10.0.11.0/24
Private-B:  10.0.12.0/24
DB-A:       10.0.21.0/24
DB-B:       10.0.22.0/24
```

---

## 5. 생성

### 5.1 CLI

```bash
# VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16
# vpc-xxx

# subnet
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone ap-northeast-2a
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.2.0/24 --availability-zone ap-northeast-2c

# IGW
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx

# route
aws ec2 create-route-table --vpc-id vpc-xxx
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx
aws ec2 associate-route-table --subnet-id subnet-xxx --route-table-id rtb-xxx

# NAT GW (private subnet 의 outbound)
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id <public-subnet> --allocation-id eipalloc-xxx
```

### 5.2 Terraform — module 이 가장 편함

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.x"

  name = "myapp-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["ap-northeast-2a", "ap-northeast-2c"]

  public_subnets   = ["10.0.1.0/24",  "10.0.2.0/24"]
  private_subnets  = ["10.0.11.0/24", "10.0.12.0/24"]
  database_subnets = ["10.0.21.0/24", "10.0.22.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false              # AZ 마다 NAT
  enable_dns_hostnames = true
  enable_dns_support   = true

  enable_s3_endpoint = true                  # VPC endpoint
}
```

---

## 6. Security Group (SG)

stateful — return traffic 자동 허용.

```hcl
resource "aws_security_group" "web" {
  vpc_id = module.vpc.vpc_id
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# DB SG — 응용 SG 만 허용
resource "aws_security_group" "rds" {
  vpc_id = module.vpc.vpc_id
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```

→ SG 참조로 IP 변경에 무관.

---

## 7. NACL (Network ACL)

stateless — 양방향 명시.

- subnet 단위
- explicit allow/deny + 순서 번호
- default = 모든 양방향 허용

→ 일반적으로 default 사용. 특수 격리 필요 시만 작성.

---

## 8. VPC Endpoint

NAT Gateway 거치지 않고 AWS 서비스 직접 접근.

### 8.1 Gateway endpoint (S3 / DynamoDB) — 무료

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.ap-northeast-2.s3"
  route_table_ids   = module.vpc.private_route_table_ids
  vpc_endpoint_type = "Gateway"
}
```

→ NAT 비용 절약.

### 8.2 Interface endpoint (대부분 서비스)

```hcl
resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.ap-northeast-2.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.endpoints.id]
  private_dns_enabled = true
}
```

→ ENI 추가 비용 ($0.01/시간/AZ + data).

---

## 9. VPC Peering / Transit Gateway

### 9.1 Peering — 2 VPC 1:1

```bash
aws ec2 create-vpc-peering-connection --vpc-id vpc-a --peer-vpc-id vpc-b
```

- 단순
- CIDR 겹치면 X
- transitive 불가 (A↔B, B↔C 면 A↔C X)

### 9.2 Transit Gateway — hub

```
VPC-A ─┐
VPC-B ─┼── TGW ── on-prem (VPN/DX)
VPC-C ─┘
```

→ 다수 VPC + on-premise hub. 비싸지만 깔끔.

---

## 10. 비용

```
VPC, subnet, RT, SG, NACL = 무료
IGW = 무료
NAT Gateway = $0.045/시간 + $0.045/GB-out — 매우 비쌈
Elastic IP (사용 X) = $0.005/시간
PrivateLink Endpoint = $0.01/시간/AZ + $0.01/GB
Data Transfer:
  같은 AZ private = 무료
  cross-AZ        = $0.01/GB (둘 다)
  outbound 인터넷 = $0.09/GB
Transit GW       = $0.05/attachment·시간 + $0.02/GB
```

→ NAT GW + DT 가 흔한 1,2 위 비용 함정.

---

## 11. 사용 시나리오

- 모든 AWS 인프라의 base (VPC 만들 일 100%)
- 응용 / DB 분리
- on-prem 연결
- 마이크로서비스 (VPC 1 + subnet 분리)
- multi-account (각 account 1 VPC + TGW hub)

---

## 12. 함정

### 12.1 default VPC
account 마다 하나. 권한 / 보안 약함 — production = 직접 새 VPC.

### 12.2 CIDR 겹침
peer / TGW 시 X. 처음에 큰 그림 (RFC1918 의 어디).

### 12.3 NAT GW 비용 폭증
private subnet 의 outbound 트래픽 폭증. VPC endpoint 활용.

### 12.4 SG vs NACL 혼동
대부분 SG 로 충분. NACL 은 subnet 격리 / DDoS 차단.

### 12.5 cross-AZ data transfer
같은 region 이지만 $/GB. 옛 모놀리식이 RDS 다른 AZ → 매월 큰 비용.

### 12.6 single NAT vs multi-AZ NAT
비용 vs 가용성. production = AZ 마다.

### 12.7 IP 고갈
small /24 + 많은 task (ECS awsvpc mode) → 빠르게 고갈. /20 권장.

### 12.8 Elastic IP 잊음
unused EIP 도 과금.

---

## 13. 학습 자료

- AWS VPC docs
- **AWS VPC workshop** — vpcworkshop.com
- **VPC Best Practices** — AWS whitepaper

---

## 14. 관련

- [[network]] — Network hub
- [[alb]] / [[cloudfront]] / [[route53]]
- [[iam]]
- [[network|↗ network 이론]]
