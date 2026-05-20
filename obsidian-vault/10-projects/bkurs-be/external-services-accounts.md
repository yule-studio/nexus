---
title: "bkurs 외부 서비스 계정 & API key 정리"
kind: ops-reference
project: bkurs-be
status: current
created_at: 2026-05-20T22:30:00+09:00
agent: engineering-agent/tech-lead
tags: [ops, accounts, external-services, secrets, bkurs]
related:
  - "[[bkurs-be]]"
  - "[[architecture-hybrid-external-services]]"
  - "[[bkurs-fe-vs-bkurs-be-gap]]"
home_hub: bkurs-be
---

# bkurs 외부 서비스 계정 & API key 정리

> production 운영을 위해 가입 / 발급 받아야 할 SaaS 계정 + 환경변수 목록.
> 각 항목은 **누가 가입 / 어떤 키 / BE 어디서 사용 / 비용 예상** 4 정보.

## 1. Tier P0 — production 띄우기 위한 필수

### 1.1 AWS 계정 (또는 GCP)

- **목적**: S3 (file storage) + RDS (DB) + SSM/Secrets Manager (secret) + IAM
- **가입 주체**: 회사 (사업자) — 결제 카드 + 사업자 등록증
- **필요 발급물**:
  - IAM user `bkurs-be-app` (least privilege)
  - Access key + Secret key
  - S3 bucket `bkurs-prod-uploads` (presigned URL 정책)
  - IAM policy: `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` on bucket
- **환경변수**:
  ```
  AWS_REGION=ap-northeast-2
  AWS_ACCESS_KEY_ID=AKIA...
  AWS_SECRET_ACCESS_KEY=...
  S3_BUCKET_NAME=bkurs-prod-uploads
  S3_PRESIGNED_URL_TTL_SECONDS=600
  ```
- **비용 예상**: S3 storage ~$0.023/GB·월, presigned URL 무료, RDS db.t3.micro ~$15/월
- **사용처 (BE)**: `FileStorageService` 구현체 (B17 → B26 PR)

### 1.2 결제 게이트웨이 — 포트원 (PortOne)

- **목적**: 카드 / 계좌이체 / 카카오페이 / 네이버페이 통합 (한국 표준)
- **가입 주체**: 회사 (사업자 등록증 + 통장사본 + 대표자 신분증)
- **가입 절차**: 가맹 계약 → 심사 (~3-5일)
- **필요 발급물**: API Key, API Secret, Channel Key, Webhook 검증 secret
- **환경변수**:
  ```
  PORTONE_API_KEY=imp_...
  PORTONE_API_SECRET=...
  PORTONE_CHANNEL_KEY=channel-key-...
  PORTONE_WEBHOOK_SECRET=...
  ```
- **Webhook URL 등록**: `https://api.bkurs.com/api/v1/webhooks/portone`
- **비용**: 결제 수수료 (카드 2.9% / 카카오페이 3.2% 등 — 가맹 계약 따라)
- **사용처 (BE)**: `PortOnePaymentGateway implements PaymentGateway` (현재 `MockPaymentGateway` 만, B4C)
- **대안**: Toss Payments — 수수료 유사, 결제 UI 깔끔

### 1.3 이메일 발송 — AWS SES 또는 SendGrid

- **목적**: 강사 invite / 학원 환영 / 공지 / 미납 독촉 / 결제 알림
- **추천**: **AWS SES** (S3 같은 계정 → 통합 관리 + 저렴)
  - 단점: 도메인 verification + DKIM/SPF 설정 필요
- **대안**: **SendGrid** (무료 100/일 → 유료 $19.95/월 ~40K)
  - 장점: 즉시 사용 가능 + dashboard 좋음
- **가입 주체**: 회사
- **필요 발급물**:
  - SES: 발신 도메인 (`noreply@bkurs.com`) 검증 + sandbox 모드 해제
  - SendGrid: API Key + Sender Identity 등록
- **환경변수 (SES)**:
  ```
  EMAIL_PROVIDER=ses
  EMAIL_FROM=noreply@bkurs.com
  AWS_SES_REGION=ap-northeast-2
  ```
