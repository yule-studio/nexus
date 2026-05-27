---
title: bkurs-be — 외부 SaaS 가이드 (발급 / 설정 / env 매핑)
kind: guide
status: active
created_at: 2026-05-21
tags: [bkurs-be, external-saas, deployment, secrets]
related:
  - "[[bkurs-be]]"
  - "[[architecture-hybrid-external-services]]"
  - "[[external-services-accounts]]"
  - "[[env-deployment-guide]]"
home_hub: "[[bkurs-be]]"
---

# bkurs-be 외부 SaaS 가이드

> 📍 **Read path**: [[getting-started]] (전체 진입점) → **본 노트** (SaaS 발급) →
> [[env-deployment-guide]] (받은 키 어디에 설정) → [[bkurs-be]] (코드 SSoT).
>
> 본 노트는 **무엇을 발급받고 어디서 받는지** 만 다룬다. 받은 키를 실제로 어디에 박는지
> (local `.env.local` / prod EKS Secret) 는 [[env-deployment-guide]] §3 참조.

## 1. 핵심 요약

bkurs-be 가 의존하는 외부 SaaS **7가지** + 각 서비스별:
1. **무엇을** 위해 쓰는지 (BE 도메인 / 흐름)
2. **발급** 방법 (회원가입 → API key 발급 단계)
3. **bkurs-be 설정** (env 변수 + application.yml prefix)
4. **검증** (실제 요청 한 번 보내 보는 법)
5. **장애 시 대안** (mock 으로 떨어뜨리는 스위치)

> 본 가이드는 **bkurs-be 의존성만** 다룬다. infra 자체 (RDS / EKS / ELB / Route53 / Sentry alert 룰)
> 는 `company-infra` 레포의 README + Terraform 가이드 참조.

| # | 서비스 | 도메인 | provider 선택지 |
|---|--------|--------|-----------------|
| 1 | DB | 핵심 | MySQL 8.x (RDS / 로컬 Docker) |
| 2 | 메시지 큐 | 비동기 | RabbitMQ (InMemory fallback 가능) |
| 3 | 객체 저장 | 파일 업로드 | AWS S3 (Mock fallback 가능) |
| 4 | 이메일 | 알림 | AWS SES **또는** SendGrid |
| 5 | SMS | 알림 (선택) | NCP SENS **또는** Twilio |
| 6 | AI | 진단 / 분석 | Anthropic Claude (Mock fallback 가능) |
| 7 | STT | 음성 → 텍스트 | NCP Clova **또는** OpenAI Whisper |
| 8 | 결제 | 학원 구독 | PortOne (Mock fallback 가능) |
| 9 | OAuth | 소셜 로그인 (선택) | Kakao / Google |
| 10 | 모니터링 (선택) | 에러 추적 | Sentry |

`Mock fallback 가능` = 해당 env 가 비어 있으면 InMemory / Mock impl 이 동작.
local / dev 환경은 Mock 로 시작해도 됨. prod 는 모두 실제 provider.

---

## 2. 서비스별 가이드

### 2.1. AWS S3 (파일 저장)

**용도**: 학생 음성 녹음 / 첨부 파일 / 생기부 PDF presigned URL 발급. `FileStorageService`
구현체 `S3FileStorage` 가 호출.

**발급**:
1. https://aws.amazon.com 회원가입 + 결제 카드 등록
2. IAM Console → User 생성 (`bkurs-app`) → Programmatic access 키 발급
3. S3 Console → Bucket 생성 (`bkurs-prod-uploads`, region `ap-northeast-2`)
4. Bucket CORS 설정:
   ```json
   [{"AllowedOrigins":["https://bkurs.com"],"AllowedMethods":["GET","PUT"],
     "AllowedHeaders":["*"],"ExposeHeaders":["ETag"],"MaxAgeSeconds":3000}]
   ```
