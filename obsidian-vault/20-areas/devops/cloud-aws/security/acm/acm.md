---
title: "AWS Certificate Manager (ACM)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:44:00+09:00
tags:
  - aws
  - security
  - tls
  - certificate
---

# AWS Certificate Manager (ACM)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | ACM hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**무료 TLS 인증서** 발급 + **자동 갱신**. ALB / CloudFront / API Gateway 의 HTTPS 의 표준 진입점.

---

## 2. 왜 / 언제

- HTTPS 가 필요한 모든 곳 (브라우저 / API / 사내 도구)
- Let's Encrypt 자동화 (certbot 등) 운영을 안 하고 싶을 때
- 다중 도메인 / wildcard 인증서가 필요할 때
- 인증서 갱신 알람 / 운영 부담을 제거하고 싶을 때

> AWS 안에서 쓰는 HTTPS 는 사실상 ACM 이 디폴트.

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Public certificate** | 공개 인터넷용 (Amazon Trust Services 발급) — **무료** |
| **Private certificate (PCA)** | 사설 CA — **유료** (월 $400+) |
| **DNS validation** | Route 53 의 CNAME → **자동 갱신 가능** (권장) |
| **Email validation** | 등록인 이메일 클릭 → 1년마다 수동 갱신 (지양) |
| **SAN** | 한 인증서에 여러 도메인 (`example.com`, `*.example.com`, `api.example.com`) |

---

## 4. 표준 패턴

### 4.1 ALB / CloudFront HTTPS

```
Route 53 (example.com)
    ↓
CloudFront / ALB ─── ACM cert (DNS validated)
    ↓
응용
```

→ 인증서가 LB / CDN 에 attach. 응용은 HTTP 만 받음 (TLS termination).

### 4.2 region 주의

| 사용처 | 인증서 region |
| --- | --- |
| ALB / NLB / API Gateway (regional) | **같은 region** (예: ap-northeast-2) |
| **CloudFront** | **반드시 us-east-1 (N. Virginia)** |

→ CloudFront 쓸 때 자주 헷갈리는 함정. us-east-1 에서 한 번 더 발급.

### 4.3 발급 흐름 (DNS validation)

```bash
aws acm request-certificate \
  --domain-name "example.com" \
  --subject-alternative-names "*.example.com" \
  --validation-method DNS \
  --region ap-northeast-2

# 출력된 CNAME 을 Route 53 에 추가 → "Issued" 상태
```

Route 53 도메인이면 콘솔에서 "Create records in Route 53" 버튼 한 번이면 끝.

---

## 5. EC2 에 직접 attach 가능?

**불가능**. ACM 인증서는 **ALB / NLB / CloudFront / API Gateway / Elastic Beanstalk / CloudFormation 등 AWS 매니지드 서비스에만 attach** 가능.

- EC2 / nginx 에 직접 쓰려면 → **Let's Encrypt** (certbot) 사용.
- 또는 **ALB 뒤에 EC2** 로 구조 변경 (ALB 가 TLS 종료).

---

## 6. 함정 / 비용

- **무료** (Public). 갱신도 자동.
- CloudFront 는 **us-east-1** — 잊으면 인증서가 안 보임.
- DNS validation CNAME 을 **삭제 X** — 삭제하면 갱신 시점에 fail.
- `*.example.com` 와 `example.com` 모두 필요하면 SAN 으로 둘 다 포함.
- 만료 30일 전 EventBridge / CloudWatch 알람 자동.

---

## 7. 관련

- [[../security|↑ Security]]
- [[../../network/cloudfront/cloudfront]]
- [[../../network/alb/alb]]
- [[../../network/route53/route53]]
- [[../../network/api-gateway/api-gateway]]
