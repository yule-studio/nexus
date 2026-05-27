---
title: "EC2 실습 2 — SSM Session Manager 로 key-less 접속 + port forward"
kind: practice
status: current
created_at: 2026-05-22T15:38:00+09:00
tags: [aws, ec2, practice, ssm, session-manager]
home_hub: ec2
---

# 실습 2 — SSM 로 SSH-less 접속

> 목표: SG inbound 22 닫고도 shell + port-forwarding + RDS tunnel. 40분.

## 1. private subnet instance 준비

```bash
# 실습 1 의 instance 를 private subnet 으로
VPC=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)

# default VPC subnet 다 public 이므로 별도 private subnet 생성
AZ=ap-northeast-2a
PRIV=$(aws ec2 create-subnet --vpc-id $VPC --cidr-block 172.31.250.0/24 \
  --availability-zone $AZ --query Subnet.SubnetId --output text)

# Route Table 은 default → IGW 가 아닌 NAT/Endpoint 만 두기 (실습이라 default 그대로 + 새 RT 안 만들고 진행)

SG=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name ssm-private --description "ssm only" --query GroupId --output text)
# inbound 0 (egress 만)

aws iam create-instance-profile --instance-profile-name FirstEc2Profile 2>/dev/null || true
# 실습 1 의 Profile 재사용

AMI=$(aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64 --query Parameter.Value --output text)

aws ec2 run-instances --image-id $AMI --instance-type t4g.nano \
  --subnet-id $PRIV --security-group-ids $SG \
  --iam-instance-profile Name=FirstEc2Profile \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=2" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ssm-private}]'

sleep 90
INST=$(aws ec2 describe-instances --filters Name=tag:Name,Values=ssm-private Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].InstanceId' --output text)
echo $INST
```

## 2. VPC Endpoint (NAT 없이 SSM)

```bash
SG_EP=$(aws ec2 create-security-group --vpc-id $VPC \
  --group-name ssm-endpoint --description "ssm vpc endpoint" --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_EP --protocol tcp --port 443 --source-group $SG

for svc in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC \
    --service-name com.amazonaws.ap-northeast-2.$svc \
    --vpc-endpoint-type Interface \
    --subnet-ids $PRIV \
    --security-group-ids $SG_EP \
    --private-dns-enabled
done
```

> Endpoint 활성까지 ~5분.

## 3. 접속

```bash
sleep 300
aws ssm start-session --target $INST
# 안에서
$ curl http://169.254.169.254/latest/api/token -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 60"
$ exit
```

## 4. Port Forward — local 8080 → instance 80

먼저 instance 안에 nginx 깔기:
```bash
aws ssm send-command --instance-ids $INST \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["dnf install -y nginx","systemctl enable --now nginx","echo OK > /usr/share/nginx/html/index.html"]'
```

local port forwarding:
```bash
aws ssm start-session --target $INST \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["80"],"localPortNumber":["8080"]}'

# 다른 터미널
curl http://localhost:8080
# > OK
```

## 5. Remote Host Port Forward — RDS tunnel

(RDS 가 있는 경우)
```bash
aws ssm start-session --target $INST \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host":["bkurs-db.cluster-xxx.ap-northeast-2.rds.amazonaws.com"],
    "portNumber":["3306"],
    "localPortNumber":["3306"]
  }'

# 다른 터미널
mysql -h 127.0.0.1 -P 3306 -u admin -p
```

→ private RDS 에 SSH tunnel 없이 접속.

## 6. Run Command (script 일괄)

```bash
aws ssm send-command --instance-ids $INST \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["uptime","free -h","df -h"]' \
  --output-s3-bucket-name bkurs-ssm-output 2>/dev/null || \
aws ssm send-command --instance-ids $INST \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["uptime","free -h","df -h"]'

# 결과 확인
CMD=$(aws ssm list-commands --query 'Commands[0].CommandId' --output text)
aws ssm get-command-invocation --command-id $CMD --instance-id $INST --query StandardOutputContent --output text
```

## 7. Audit 활성

```bash
aws s3 mb s3://bkurs-ssm-audit-$RANDOM 2>/dev/null

aws ssm update-document --name SSM-SessionManagerRunShell \
  --document-version "\$LATEST" \
  --content '{
    "schemaVersion":"1.0",
    "description":"audit",
    "sessionType":"Standard_Stream",
    "inputs":{
      "s3BucketName":"bkurs-ssm-audit",
      "s3EncryptionEnabled":true,
      "cloudWatchLogGroupName":"/ssm/sessions",
      "cloudWatchEncryptionEnabled":true
    }
  }'
```

다음 session 부터 input/output 이 S3 + CloudWatch.

## 8. IAM tag-based 권한

다른 IAM user 의 policy:
```json
{
  "Effect": "Allow",
  "Action": "ssm:StartSession",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringEquals": { "aws:ResourceTag/Environment": "dev" }
  }
}
```

→ `Environment=dev` tag 인 instance 만 접근.

## 9. ssh config 통합

`~/.ssh/config`:
```
Host i-* mi-*
  ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

```bash
# Instance Profile + Session Manager preference 의 RunAs 설정 후
ssh ec2-user@$INST
# 단, SSH key 가 instance 에 등록돼야 함 (instance-connect 또는 직접 push)
```

## 10. Clean up

```bash
aws ec2 terminate-instances --instance-ids $INST

aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC \
  --query 'VpcEndpoints[?contains(`["ssm","ssmmessages","ec2messages"]`, ServiceName)].VpcEndpointId' --output text \
  | xargs -I{} aws ec2 delete-vpc-endpoints --vpc-endpoint-ids {}

aws ec2 delete-security-group --group-id $SG_EP
aws ec2 delete-security-group --group-id $SG
aws ec2 delete-subnet --subnet-id $PRIV
```

## 11. 학습 포인트

- inbound 0 → SSM 만 통한 접속
- VPC Endpoint 3 종 (`ssm` / `ssmmessages` / `ec2messages`) 의 역할
- Port Forward — local / remote 두 종류
- Run Command — fleet-wide script
- S3 + CloudWatch audit
- IAM tag-based 권한

## 12. 다음

- ▶ [[03-asg-with-launch-template]]
- ▶ [[../04-operations/01-ssm-session-manager]]
- ▶ [[../05-security/01-iam-instance-profile]]

## 13. 참고

- [[../ec2]]
- [[../04-operations/01-ssm-session-manager]]