5. IAM User 에 S3 정책 부여 (least privilege):
   ```json
   {"Version":"2012-10-17","Statement":[{
     "Effect":"Allow","Action":["s3:PutObject","s3:GetObject","s3:DeleteObject"],
     "Resource":"arn:aws:s3:::bkurs-prod-uploads/*"}]}
   ```

**bkurs-be env**:
```bash
AWS_REGION=ap-northeast-2
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET_NAME=bkurs-prod-uploads
S3_PRESIGNED_URL_TTL_SECONDS=600   # 10분 유효
```

**검증**:
```bash
curl -X POST https://api.bkurs.com/api/v1/files/upload-url \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"filename":"test.mp3","contentType":"audio/mpeg"}'
# 응답 의 uploadUrl 로 PUT 시 200 OK 면 성공
```

**fallback**: `AWS_ACCESS_KEY_ID` 비어 있으면 `MockFileStorage` (인메모리, presigned URL 가짜)
가 활성화. 단위 / 통합 테스트는 모두 Mock 으로 실행.

---

### 2.2. AWS SES **또는** SendGrid (이메일)

**용도**: 학원 가입 승인 알림, 강사 초대 토큰 이메일, 비밀번호 리셋 등.
`NotificationService` 구현체.

**선택 기준**:
- **SES**: AWS 이미 쓰는 경우 (S3 와 동일 IAM 사용) → SES 권장. 가격 우수 ($0.10/1k).
  단, **sandbox 해제** 필요 (가입 직후엔 verified 주소만 발송 가능 → AWS 지원 티켓).
- **SendGrid**: AWS 외부에서 단독 운영, API key 1개로 끝, 자체 dashboard 우수.
  무료 100통/일.

#### 2.2.1. AWS SES

**발급**:
1. AWS Console → SES → 도메인 등록 (`bkurs.com`) → SPF/DKIM TXT 추가 → verified
2. Sandbox 해제 요청 (Production access) — 24~48시간 소요
3. From 이메일 verified (`noreply@bkurs.com`)

**env**:
```bash
EMAIL_PROVIDER=ses
EMAIL_FROM=noreply@bkurs.com
AWS_SES_REGION=ap-northeast-2
# AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY 는 S3 와 공유 (IAM 정책에 SES 권한 추가)
```

#### 2.2.2. SendGrid

**발급**:
1. https://sendgrid.com 회원가입 (free tier)
2. Settings → Sender Authentication → Single Sender (`noreply@bkurs.com`) verified
3. Settings → API Keys → Create API Key (Full Access 또는 Mail Send only)

**env**:
```bash
EMAIL_PROVIDER=sendgrid
EMAIL_FROM=noreply@bkurs.com
SENDGRID_API_KEY=SG.xxxxxxxxxxxxxxxxxxxxxx
```

**검증** (공통):
```bash
curl -X POST https://api.bkurs.com/api/v1/owner/me/teacher-invites \
  -H "Authorization: Bearer $OWNER_TOKEN" \
  -d '{"email":"test+invite@example.com"}'
# 응답 200 + 발송 receipt log 확인
```

**fallback**: provider env 비어 있으면 `MockNotificationService` 활성화 (log 출력만).

---

### 2.3. Anthropic Claude (AI)

**용도**: 진로 진단 AI 해석, 수행평가 답안 분석, 면접 답변 코칭. `AiAssistant` 구현체.

**발급**:
1. https://console.anthropic.com 회원가입
2. Billing → 결제 수단 등록 + 초기 크레딧 ($5+) 충전
3. API Keys → "Create Key" → 사용 목적 메모 (`bkurs-prod`)
4. **rate limit 확인** — tier 1 은 분당 50 req. tier 상승은 사용량 누적 후 자동.

**env**:
```bash
AI_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-api03-xxxxxxxxxxxxx
ANTHROPIC_MODEL=claude-sonnet-4-6   # 또는 claude-haiku-4-5 (비용 1/5)
ANTHROPIC_MAX_TOKENS=4096
AI_REQUEST_TIMEOUT_SECONDS=60
```

