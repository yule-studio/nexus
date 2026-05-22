---
title: "EC2 실습 1 — 첫 인스턴스 (Console + CLI + Terraform)"
kind: practice
status: current
created_at: 2026-05-22T15:36:00+09:00
tags: [aws, ec2, practice, terraform]
home_hub: ec2
---

# 실습 1 — 첫 EC2 인스턴스

> 목표: Amazon Linux 2023 ARM64 + user-data 로 nginx 자동 설치 + ALB 없이 public IP 직접 → 브라우저. 20분.

## 사전 준비

- AWS CLI v2 + region `ap-northeast-2`
- (Terraform 사용 시) `terraform >= 1.7`

## 1. SG + Key Pair

```bash
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)

SG=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name first-ec2 --description "first ec2 lab" \
  --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 80 --cidr 0.0.0.0/0
# 22 는 절대 노출 X — SSM 사용

aws ec2 create-key-pair --key-name first-ec2 --query KeyMaterial --output text > ~/.ssh/first-ec2.pem
chmod 400 ~/.ssh/first-ec2.pem
```

## 2. user-data

`user-data.sh`:
```bash
#!/bin/bash
set -euxo pipefail
dnf update -y
dnf install -y nginx
echo "<h1>Hello from $(hostname)</h1>" > /usr/share/nginx/html/index.html
systemctl enable --now nginx
```

## 3. AMI (Amazon Linux 2023 ARM64)

```bash
AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 \
  --query Parameter.Value --output text)
echo $AMI
```

## 4. Instance Profile (SSM)

```bash
aws iam create-role --role-name FirstEc2Role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name FirstEc2Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name FirstEc2Profile
aws iam add-role-to-instance-profile --instance-profile-name FirstEc2Profile --role-name FirstEc2Role
```

## 5. Launch (CLI)

```bash
SUBNET=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC \
  Name=defaultForAz,Values=true \
  --query 'Subnets[0].SubnetId' --output text)

aws ec2 run-instances \
  --image-id $AMI --instance-type t4g.small \
  --key-name first-ec2 \
  --security-group-ids $SG --subnet-id $SUBNET \
  --user-data file://user-data.sh \
  --iam-instance-profile Name=FirstEc2Profile \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":20,"VolumeType":"gp3","DeleteOnTermination":true,"Encrypted":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=first-ec2}]'

INST=$(aws ec2 describe-instances --filters Name=tag:Name,Values=first-ec2 Name=instance-state-name,Values=pending,running \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
echo "instance: $INST"
```

## 6. Public IP + 검증

```bash
sleep 60   # 부팅 + user-data 완료 대기

IP=$(aws ec2 describe-instances --instance-ids $INST \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "http://$IP"
curl http://$IP
# > <h1>Hello from ip-xxx</h1>
```

## 7. SSM 접속

```bash
aws ssm start-session --target $INST
# inside:
$ whoami
ssm-user
$ sudo systemctl status nginx
```

## 8. CloudWatch Agent (옵션)

[[../04-operations/02-monitoring-cloudwatch]] §4 참조.

## 9. Terraform 버전

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.50" }
  }
}
provider "aws" { region = "ap-northeast-2" }

data "aws_vpc" "default" { default = true }
data "aws_subnets" "default" { filter { name = "vpc-id"; values = [data.aws_vpc.default.id] } }
data "aws_ssm_parameter" "ami" { name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64" }

resource "aws_security_group" "web" {
  vpc_id = data.aws_vpc.default.id
  name   = "first-ec2"
  ingress { from_port = 80, to_port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0,  to_port = 0,  protocol = "-1", cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_iam_role" "ec2" {
  name = "FirstEc2Role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow", Principal = { Service = "ec2.amazonaws.com" }, Action = "sts:AssumeRole" }]
  })
}
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
resource "aws_iam_instance_profile" "ec2" { name = "FirstEc2Profile"; role = aws_iam_role.ec2.name }

resource "aws_instance" "web" {
  ami                    = data.aws_ssm_parameter.ami.value
  instance_type          = "t4g.small"
  subnet_id              = data.aws_subnets.default.ids[0]
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2.name
  user_data              = file("user-data.sh")
  metadata_options { http_tokens = "required"; http_put_response_hop_limit = 2 }
  root_block_device { volume_size = 20; volume_type = "gp3"; encrypted = true }
  tags = { Name = "first-ec2" }
}

output "url" { value = "http://${aws_instance.web.public_ip}" }
```

```bash
terraform init && terraform apply -auto-approve
```

## 10. Clean up

```bash
aws ec2 terminate-instances --instance-ids $INST
aws iam remove-role-from-instance-profile --instance-profile-name FirstEc2Profile --role-name FirstEc2Role
aws iam delete-instance-profile --instance-profile-name FirstEc2Profile
aws iam detach-role-policy --role-name FirstEc2Role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name FirstEc2Role
aws ec2 delete-security-group --group-id $SG
aws ec2 delete-key-pair --key-name first-ec2
rm ~/.ssh/first-ec2.pem
# Terraform: terraform destroy -auto-approve
```

## 11. 학습 포인트

- Amazon Linux 2023 ARM64 (Graviton t4g)
- user-data 의 부팅 자동화
- IMDSv2 강제 (HttpTokens=required)
- SSM Session Manager — SSH 22 없이 접속
- EBS gp3 + encryption default
- Public IP 직접 (production 은 ALB)

## 12. 다음

- ▶ [[02-ssm-ssh-less]]
- ▶ [[03-asg-with-launch-template]]
- ▶ [[../05-security/02-imdsv2]]

## 13. 참고

- [[../ec2]]
- [[../01-fundamentals/04-amis-and-bootstrap]]
- [[../05-security/01-iam-instance-profile]]