- **환경변수 (SendGrid)**:
  ```
  EMAIL_PROVIDER=sendgrid
  SENDGRID_API_KEY=SG....
  EMAIL_FROM=noreply@bkurs.com
  ```
- **비용**: SES $0.10/1000건 · SendGrid 무료 100/일
- **사용처 (BE)**: `NotificationService` 구현체 (B18 → B23)

### 1.4 SMS 발송 — NCP SENS (Naver Cloud)

- **목적**: 학원 미납 독촉 / 결제 알림 / 휴대폰 인증 (선택)
- **추천**: **NCP SENS** (한국 도착률 1순위 + 카카오 알림톡 통합)
- **가입 주체**: 회사 (사업자) + KISA 발신번호 등록 (4-7일 심사)
- **필요 발급물**:
  - NCP 계정 + Access Key + Secret Key
  - SENS service registration → Service ID
  - 발신번호 등록 (회사 대표 번호 또는 070)
- **환경변수**:
  ```
  SMS_PROVIDER=ncp-sens
  NCP_ACCESS_KEY=...
  NCP_SECRET_KEY=...
  NCP_SENS_SERVICE_ID=ncp:sms:kr:...
  SMS_FROM_NUMBER=02-...
  ```
- **비용**: SMS ₩9/건 · LMS ₩30/건 · 알림톡 ₩7-8/건 (카톡 친구 우선)
- **대안**: Twilio — 글로벌, 비싸지만 sandbox 즉시
- **사용처 (BE)**: `NotificationService` SMS 부분 (B18)

### 1.5 AI — Anthropic Claude API

- **목적**: 자소서 합성 / 수행평가 분석 / 모의면접 평가 / 강사 채점 보조
- **가입 주체**: 회사 (개인 카드도 가능하지만 회사 카드 권장)
- **가입 절차**: https://console.anthropic.com — 즉시 가입, 크레딧 $5 무료
- **필요 발급물**: API Key (`sk-ant-api03-...`)
- **환경변수**:
  ```
  AI_PROVIDER=anthropic
  ANTHROPIC_API_KEY=sk-ant-api03-...
  ANTHROPIC_MODEL=claude-sonnet-4-6
  ANTHROPIC_MAX_TOKENS=4096
  AI_REQUEST_TIMEOUT_SECONDS=60
  ```
- **모델 선택**:
  - **claude-sonnet-4-6**: 권장 (균형 — $3/1M input, $15/1M output)
  - **claude-haiku-4-5**: 비용 절감 ($0.25/1M input)
  - **claude-opus-4-7**: 고품질 ($15/1M input)
- **비용 예상**: 학생 1명당 월 ~$0.5-2 (자소서 5회 + 면접 10회 가정)
- **사용처 (BE)**: `AiAssistant` 구현체 (B19 → B24)

### 1.6 STT — OpenAI Whisper 또는 NCP Clova Speech

- **목적**: 모의면접 / 숙제 녹음 → 텍스트 전사
- **옵션 A — OpenAI Whisper API**:
  - 가입: https://platform.openai.com
  - 키: `OPENAI_API_KEY`
  - 비용: $0.006/분
  - 정확도: 매우 높음 (다국어)
- **옵션 B — NCP Clova Speech** (추천 — 한국어 특화):
  - 가입: NCP 계정 + Clova Speech 서비스 활성화
  - 키: `NCP_CLOVA_INVOKE_URL`, `NCP_CLOVA_SECRET`
  - 비용: ~₩7/분
  - 정확도: 한국어 발음 / 한국 이름 등 더 정확
- **환경변수 (옵션 B)**:
  ```
  STT_PROVIDER=clova
  NCP_CLOVA_INVOKE_URL=https://clovaspeech-...
  NCP_CLOVA_SECRET=...
  ```
- **사용처 (BE)**: `SttService` 구현체 (B20 → B25)

## 2. Tier P1 — 정식 출시 후

### 2.1 OAuth 소셜 로그인

