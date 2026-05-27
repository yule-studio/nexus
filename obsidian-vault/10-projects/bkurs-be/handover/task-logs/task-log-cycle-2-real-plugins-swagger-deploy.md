---
title: "사이클 2 task-log — B22~B25 real plug-ins + restructure + Swagger + Docker"
kind: task-log
project: bkurs-be
status: completed
created_at: 2026-05-21T00:50:00+09:00
agent: engineering-agent/coding
tags: [task-log, cycle-2, real-plugins, swagger, docker, deploy]
related:
  - "[[../bkurs-be]]"
  - "[[../hybrid-external-services]]"
  - "[[../services-accounts]]"
home_hub: bkurs-be
---

# 사이클 2 — B22~B25 real plug-ins + restructure + Swagger + Docker

## 결과 (7 PR / 8 issue)

| PR | 이슈 | 내용 | 상태 |
|---|---|---|---|
| #78 | #77 | B22 PortOnePaymentGateway real | merged |
| #80 | #79 | B23 SendGridNotificationService real | closed/cascade |
| #82 | #81 | B24 ClaudeAiAssistant real | closed/cascade |
| #84 | #83 | B25 ClovaSttService real | closed/cascade |
| #86 | #85 | restructure (web→controller, dto split) | closed/cascade |
| #89 | #88 | Swagger UI + 8 group | closed/cascade |
| #91 | #90 | Dockerfile + GHCR workflow + dev profile | closed/cascade |
| - | #87 | envelope migration 153 endpoint | 후속 사이클 |

dev cascade commit: `218a89b`.

## 외부 SaaS plug-in (B22~B25)

모두 `@ConditionalOnProperty(impl=<provider>)` 활성화. 기본 Mock 유지.

| service | impl | provider class | properties prefix | 필요 env |
|---|---|---|---|---|
| Payment | `portone` | PortOnePaymentGateway | `bkurs.external.payment.portone` | `PORTONE_API_KEY` / `_SECRET` / `_CHANNEL_KEY` |
| Notification | `sendgrid` | SendGridNotificationService | `bkurs.external.notification.sendgrid` | `SENDGRID_API_KEY` / `EMAIL_FROM` |
| Ai | `anthropic` | ClaudeAiAssistant | `bkurs.external.ai.anthropic` | `ANTHROPIC_API_KEY` |
| Stt | `clova` | ClovaSttService | `bkurs.external.stt.clova` | `NCP_CLOVA_INVOKE_URL` / `_SECRET` |

모두 REST API (WebClient) 통합. 실 계정 없이 config 단위 검증.

## Restructure (#86)

- `web/` → `controller/` (21 도메인)
- `web/XHandler.java` → `handler/`
- `dto/CreateXRequest` → `dto/request/`
- `dto/XResponse`, `XView` → `dto/response/`
- 183 파일 변경 (대부분 git mv + import 갱신)
- common/web/ 그대로 (Filter / Filter 상수)
- common/handler/ 그대로

## Swagger (#89)

- springdoc-openapi-starter-webmvc-ui:2.8.13 (Spring 6.2 호환)
- `OpenApiConfig` — Info / Servers (local/dev/prod) / SecurityScheme bearer JWT
- 8 GroupedOpenApi:
  - 01-student 🎓 학생
  - 02-teacher 👨‍🏫 강사
  - 03-owner 🏢 학원장
  - 04-auth 🔐 인증
  - 05-files-pdf 📁 파일/PDF
  - 06-plan-billing 💳 플랜/결제 (admin)
  - 99-health ❤️ 헬스/액추에이터
  - 00-all 📚 전체 (개발자용)
- group id 영문 + displayName 한국어 (URL encoding 회피)
- application.yml — alpha sort / tryItOut / persistAuthorization / displayRequestDuration
- SecurityConfig — /swagger-ui/** + /v3/api-docs/** permitAll
- URL: `http://localhost:8080/swagger-ui.html`

## Docker + Deploy (#91)

### Dockerfile
- Multi-stage (Gradle build → JRE 21 runtime)
- 비root user (bkurs)
- 헬스체크 (actuator)
- JVM container memory 인식 (MaxRAMPercentage=75)

### .github/workflows/deploy-dev.yml
- dev push 시 GHCR image build + push 자동
- tags: dev / dev-<sha> / dev-latest
- 실 배포 step (ECS / EC2 / k8s) commented placeholder

### application-dev.yml
- env 필수 (no default for secret)
- SSL/HTTPS 강제
- stacktrace 숨김
- 외부 SaaS impl 환경변수 (`PAYMENT_IMPL=portone` 등)

### docker-compose.dev.yml
- 로컬 prod-like 시뮬레이션

## 후속 사용자 액션

1. **외부 SaaS 계정 가입** (Tier P0 — 외부 services accounts vault)
   - AWS (S3 + RDS + IAM)
   - PortOne (사업자 + 가맹 심사 ~5일)
   - SendGrid 또는 AWS SES
   - NCP SENS (KISA 발신번호 등록 ~5일)
   - Anthropic Console
   - NCP Clova Speech

2. **GitHub Actions Secrets 추가**
   - DB_PASSWORD / JWT_SECRET
   - GHCR 자동 (GITHUB_TOKEN)
   - 후속 배포 step 활성화 시: AWS_ACCESS_KEY / SSH key 등

3. **deploy-dev.yml deploy step 활성화**
   - 호스팅 옵션 선택 (ECS / EC2 / k8s)
   - 해당 step uncomment + secret 추가

4. **application-prod.yml 보강** (정식 launch 시)

## 후속 사이클 작업

- **#87 envelope migration** (153 endpoint 전체 ApiEnvelope) — 68 도메인 예외 base 변경 + 17 handler 제거 + 회귀 test 수정 = 100+ 파일.
- **B26 S3FileStorage** (AWS SDK)
- **B27 OpenPdfGenerator** (한글 폰트 bundle)
- **B28 OAuth** (카카오 / 구글)
- **B29 Sentry** monitoring
- **inroad-high 신규 화면** → bkurs-fe + bkurs-be API (task #178)
- **bkurs-fe Supabase → Spring 전환**

## 관련

- [[../bkurs-be]] — hub
- [[../hybrid-external-services]] — 아키텍처
- [[../services-accounts]] — 계정 + 비용
- [[task-log-phase-c-merge-cascade]] — 이전 dev cascade
