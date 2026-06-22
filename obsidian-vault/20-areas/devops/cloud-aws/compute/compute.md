---
title: "AWS Compute (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:05:00+09:00
tags:
  - aws
  - compute
  - hub
---

# AWS Compute (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

### 1.1. VM / 기본
| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **EC2** | [[ec2/ec2\|ec2]] | 가상 머신 (VM) — 가장 기본 |
| **Lightsail** | [[lightsail/lightsail\|lightsail]] | 정액제 단순 VPS — 개인 / 소규모 (DigitalOcean 대응) |

### 1.2. 컨테이너
| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **ECS** | [[ecs/ecs\|ecs]] | Container 오케스트레이션 — Fargate / EC2 launch |
| **EKS** | [[eks/eks\|eks]] | 매니지드 Kubernetes — 표준 / 이식성 |
| **ECR** | [[ecr/ecr\|ecr]] | 컨테이너 이미지 레지스트리 (+scan / lifecycle) |
| **App Mesh** | [[app-mesh/app-mesh\|app-mesh]] | 서비스 메시 (Envoy) — 트래픽 라우팅 / 관측 |
| **Copilot** | [[copilot/copilot\|copilot]] | ECS / Fargate / App Runner 배포 CLI |

### 1.3. 서버리스 / PaaS
| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Lambda** | [[lambda/lambda\|lambda]] | Serverless 함수 — event-driven |
| **App Runner** | [[app-runner/app-runner\|app-runner]] | 컨테이너 / 소스 → 자동 빌드·배포 PaaS (Cloud Run 대응) |
| **Elastic Beanstalk** | [[elastic-beanstalk/elastic-beanstalk\|elastic-beanstalk]] | 1세대 PaaS — zip 업로드 → EC2+ALB+ASG 자동 |

### 1.4. 배치 / 스케일
| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Batch** | [[batch/batch\|batch]] | 대량 배치 작업 큐 + 컴퓨팅 자동 확장 (HPC / ML / ETL) |
| **Auto Scaling** | [[auto-scaling/auto-scaling\|auto-scaling]] | 수요(metric/schedule/예측)에 따라 용량 자동 조정 |

### 1.5. 엣지 / 하이브리드
| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Local Zones** | [[local-zones/local-zones\|local-zones]] | 대도시 가까운 mini region (저지연 < 10ms) |
| **Wavelength** | [[wavelength/wavelength\|wavelength]] | 통신사 5G 망 안에 컴퓨팅 (모바일 1-hop) |
| **Outposts** | [[outposts/outposts\|outposts]] | 온프레미스 데이터센터에 AWS 인프라 설치 |

---

## 2. 선택 가이드

| 시나리오 | 추천 |
| --- | --- |
| 전통 응용 / 큰 서버 | EC2 |
| Container + AWS native | ECS (Fargate) |
| Container + 표준 / 이식성 | EKS (별도, Kubernetes hub) |
| Event-driven / 짧은 작업 | Lambda |
| 정적 사이트 | S3 + CloudFront ([[../storage/s3]]) |
| 거대 batch | Batch (별도) |
| HPC / GPU | EC2 GPU instance |

---

## 3. 가격 모델

| 모델 | 의미 |
| --- | --- |
| On-Demand | 사용한 만큼 (기본) |
| Reserved (RI) | 1년/3년 약정 — 30-70% 할인 |
| Savings Plan | 시간당 약정 (더 유연) |
| Spot | 입찰 — 최대 90% 할인 + 강제 종료 |
| Dedicated Host | 단독 물리 서버 (라이선스) |

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../network/vpc]] — Compute 가 VPC 안에 있음
- [[../security/iam]] — IAM role 로 권한