- **카카오 로그인** (한국 1순위):
  - 가입: https://developers.kakao.com
  - 앱 등록 + 사이트 도메인 / Redirect URI 등록
  - 키: `KAKAO_CLIENT_ID`, `KAKAO_CLIENT_SECRET`
  - 비용: 무료
- **Google 로그인**:
  - GCP Console OAuth 2.0
  - 키: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
  - 비용: 무료
- **사용처 (BE)**: `AuthService` 확장 (B28)

### 2.2 모니터링 — Sentry

- **목적**: 에러 추적 + alerting + release 별 trend
- **가입**: https://sentry.io (개발자 1명 무료, team plan $26/월)
- **키**: `SENTRY_DSN` (project 별)
- **환경변수**:
  ```
  SENTRY_DSN=https://...@sentry.io/...
  SENTRY_ENVIRONMENT=production
  SENTRY_RELEASE=${git sha}
  ```
- **사용처**: Spring Boot Actuator + sentry-spring-boot-starter (B29)

### 2.3 메트릭 — Grafana Cloud 또는 Datadog

- **Grafana Cloud Free**: Prometheus + Loki + 10k series 무료
- **Datadog**: 호스트 당 $15/월
- 본 단계는 actuator + Prometheus exporter 만으로 시작 가능

## 3. Tier P2 — 모바일 앱 출시 시

### 3.1 Firebase Cloud Messaging (FCM) — Push 알림

- 가입: Firebase Console + Google 계정
- 키: `FIREBASE_SERVICE_ACCOUNT_JSON` (BE 에 파일로)
- iOS: APNS 인증서 별도

## 4. 환경 별 secret 관리

| 환경 | 저장소 |
|---|---|
| local | `.env.local` (gitignore) — `.env.example` 가 참고 |
| CI | GitHub Actions Secrets |
| staging | AWS SSM Parameter Store (`/bkurs/staging/*`) |
| production | AWS SSM Parameter Store (`/bkurs/prod/*`) 또는 Vault |

**rotation 정책**:
- API key 6개월 마다 rotate (or 즉시 if 노출 감지)
- DB password 1년 마다 rotate
- JWT secret 변경 시 모든 토큰 무효화 (downtime 필요) → 신규 토큰 발급 후 기존 grace 7일

## 5. Secret 누수 (현재 상태) — 즉시 조치

git history 에 노출된 비번:
- `13950991*` (MySQL `mw` user, root 둘 다)
- commit 3개: `e39c560`, `777eb2e`, `4138a18` (origin push 됨)

**조치**:
1. ✅ 현재 작업: `docker-compose.yml` / `application-local.yml` env var 화 (별도 PR)
2. ⚠️ MySQL `mw` 비번 rotate — 로컬 DB 만 사용 중이면 즉시 영향 없으나, public 노출되어 있으므로 rotate 권장
3. ⚠️ git history rewrite (`git filter-repo`) — 협업자 영향 있어 신중 검토. 비번 rotate 면 history 는 둬도 됨 (이미 무효)

## 6. 비용 시뮬레이션 (학원 100개 / 학생 5000명 / 월)

| 항목 | 비용 |
|---|---|
| AWS S3 (영상 100GB) | ~$2.3 |
| RDS db.t3.small | ~$30 |
| SES (월 50K 이메일) | ~$5 |
| NCP SMS (월 20K 알림톡) | ~₩160K (= ~$120) |
| Claude Sonnet (월 5M token) | ~$30 |
| Clova Speech (월 5K 분) | ~₩35K (= ~$26) |
| 결제 수수료 (월 2K 건 × 평균 ₩50K × 3%) | ~₩3M (= ~$2.2K) |
| Sentry team | $26 |
| **합계 (운영 비용, 결제 수수료 제외)** | **~$240/월** |

결제 수수료는 매출 비례 (학원 구독비) 라 별도 회계.

## 관련

- [[architecture-hybrid-external-services]] — Hybrid 아키텍처 (각 외부 서비스 어디서 어떻게 호출)
- [[bkurs-fe-vs-bkurs-be-gap]] — FE 에 있고 BE 에 없는 기능 갭
- [[bkurs-be]] — 프로젝트 hub
