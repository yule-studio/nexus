---
title: bkurs — env / 배포 가이드 (FE + BE, company-infra 연계)
kind: guide
status: active
created_at: 2026-05-21
tags: [bkurs-be, bkurs-fe, deployment, env, secrets, company-infra]
related:
  - "[[bkurs-be]]"
  - "[[external-services-guide]]"
  - "[[../bkurs-fe/bkurs-fe]]"
home_hub: "[[bkurs-be]]"
---

# bkurs — env / 배포 가이드

> 📍 **Read path**: [[getting-started]] (전체 진입점) → [[external-services-guide]]
> (SaaS 발급) → **본 노트** (env 어디에 설정) → [[bkurs-be]] (코드 SSoT).
>
> 본 노트는 **bkurs-be / bkurs-fe 레포 안에서 다루는 env 만** 다룬다. AWS Secret
> Manager / EKS Cluster 자체 / Terraform tfstate 는 `company-infra` 레포 SSoT.

## 1. 핵심 요약

이 노트는 **bkurs-fe / bkurs-be 레포에 직접 설정해야 하는 env / secret** 만 다룬다.
인프라 자체 (RDS / EKS / Route53 / ELB / Terraform / IAM 등) 는 별도 레포
`company-infra` 가 SSoT.

**경계선**:
- 본 노트 (bkurs-be / bkurs-fe 레포): application.yml / .env.example / Dockerfile / Github
  Actions workflow / docker-compose 안에서 다루는 모든 것.
- `company-infra` 레포: VPC / Subnet / RDS / EKS cluster / ALB / Route53 / IAM role /
  Secret Manager 자체 / Terraform tfstate.

> bkurs-be 가 환경변수로 받는 값들은 본 노트, **어디서 환경변수가 주입되는지** 는
> company-infra 의 Terraform module / EKS Secret 이 결정.

---

## 2. local 개발 — 0 부터 띄우기

### 2.1. bkurs-be local

```bash
cd bkurs-be
cp .env.example .env.local
vi .env.local  # DB_PASSWORD / JWT_SECRET 만 채우면 일단 OK (외부 SaaS 는 Mock)

# Docker 로 MySQL + RabbitMQ 띄우기
docker compose --env-file .env.local up -d mysql rabbitmq

# 환경변수 주입 후 부트
export $(cat .env.local | grep -v '^#' | xargs)
./gradlew bootRun
# → http://localhost:8080/api/v1/health
```

local 단계에서 **반드시 채워야** 하는 env:
- `DB_PASSWORD` (Docker MySQL 의 password 와 동일)
- `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD` (docker-compose 용)
- `JWT_SECRET` (32 bytes 이상, `openssl rand -base64 64`)

선택 (Mock 으로 시작):
- AWS / Anthropic / Clova / SendGrid / PortOne 모두 비워둬도 booted.
  외부 호출 시 InMemory / Mock impl 활성화.

### 2.2. bkurs-fe local

```bash
cd bkurs-fe
cp .env.example .env.local
vi .env.local  # VITE_API_BASE_URL 만 신경 쓰면 됨

npm install
npm run dev
# → http://localhost:5173
```

local 에서 **반드시 채워야** 하는 env:
- `VITE_API_BASE_URL=http://localhost:8080/api` (bkurs-be 가 8080 에 떠 있을 때)

선택:
- `VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` — concept 진단의 Supabase fallback
  경로 (대부분 swap 완료, 잔여 화면이 있으면 필요).

---

## 3. dev / prod 배포 — env 주입 경로

### 3.1. 큰 그림

```
company-infra (Terraform)
  ├─ AWS Secret Manager (DB_PASSWORD, JWT_SECRET, ANTHROPIC_API_KEY, …)
  ├─ EKS cluster
  │   └─ Deployment (bkurs-be:latest)
  │        └─ envFrom: secretRef → SecretManager 자동 동기화 (External Secrets Operator)
  │
  ├─ ALB (api.bkurs.com → bkurs-be Pod)
  └─ CloudFront + S3 (web.bkurs.com → bkurs-fe 정적 파일)
```

