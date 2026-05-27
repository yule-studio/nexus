---
title: "EC2 02-1 — ENI / Private IP / Public IP / Elastic IP"
kind: knowledge
status: current
created_at: 2026-05-22T15:10:00+09:00
tags: [aws, ec2, networking, eni, ip, elastic-ip]
home_hub: ec2
---

# ENI / IP

> ENI = 가상 네트워크 카드. Private IP (subnet 내) / Public IP (auto) / Elastic IP (영구) / IPv6. Instance 당 ENI 여러 개 가능.

## 1. ENI (Elastic Network Interface)

| 필드 | 의미 |
| --- | --- |
| Primary Private IP | subnet CIDR 내 자동/수동 |
| Secondary Private IP | 같은 ENI 에 추가 (최대 type 따라) |
| Public IP (auto) | subnet `auto-assign-public-ip` |
| Elastic IP | 영구 (분리 가능) |
| IPv6 | dual-stack VPC |
| SG | 5 개 max attach |
| MAC | 고정 |
| Source/Dest check | NAT instance 시 disable |

```bash
aws ec2 create-network-interface --subnet-id subnet-xxx \
  --groups sg-xxx --private-ip-address 10.0.1.50

aws ec2 attach-network-interface --network-interface-id eni-xxx \
  --instance-id i-xxx --device-index 1
```

## 2. Instance 당 ENI / IP 한도

| Type | ENI max | IP per ENI |
| --- | --- | --- |
| t3.nano | 2 | 2 |
| t3.large | 3 | 12 |
| m7g.large | 3 | 10 |
| m7g.16xlarge | 8 | 50 |
| c7g.metal | 50 | 50 |

→ Container 의 awsvpc (per task ENI) 한도 — ENI Trunking 활성 시 5x.

## 3. Primary vs Secondary ENI

| | Primary (eth0) | Secondary (eth1+) |
| --- | --- | --- |
| 분리 | ✗ (instance 종료 시만) | ✓ (running 도 가능) |
| 활용 | 일반 | failover / multi-homed / appliance |

## 4. Public IP

Subnet 의 `MapPublicIpOnLaunch=true` 일 때 자동 부여:
```bash
aws ec2 modify-subnet-attribute --subnet-id subnet-xxx \
  --map-public-ip-on-launch
```

- stop → start 시 **다른 IP** 부여됨
- terminate 시 release
- 1 IPv4 = $0.005/h (2024-02 부터 — public IPv4 유료)

## 5. Elastic IP

```bash
# Allocate
aws ec2 allocate-address --domain vpc
# {"PublicIp":"3.34.x.x","AllocationId":"eipalloc-xxx"}

# Associate
aws ec2 associate-address --instance-id i-xxx --allocation-id eipalloc-xxx

# 분리
aws ec2 disassociate-address --association-id eipassoc-xxx

# 반환
aws ec2 release-address --allocation-id eipalloc-xxx
```

| 가격 |
| --- |
| Attached + running = $0.005/h ($3.6/월) |
| Detached / stopped 도 $0.005/h ($3.6/월) — **idle EIP 비용 주의** |
| Additional EIP per instance = 같은 가격 |

> 사용 안 하면 **반드시 release**.

## 6. IPv6

```bash
aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxx \
  --amazon-provided-ipv6-cidr-block

aws ec2 associate-subnet-cidr-block --subnet-id subnet-xxx \
  --ipv6-cidr-block 2406:da12:cef:cd00::/64
```

장점:
- IPv6 = **무료** (IPv4 처럼 시간 과금 없음)
- 비용 절감 트렌드 (egress 줄임)

## 7. Source/Dest check

NAT instance / 방화벽 appliance 시:
```bash
aws ec2 modify-instance-attribute --instance-id i-xxx \
  --no-source-dest-check
```

→ instance 가 본인 IP 아닌 packet 도 forward 가능.

## 8. ENI Hotplug

running instance 에 ENI attach/detach 가능:
- 다중 NIC 격리
- High Availability — secondary ENI 분리 → 다른 instance 에 attach
- Floating IP 패턴

## 9. Network Performance / EBS Burst

| Type | EBS Bandwidth | Network |
| --- | --- | --- |
| t3.medium | up to 2.085 Gbps | up to 5 Gbps |
| m7g.large | up to 10 Gbps | up to 12.5 Gbps |
| m7g.16xlarge | 30 Gbps | 50 Gbps |
| hpc7g.16xlarge | 50 Gbps | 200 Gbps EFA |

## 10. Enhanced Networking

| | SR-IOV (legacy) | ENA (현대) | EFA (HPC) |
| --- | --- | --- | --- |
| 대역폭 | 10 Gbps | 25~100 Gbps | 200~400 Gbps |
| Latency | μs | μs | 사용자공간 |
| 활성 | 자동 | 자동 (Nitro) | optional |

## 11. Prefix Delegation

```bash
aws ec2 assign-private-ip-addresses --network-interface-id eni-xxx \
  --ipv4-prefix-count 4
```

→ ENI 당 IP 16 / 32 prefix → Container density ↑ (EKS / ECS awsvpc).

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| stop → start 시 public IP 변동 | Elastic IP |
| Elastic IP detach 후 release 안 함 | release |
| IPv4 비용 폭주 | IPv6 마이그 |
| ENI Trunking 비활성 | 활성 (per task 5x) |
| Source/Dest check on NAT | disable |
| SG 5 개 초과 attach 시도 | quota 증액 |

## 13. CDK / Terraform

```hcl
resource "aws_network_interface" "extra" {
  subnet_id       = aws_subnet.private.id
  private_ips     = ["10.0.1.50"]
  security_groups = [aws_security_group.app.id]
}

resource "aws_network_interface_attachment" "extra" {
  instance_id          = aws_instance.app.id
  network_interface_id = aws_network_interface.extra.id
  device_index         = 1
}

resource "aws_eip" "app" {
  domain   = "vpc"
  instance = aws_instance.app.id
}
```

## 14. 다음

- ▶ [[02-security-group-and-nacl]]
- ▶ [[03-placement-group]]
- ▶ [[../../network/vpc/vpc]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [ENI 공식](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html) |
| [IPv4 비용 발표](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/) |

## 16. 참고

- [[../ec2]]
- [[../01-fundamentals/02-instance-types]]
- [[../../../network/vpc/vpc]]