**application.yml prefix**: `bkurs.external.ai.anthropic.*` (참조: `AnthropicProperties.java`).

**검증**:
```bash
# 학생이 진단 complete 한 직후 강사 측 endpoint 호출
curl https://api.bkurs.com/api/v1/teachers/me/students/$SID/concept-diagnosis \
  -H "Authorization: Bearer $TOKEN"
# 응답의 aiAnalysisJson 필드에 Claude 응답 포함 확인
```

**fallback**: `ANTHROPIC_API_KEY` 비어 있으면 `MockAiAssistant` 활성화 (정형 텍스트 반환).
local / staging 권장.

**비용 절감 팁**: prompt 캐싱 (5분 TTL) + max_tokens 보수적 + Haiku 모델 사용.
자세한 cost 모델은 [[../research-cost-model-claude|cost-model-claude]] 참조.

---

### 2.4. NCP Clova **또는** OpenAI Whisper (STT)

**용도**: 학생 음성 녹음 → 텍스트 전사. `SttService` 구현체.

**선택 기준**:
- **Clova**: 한국어 정확도 최상. 의료/입시 용어 인식 우수. 비싼 편 (분당 ₩50).
- **Whisper**: OpenAI API, 다국어 일관, $0.006/분. 한국어 정확도 Clova 보다 약간 떨어짐.

#### 2.4.1. NCP Clova

**발급**:
1. https://www.ncloud.com 회원가입 + 신용카드 등록
2. AI·NAVER API → Clova Speech 신청 → Service ID 발급
3. Console → Clova Speech → Invoke URL + Secret 확인

**env**:
```bash
STT_PROVIDER=clova
NCP_CLOVA_INVOKE_URL=https://clovaspeech-gw.ncloud.com/external/v1/.../recognizer/url
NCP_CLOVA_SECRET=xxxxxxxxxxxxxxxx
```

#### 2.4.2. OpenAI Whisper

**발급**:
1. https://platform.openai.com 회원가입 + 결제 수단 등록
2. API Keys → Create new secret key (`bkurs-prod`)

**env**:
```bash
STT_PROVIDER=whisper
OPENAI_API_KEY=sk-xxxxxxxxxxxxx
```

**검증**:
```bash
# 학생 음성 업로드 → 전사 결과 polling
curl https://api.bkurs.com/api/v1/students/me/homework/$HID/transcript \
  -H "Authorization: Bearer $TOKEN"
```

**fallback**: provider env 비어 있으면 `MockSttService` (filename echo).

---

### 2.5. PortOne (결제)

**용도**: 학원장 구독 결제 (월/연 단위). `PaymentGateway` 구현체.

**발급**:
1. https://portone.io 회원가입 (사업자등록증 필요)
2. PG 사 계약 (이니시스 / KCP / 토스페이먼츠 중 선택) — 본인 인증 + 정산 계좌
3. PortOne Console → 채널 생성 → API Key / Secret / Channel Key / Webhook Secret 확보

**env**:
```bash
PAYMENT_PROVIDER=portone
PORTONE_API_KEY=imp_xxxxxxxxxxxxx
PORTONE_API_SECRET=xxxxxxxxxxxxxxxx
PORTONE_CHANNEL_KEY=channel-key-xxxxxxxxxxxxx
PORTONE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxx
```

**검증**:
```bash
# 학원장 구독 결제 시뮬
curl -X POST https://api.bkurs.com/api/v1/academies/me/subscription \
  -H "Authorization: Bearer $OWNER_TOKEN" \
  -d '{"planId":1,"billingCycleMonths":1}'
# 응답 의 paymentUrl 로 결제 페이지 진입 → 카드 입력 → 성공 시 webhook 수신
```