bkurs-be 의 Pod 가 부팅할 때 EKS Secret → 환경변수로 자동 주입. application.yml 의
`${ENV_NAME}` placeholder 가 해당 값을 읽음.

### 3.2. bkurs-be 가 받아야 하는 prod env (전수)

| env | 출처 | 비고 |
|-----|------|------|
| `SPRING_PROFILES_ACTIVE=prod` | EKS Deployment env (plain) | profile 분기 |
| `DB_HOST` | RDS endpoint (Terraform output) | EKS Secret |
| `DB_PORT=3306` | plain | |
| `DB_NAME=bkurs_prod` | plain | |
| `DB_USERNAME` | EKS Secret | |
| `DB_PASSWORD` | EKS Secret (RDS master pwd) | |
| `JWT_SECRET` | EKS Secret (`openssl rand -base64 64`) | rotation 주기 6개월 |
| `FRONTEND_URL=https://web.bkurs.com` | plain | CORS allowed-origin |
| `RABBITMQ_HOST` / `RABBITMQ_USERNAME` / `RABBITMQ_PASSWORD` | EKS Secret | |
| `AWS_REGION=ap-northeast-2` | plain | |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | EKS Secret | S3 / SES 공용. IAM Role 로 대체 가능 (EKS IRSA) |
| `S3_BUCKET_NAME=bkurs-prod-uploads` | plain | |
| `EMAIL_PROVIDER=ses` (또는 sendgrid) | plain | |
| `EMAIL_FROM=noreply@bkurs.com` | plain | |
| `SENDGRID_API_KEY` | EKS Secret | sendgrid 일 때 |
| `AI_PROVIDER=anthropic` | plain | |
| `ANTHROPIC_API_KEY` | EKS Secret | |
| `ANTHROPIC_MODEL=claude-sonnet-4-6` | plain | |
| `STT_PROVIDER=clova` | plain | |
| `NCP_CLOVA_INVOKE_URL` / `NCP_CLOVA_SECRET` | EKS Secret | |
| `PAYMENT_PROVIDER=portone` | plain | |
| `PORTONE_API_KEY` / `PORTONE_API_SECRET` / `PORTONE_CHANNEL_KEY` / `PORTONE_WEBHOOK_SECRET` | EKS Secret | |
| `SENTRY_DSN` | EKS Secret | |
| `SENTRY_ENVIRONMENT=production` | plain | |

> "plain" = ConfigMap 또는 Deployment env 에 직접 박아도 무방 (값 자체가 비밀이 아닌 것).
> "EKS Secret" = AWS Secret Manager → External Secrets Operator → K8s Secret → Pod env.

### 3.3. bkurs-fe 가 받아야 하는 prod env

build-time injection (Vite). `VITE_*` 만 client 번들에 포함되므로 **secret 절대 금지**.

| env | 출처 |
|-----|------|
| `VITE_API_BASE_URL=https://api.bkurs.com/api` | Github Actions workflow secret (또는 plain) |

> FE 에는 secret 이 들어가서는 안 됨 — JS 번들에 노출됨. API key 가 필요한 모든 호출은
> BE 가 proxy.

---

## 4. company-infra 연계 — 인계 항목

`company-infra` 팀에 넘겨야 하는 정보:

1. **secret 키 목록 + 값** (위 §3.2 의 "EKS Secret" 행 전부)
   - 형식 권장: 1 secret 당 1 entry (rotation 단위로 분리).
   - 권장 secret 이름 prefix: `bkurs-be/<env>/<key>` (예: `bkurs-be/prod/JWT_SECRET`).
2. **이미지 레포지토리**: ECR `123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/bkurs-be:1.2.3`
3. **health check endpoint**: `GET /api/v1/health` (200 OK + `{"status":"ok"}`)
4. **포트**: 8080 (http)
5. **resource request/limit** 권장: cpu 500m / mem 1Gi (request), cpu 2 / mem 2Gi (limit)
6. **Probe**:
   - liveness: `GET /actuator/health` (10s 후 시작, 30s 간격)
   - readiness: `GET /api/v1/health` (5s 후 시작, 10s 간격)
