---
title: "NLB 01-2 — Static IP (EIP) + Cross-Zone + Source IP 보존"
kind: knowledge
status: current
created_at: 2026-05-23T10:32:00+09:00
tags: [aws, nlb, fundamentals, static-ip, eip, cross-zone, source-ip]
home_hub: nlb
---

# NLB Static IP + Cross-Zone

> Per-AZ EIP (static IP) + Cross-zone trade-off + Source IP 보존 (Layer 4 의 핵심 강점).

## 1. Static IP (EIP per AZ)

```bash
# EIP allocate per AZ
EIP_A=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
EIP_B=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
EIP_C=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)

# NLB 생성 시 매핑
aws elbv2 create-load-balancer --name bkurs-nlb \
  --type network --scheme internet-facing \
  --subnet-mappings \
    SubnetId=$SUBNET_A,AllocationId=$EIP_A \
    SubnetId=$SUBNET_B,AllocationId=$EIP_B \
    SubnetId=$SUBNET_C,AllocationId=$EIP_C \
  --ip-address-type ipv4
```

→ 3 AZ × 1 EIP. Public DNS = NLB DNS, 그러나 IP 는 EIP 고정.

## 2. Static IP 활용

```
1. Corporate Firewall Whitelist
   - 회사 firewall 이 특정 IP 만 허용
   - NLB EIP 를 whitelist
   
2. IoT Device 의 hard-coded endpoint
   - device firmware 에 IP 명시
   - DNS 변경 어려움

3. Compliance / Audit
   - "static IP" 요구 사항

4. PrivateLink Provider
   - VPC Endpoint Service 의 backend 가 NLB
```

> ALB 는 DNS 만 (IP 가변) — Static IP 가 NLB 의 핵심 차별점.

## 3. Source IP 보존

```
Client (203.0.113.10) → NLB → Target

NLB Instance Target (default):
  Target 이 보는 source IP = 203.0.113.10 (보존)

NLB IP Target + preserve_client_ip=true:
  Target 이 보는 source IP = 203.0.113.10

NLB IP Target + preserve_client_ip=false (default for IP target):
  Target 이 보는 source IP = NLB ENI IP (보존 X)

ALB:
  Target 이 보는 source IP = ALB IP
  X-Forwarded-For HTTP header 로 만
```

| Target Type | preserve_client_ip default |
| --- | --- |
| instance | true |
| ip | false (활성 가능) |
| ALB | n/a |

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=preserve_client_ip.enabled,Value=true
```

## 4. Source IP 보존의 영향

```
Application 측:
✓ 진짜 client IP (audit / rate limit / geo)
✓ X-Forwarded-For 의존 X
✗ SG 의 source rule 시 — client IP CIDR 가 다양함 (0.0.0.0/0 needed)
✗ Health check 의 출발지가 NLB IP 와 다름
```

### SG 의 일반 패턴
```bash
# Target SG inbound (Source IP 보존 시)
aws ec2 authorize-security-group-ingress --group-id $SG_TARGET \
  --protocol tcp --port 8080 \
  --cidr 0.0.0.0/0        # client IP 가 다양함

# 또는 NLB Subnet CIDR (cross-zone 차단된 경우)
--cidr 10.0.0.0/24
```

> Source IP 보존 + Public NLB = SG 의 client allow rule 어려움. NACL 활용.

## 5. Proxy Protocol v2 (대안)

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=proxy_protocol_v2.enabled,Value=true
```

→ Application 이 PROXY header 처리. Source IP 정보 binary header 로.

```
PROXY TCP4 203.0.113.10 10.0.10.5 54321 8080\r\n
```

Nginx 의 PROXY protocol:
```nginx
listen 8080 proxy_protocol;
real_ip_header proxy_protocol;
set_real_ip_from 10.0.0.0/16;
```

## 6. Cross-Zone Load Balancing

```bash
# Enable
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $NLB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

### Default OFF — 의미
```
3 AZ NLB:
- Targets in AZ-a (1 target, healthy)
- Targets in AZ-b (10 targets, healthy)  
- Targets in AZ-c (10 targets, healthy)

Cross-zone OFF:
  Client → 각 AZ 의 NLB IP 중 random
  - AZ-a 의 1 target 이 1/3 traffic 받음 (overload)
  - AZ-b/c 의 10 target 이 1/30 traffic 씩

Cross-zone ON:
  21 target 균등 분산
```

### Cost
| | |
| --- | --- |
| Cross-zone OFF (default) | DT $0 |
| Cross-zone ON | DT $0.01/GB (cross-AZ traffic) |

> Production = Cross-zone ON 권장 (균등 분산). Cost는 measure 후.

## 7. ALB 비교

| | ALB | NLB |
| --- | --- | --- |
| Cross-zone | Free + default ON | $0.01/GB + default OFF |
| Source IP | X-Forwarded-For | TCP 직접 |
| Static IP | ✗ | ✓ EIP |
| Layer | 7 | 4 |

## 8. Health Check 와 Cross-Zone

```
Cross-zone OFF:
- AZ-a 의 모든 target unhealthy → AZ-a 의 client traffic = drop
- Client 는 DNS 의 AZ-a IP 응답 받았는데 connection refused

Cross-zone ON:
- AZ-a target down 도 AZ-b/c 가 처리
```

→ Health check의 false negative 시 cross-zone ON 으로 graceful failover.

## 9. AZ Failover (Route 53)

```bash
# NLB DNS 자체가 AZ down 시 자동 unhealthy IP 제거
# Route 53 health check 와 결합:

aws route53 create-health-check --health-check-config \
  Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=$NLB_DNS,Port=443

# A record (alias) 의 health check
aws route53 change-resource-record-sets --hosted-zone-id $HZ \
  --change-batch ...
```

## 10. NLB의 ENI

```
NLB = per-AZ 1 ENI (또는 EIP)
- Subnet 의 IP 사용
- ENI 이름: ELB net/<nlb-name>/<id>
- IP / VPC Endpoint 의 endpoint IP
```

## 11. NLB Pricing — DT

```
Cross-zone OFF: 같은 AZ 의 traffic = $0
Cross-zone ON:  cross-AZ traffic = $0.01/GB
Internet:       NLB EIP 자체는 무료 (NLB 의 일부)
```

> ALB / EC2 의 standalone EIP 는 $0.005/h (2024+). NLB attached EIP 는 무료.

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Static IP 가정 (ALB) | NLB 사용 |
| Cross-zone OFF + 불균등 → overload | enable |
| Source IP 가정 (X-Forwarded-For) | preserve_client_ip + SG 0.0.0.0/0 |
| EIP 매핑 안 함 | --subnet-mappings 명시 |
| IP target + Source IP 가정 X | preserve_client_ip=true |
| Health Check 의 false negative | cross-zone ON |

## 13. 다음

- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-tls-and-mtls]]
- ▶ [[../02-features/02-privatelink-endpoint-service]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [Cross-zone Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-cross-zone.html) |
| [Source IP 보존](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#client-ip-preservation) |

## 15. 참고

- [[../nlb]]
- [[01-listener-and-target-group]]