**fallback**: `PAYMENT_PROVIDER=mock` (default) — 결제 즉시 SUCCESS 처리.

> ⚠️ PortOne 은 prod 전환 시 **PG 사 본 계약 + 매출 정산 계좌 등록** 필요. 최소 2주 소요.

---

### 2.6. NCP SENS / Twilio (SMS, 선택)

학원장 알림 / 강사 초대를 이메일 외 SMS 로도 보내고 싶을 때. 현재 default disabled.

**NCP SENS**:
```bash
SMS_PROVIDER=ncp-sens
NCP_ACCESS_KEY=...
NCP_SECRET_KEY=...
NCP_SENS_SERVICE_ID=ncp:sms:kr:...
SMS_FROM_NUMBER=02-xxx-xxxx  # 발신번호 사전 등록 필수
```

**Twilio**: 해외 발송. https://twilio.com → API key + From number 발급.

---

### 2.7. Sentry (모니터링, 선택)

**용도**: 5xx 에러 / 미처리 예외 추적. `SentryConfig` 활성화.

**발급**:
1. https://sentry.io 회원가입
2. Projects → Create Project → java-spring → DSN 복사

**env**:
```bash
SENTRY_DSN=https://xxxxxx@oxxxxxx.ingest.sentry.io/xxxxxx
SENTRY_ENVIRONMENT=production
SENTRY_RELEASE=bkurs-be@1.2.3
```

---

## 3. 환경별 활성화 매트릭스

| 서비스 | local | dev (staging) | prod |
|--------|-------|---------------|------|
| MySQL | Docker | RDS (소형) | RDS (Multi-AZ) |
| RabbitMQ | Docker | RabbitMQ Cloud / EC2 | RabbitMQ HA cluster |
| S3 | Mock | 실 S3 (dev 버킷) | 실 S3 |
| Email | Mock | SES sandbox | SES / SendGrid |
| AI Claude | Mock | 실 API (Haiku) | 실 API (Sonnet) |
| STT | Mock | 실 API (월 100건 한도) | 실 API |
| Payment | Mock | Mock 또는 PortOne test | PortOne live |
| Sentry | disabled | enabled (dev DSN) | enabled (prod DSN) |

> local 은 가급적 모두 Mock — 키 관리 부담 0. dev / prod 가 실 API.

## 4. 다음에 볼 것

- ▶ **키 받았으면**: [[env-deployment-guide]] §2 (local `.env.local`) / §3 (prod EKS Secret)
- ▶ **신규 SaaS 추가**: [[env-deployment-guide]] §6 (local → dev 승급 체크리스트)
- ▶ **gateway 추상화 패턴**: [[architecture-hybrid-external-services]]
- ▶ **계정 발급 진척 추적**: [[external-services-accounts]]
- ▶ **전체 진입점으로 돌아가기**: [[getting-started]]

## 5. 관련 노트

- [[bkurs-be]] — 프로젝트 허브 (PR 히스토리)
- [[../bkurs-fe/screen-logic-and-api-map]] — FE 측 어떤 화면이 어떤 SaaS 의존하는지

## 6. 참고

- 코드 SSoT (gateway 인터페이스):
  - `bkurs-be/src/main/java/com/bkurs/file/gateway/FileStorageService.java`
  - `bkurs-be/src/main/java/com/bkurs/ai/gateway/AiAssistant.java`
  - `bkurs-be/src/main/java/com/bkurs/notification/gateway/NotificationService.java`
  - `bkurs-be/src/main/java/com/bkurs/stt/gateway/SttService.java`
  - `bkurs-be/src/main/java/com/bkurs/payment/gateway/PaymentGateway.java`
- env 예시: `bkurs-be/.env.example`
- application.yml prefix: `bkurs.external.<service>.<provider>.*`
- Mock ↔ 실 impl swap 메커니즘: 각 module 의 `@ConditionalOnProperty` (provider env 값 기반)
