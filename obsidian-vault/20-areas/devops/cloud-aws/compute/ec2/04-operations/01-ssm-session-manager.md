---
title: "EC2 04-1 — SSM Session Manager (key-less / audit)"
kind: knowledge
status: current
created_at: 2026-05-22T15:24:00+09:00
tags: [aws, ec2, operations, ssm, session-manager]
home_hub: ec2
---

# SSM Session Manager

> SSH 22 인터넷 노출 불필요. IAM 인증 + audit log + port forwarding + run command. EC2 접속의 표준.

## 1. 동작

```mermaid
flowchart LR
  User[Developer Laptop] --> CLI[aws ssm start-session]
  CLI -->|HTTPS 443| API[SSM API]
  API --> Agent[ssm-agent on EC2]
  Agent -->|local exec| Shell[/bin/bash]
  API -->|audit| Logs[CloudWatch / S3]
```

- inbound port 0 (instance 가 SSM 에 outbound only)
- HTTPS / WebSocket
- session 자체 IAM 인증

## 2. 활성 조건 4

| 조건 | 설정 |
| --- | --- |
| SSM Agent 설치 | Amazon Linux 2/2023, Ubuntu 18+, Windows 2016+ — pre-installed |
| Instance Profile + SSM 권한 | `AmazonSSMManagedInstanceCore` |
| Outbound HTTPS (443) | SG egress 443 또는 VPC Endpoint |
| Session Manager Plugin (local) | `brew install --cask session-manager-plugin` |

## 3. Instance Profile

```bash
aws iam create-role --role-name EC2SSMRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy --role-name EC2SSMRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name EC2SSMProfile
aws iam add-role-to-instance-profile --instance-profile-name EC2SSMProfile --role-name EC2SSMRole

# Instance 에 attach
aws ec2 associate-iam-instance-profile --instance-id i-xxx \
  --iam-instance-profile Name=EC2SSMProfile
```

## 4. Session 시작

```bash
# Shell
aws ssm start-session --target i-xxx

# Specific user (Session Manager preference 설정 후)
aws ssm start-session --target i-xxx \
  --document-name AWS-StartInteractiveCommand \
  --parameters command="sudo su - ubuntu"
```

## 5. Port Forwarding

```bash
# Local 5432 → instance 5432
aws ssm start-session --target i-xxx \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["5432"],"localPortNumber":["5432"]}'

# RDS / 내부 endpoint 까지
aws ssm start-session --target i-bastion \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host":["bkurs-db.cluster-xxx.ap-northeast-2.rds.amazonaws.com"],
    "portNumber":["3306"],
    "localPortNumber":["3306"]
  }'
```

→ private RDS 에 SSH tunneling 없이 접속.

## 6. Run Command (script 일괄 실행)

```bash
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets "Key=tag:Environment,Values=prod" \
  --parameters 'commands=["systemctl restart nginx","systemctl status nginx"]' \
  --comment "restart nginx" \
  --output-s3-bucket-name bkurs-ssm-output
```

→ tag 기준 fleet-wide 실행, output S3.

## 7. Cluster audit log

```bash
aws ssm update-document --name SSM-SessionManagerRunShell \
  --content '{
    "schemaVersion": "1.0",
    "description": "session settings",
    "sessionType": "Standard_Stream",
    "inputs": {
      "s3BucketName": "bkurs-ssm-audit",
      "s3EncryptionEnabled": true,
      "cloudWatchLogGroupName": "/ssm/sessions",
      "cloudWatchEncryptionEnabled": true,
      "kmsKeyId": "alias/ssm"
    }
  }' --document-version "$LATEST"
```

→ 모든 session input/output 이 S3 + CloudWatch 에 저장.

## 8. VPC Endpoint (private subnet)

NAT 없이 SSM 사용:
```bash
for svc in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
    --service-name com.amazonaws.ap-northeast-2.$svc \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-priv-a subnet-priv-b subnet-priv-c \
    --security-group-ids sg-endpoint \
    --private-dns-enabled
done
```

→ private subnet 의 instance 도 NAT 없이 SSM 접속.

## 9. SSH over SSM

ssh config:
```
Host bastion
  HostName i-xxx
  User ec2-user
  IdentityFile ~/.ssh/id_rsa
  ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

```bash
ssh bastion
```

→ SSH 표준 도구 (scp, rsync) 가 SSM tunnel 위로 동작.

## 10. IAM 제한 — Tag based

```json
{
  "Effect": "Allow",
  "Action": "ssm:StartSession",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/Environment": "dev",
      "aws:PrincipalTag/Role": "Developer"
    }
  }
}
```

→ Developer tag 가진 사람만 dev tag 인 instance 접근.

## 11. Fleet Manager (GUI)

Console → Systems Manager → Fleet Manager:
- file browse / edit
- registry / process
- performance counter
- Connect → RDP / SSH

## 12. SSH 와 비교

| | SSH (key) | SSM Session Manager |
| --- | --- | --- |
| Port 22 노출 | 필요 | 불필요 |
| Key 관리 | pem 파일 | IAM (key 없음) |
| Audit | sshd log | S3 + CloudWatch |
| MFA | sshd 설정 | IAM MFA 자동 |
| 권한 변경 | OS user / key 재배포 | IAM policy |
| 부팅 시 자동 enable | ✗ | ✓ (Agent 만 있으면) |
| Port forwarding | ✓ | ✓ |
| Cross-VPC bastion 필요 | ✓ | ✗ (VPC Endpoint) |

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Agent 안 실행 | systemctl enable --now amazon-ssm-agent |
| Outbound 443 차단 | SG egress 443 또는 VPC Endpoint |
| Plugin 미설치 | local 설치 |
| audit log 미설정 | document 설정 |
| 모든 user 가 root | run-as 명시 |
| Session timeout 무한 | document `idleSessionTimeout` |

## 14. 다음

- ▶ [[02-monitoring-cloudwatch]]
- ▶ [[03-troubleshooting]]
- ▶ [[../05-security/01-iam-instance-profile]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [Session Manager — AWS](https://www.youtube.com/watch?v=hcOWoEDfyJg) |
| [공식](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html) |

## 16. 참고

- [[../ec2]]
- [[../02-networking/02-security-group-and-nacl]]
- [[../../ecs/04-operations/01-ecs-exec]]
