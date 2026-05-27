---
title: "EC2 01-3 — Pricing Models (On-Demand / RI / SP / Spot / Dedicated)"
kind: knowledge
status: current
created_at: 2026-05-22T15:06:00+09:00
tags: [aws, ec2, fundamentals, pricing, spot, savings-plan, reserved]
home_hub: ec2
---

# EC2 Pricing Models

> 5 모델 × Capacity Reservation. baseline 60~80% = SP/RI / spike = OD / 내결함 batch = Spot 의 mix 가 일반적.

## 1. 5 모델 한눈에

| 모델 | 할인 | 약정 | 강제 종료 | 권장 use |
| --- | --- | --- | --- | --- |
| **On-Demand** | 0% | 없음 | 없음 | 시작 / 짧은 작업 / 불확실 |
| **Reserved Instance** | ~70% (3y all-up) | 1y/3y + type 고정 | 없음 | 같은 type 장기 운영 |
| **Savings Plan** | ~50% (3y all-up) | 1y/3y + $/h | 없음 | type/region 변동 허용 |
| **Spot** | 70~90% | 없음 (시장가) | 2분 통지 | 내결함 batch / dev |
| **Dedicated Host / Instance** | premium | 옵션 | 없음 | 라이센스 BYOL |

## 2. On-Demand

```bash
aws ec2 run-instances --instance-type m7g.large --count 1 ...
```

- 초 단위 과금 (최소 60초)
- baseline 가격 (다른 모델 비교 기준)
- 시작/종료 free

## 3. Reserved Instance (RI)

3 차원 약정:
- **약정 기간** — 1y / 3y
- **결제 방식** — No / Partial / All Upfront
- **scope** — Region / AZ (capacity 보장 + AZ scope)

3 type:
- **Standard RI** — 최대 할인 (3y all = 70%), type 고정
- **Convertible RI** — type 변경 가능, 할인 작음
- **Scheduled RI** — 시간대 정함 (EOL)

```bash
aws ec2 describe-reserved-instances-offerings \
  --instance-type m7g.large --offering-class standard \
  --offering-type "All Upfront" --product-description "Linux/UNIX"
```

## 4. Savings Plan (SP)

**$/hour 약정** — type/region/family 자유:

### Compute Savings Plan
- EC2 + Fargate + Lambda 다 커버
- 모든 region / family
- 할인 ~17% (1y NU) ~ ~50% (3y AU)

### EC2 Instance Savings Plan
- 특정 family + region 고정
- 할인 더 큼 (~28% (1y NU) ~ ~72% (3y AU))

```bash
aws savingsplans describe-savings-plans-offerings \
  --plan-types Compute --offering-payment-options "All Upfront" \
  --term 3
```

> 신규 약정 = **Savings Plan 우선**. RI 보다 유연. type 변경 자유.

## 5. Spot Instance

```bash
aws ec2 run-instances --instance-market-options "MarketType=spot,SpotOptions={MaxPrice=0.05,InstanceInterruptionBehavior=terminate}" ...
```

### 동작
- 시장가 (변동) — 90% 까지 할인
- AWS 가 capacity 필요 시 **2분 사전 통지 후 회수**
- termination / stop / hibernate 중 선택

### Spot Fleet / EC2 Fleet
다양한 instance type / AZ pool 에서 capacity 확보:
```json
{
  "AllocationStrategy": "capacityOptimized",
  "TargetCapacity": 100,
  "LaunchSpecifications": [
    { "InstanceType": "m7g.large", "SubnetId": "..." },
    { "InstanceType": "m6g.large", "SubnetId": "..." },
    { "InstanceType": "m6i.large", "SubnetId": "..." }
  ]
}
```

> AllocationStrategy=`capacityOptimized` 이 interruption rate 가장 낮음.

### 적합한 워크로드
- ✓ stateless web (auto-replaced)
- ✓ CI / batch / build
- ✓ ML training (checkpoint)
- ✓ render farm
- ✗ DB primary
- ✗ stateful session

### Interruption 처리
```bash
# IMDS 에서 interruption notice 확인
curl http://169.254.169.254/latest/meta-data/spot/instance-action
# {"action":"terminate","time":"2026-05-22T15:10:00Z"}
```

→ graceful shutdown / draining.

## 6. Dedicated Host / Instance

| | Dedicated Host | Dedicated Instance |
| --- | --- | --- |
| host 가시성 | ✓ (host ID, socket, core) | ✗ |
| BYOL 라이센스 | ✓ | △ |
| 가격 | host 단위 | per instance + premium |
| 용도 | Oracle / SQL Server / VMware | regulatory tenant 격리 |

## 7. Capacity Reservation

- type / AZ 에 capacity 확보 (취소 자유)
- 가격 = On-Demand (할인 없음, 자리 보장)
- SP/RI 와 결합 → capacity + 할인

## 8. mix 전략

```
baseline 60-80%  → Compute Savings Plan 3y AU
peak 20-40%      → On-Demand
batch / dev      → Spot Fleet capacityOptimized
critical baseline→ Capacity Reservation + SP
```

## 9. 비용 비교 (m7g.large, Seoul, 730h/월)

| 모델 | 월 비용 | 절감 |
| --- | --- | --- |
| On-Demand | $62.5 | — |
| Compute SP 1y AU | ~$47 | 25% |
| Compute SP 3y AU | ~$32 | 49% |
| EC2 SP 3y AU (region/family 고정) | ~$28 | 55% |
| Standard RI 3y AU | ~$19 | 70% |
| Spot (capacityOptimized) | ~$15 | 76% |

## 10. Free Tier (계정 1년)

- t2.micro 또는 t3.micro 750h/월
- 30 GB EBS gp2
- 750h public IPv4

## 11. Cost Explorer 활용

```
Cost Explorer → Recommendations
- Reserved Instance 추천
- Savings Plan 추천
- Right-sizing 추천
```

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| 100% baseline 약정 | 60-80% 만 |
| 3년 전체 약정 | spike 는 OD / Spot |
| Spot DB | stateless 만 |
| RI 만 (Convertible 안 함) | SP 우선 |
| Capacity Reservation 잊음 | unused = OD 비용 만 |

## 13. bkurs-be 권장 mix

```
ECS Fargate ARM64:
- Compute SP 3y AU (60% baseline) → -50%
- FARGATE_SPOT (40%)              → -70%
- 부족 시 On-Demand 자동 fallback (Capacity Provider)
```

월 비용: $200 → **약 $90**.

## 14. 다음

- ▶ [[04-amis-and-bootstrap]]
- ▶ [[../02-networking/01-eni-and-ip]]
- ▶ [[../../ecs/04-operations/04-cost-optimization]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [Savings Plan vs RI](https://aws.amazon.com/savingsplans/compute-pricing/) |
| [Spot Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html) |

## 16. 참고

- [[../ec2]]
- [[02-instance-types]]
- [[../../ecs/04-operations/04-cost-optimization]]