7. **CORS allowed-origin**: `FRONTEND_URL` env 로 단일 origin. multi 가 필요하면 BE 코드
   변경 후 배포.

> ⚠️ AWS access key 를 직접 EKS Secret 으로 주입하지 말고, **IAM Role for Service Accounts (IRSA)** 권장:
> EKS Service Account 에 IAM Role 연결 → AWS SDK 가 자동 credential 획득. key/secret env 불필요.

---

## 5. CI/CD — Github Actions

### 5.1. bkurs-be

`.github/workflows/dev-deploy.yml` 가 main → dev branch push 시 trigger:
1. `./gradlew test` (DB_PASSWORD repo secret 주입)
2. Docker build → ECR push (`bkurs-be:${{ github.sha }}`)
3. `kubectl set image` 으로 EKS Deployment 갱신

**Github Actions repo secret** (Settings → Secrets and variables → Actions):
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — ECR push 권한 IAM user
- `KUBECONFIG_BASE64` — `cat kubeconfig | base64`
- `DB_PASSWORD` — 테스트 단계용 (production 값 아님, 빌드 머신 mysql 컨테이너 비번)

### 5.2. bkurs-fe

`.github/workflows/web-deploy.yml`:
1. `npm ci && npm run build` (env 주입 `VITE_API_BASE_URL`)
2. S3 sync (`dist/` → `bkurs-prod-web` bucket)
3. CloudFront invalidation (`/index.html`)

**Github Actions repo secret**:
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — S3 sync 권한 IAM user
- `CLOUDFRONT_DISTRIBUTION_ID`
- `VITE_API_BASE_URL` — `https://api.bkurs.com/api`

---

## 6. local → dev 승급 체크리스트

신규 외부 SaaS 추가 시 dev 환경에서 확인할 것들:

- [ ] `.env.example` 에 새 env 추가 (주석 + provider 선택지 명시)
- [ ] application.yml 의 `bkurs.external.<service>.*` prefix 에 매핑
- [ ] `@ConfigurationProperties` record 신규 또는 확장
- [ ] Mock fallback impl 유지 (provider env 비어 있을 때 동작)
- [ ] dev EKS Secret 에 신 env 추가 (company-infra 팀 인계)
- [ ] 통합 테스트에서 env 없이도 통과하는지 (Mock 동작) 확인
- [ ] vault `external-services-guide.md` 에 §2 새 절 추가
- [ ] vault `env-deployment-guide.md` 의 §3.2 표에 행 추가

---

## 7. 운영 주기

- **secret rotation 주기**:
  - JWT_SECRET: 6개월 (token 만료 + refresh family revoke 동반)
  - DB_PASSWORD: 12개월 (RDS console 에서 변경 + Secret Manager + 재배포)
  - 외부 API key (Anthropic / SendGrid / PortOne 등): 6개월 또는 침해 의심 시 즉시
- **secret 누출 시**: company-infra 의 incident response runbook → 즉시 rotate + 영향
  분석 + git history 점검.

## 8. 다음에 볼 것

- ▶ **API 호출 흐름**: [[../bkurs-fe/screen-logic-and-api-map]] (화면 → endpoint)
- ▶ **새 SaaS 발급 필요**: [[external-services-guide]] §2 (서비스별 발급 단계)
- ▶ **CI hooks / probe 변경 작업**: `bkurs-be/Dockerfile` + `.github/workflows/dev-deploy.yml`
- ▶ **전체 진입점으로 돌아가기**: [[getting-started]]

## 9. 관련 노트

- [[bkurs-be]] — 프로젝트 허브 (PR 히스토리 + 결정 누적)
- [[external-services-guide]] — 각 SaaS 별 발급 / 설정 detail
- [[../bkurs-fe/bkurs-fe]] — FE 허브
- (company-infra 레포의 README — Terraform / EKS / Secret Manager 흐름)

## 10. 참고

- bkurs-be SSoT: `bkurs-be/.env.example`
- bkurs-fe SSoT: `bkurs-fe/.env.example`
- application.yml 환경 분기: `application-local.yml` / `application-prod.yml`
- Dockerfile: `bkurs-be/Dockerfile`
- CI: `.github/workflows/`
