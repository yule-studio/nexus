---
title: "bkurs 아키텍처 — Hybrid 외부 서비스 연동"
kind: architecture
project: bkurs-be
status: decided
created_at: 2026-05-20T22:00:00+09:00
agent: engineering-agent/tech-lead
tags: [architecture, hybrid, external-services, bkurs]
related:
  - "[[bkurs-be]]"
  - "[[../bkurs-fe/bkurs-fe]]"
  - "[[decisions/decision-record-bridge]]"
home_hub: bkurs-be
---

# bkurs 아키텍처 — Hybrid 외부 서비스 연동

> bkurs-be (Spring Boot 백엔드) 와 bkurs-fe (React 프론트) 가 외부 SaaS / 인프라
> 와 연결되는 표준 패턴. **BE-orchestrated 가 기본, file upload 만 presigned URL
> 예외**.

## 1. 핵심 원칙

- **Secret 은 server 전용**. AI API key / PG signing key / S3 access key 모두 BE
  환경변수로만. FE / Mobile 에는 절대 노출 X.
- **Business logic 일관성**. "PE 채점 → record 자동 생성 → 알림 발송" 같은 흐름이
  같은 트랜잭션 / saga 안에서. Client 가 중간에 끊기면 데이터 불일치.
- **Provider swap 가능**. Interface + Mock 패턴 → 실제 SaaS 1-class swap.
- **Audit 단일 지점**. 모든 외부 호출 로그가 BE 한 곳에서.

## 2. 전체 흐름도

```mermaid
flowchart TB
    subgraph "Client (bkurs-fe)"
        Web[Web Browser]
        Mobile[Mobile App]
    end

    subgraph "bkurs-be (Spring Boot)"
        API[REST API<br/>153 endpoint]
        Auth[Auth + JWT]
        Domain[Domain Services<br/>B2~B16]
        Bridge[Record Bridge<br/>B9]

        subgraph "External Service 추상화"
            FileSvc[FileStorageService]
            NotifySvc[NotificationService]
            AiSvc[AiAssistant]
            SttSvc[SttService]
            PdfSvc[PdfGenerator]
            PaySvc[PaymentGateway]
        end
    end

    subgraph "External SaaS / Infra"
        S3[(S3 / GCS)]
        Email[SendGrid / SES]
        SMS[Twilio / NCP]
        Claude[Claude API]
        Whisper[Whisper API]
        PortOne[포트원 / 토스]
        OAuth[카카오 / 구글 OAuth]
        Monitor[Sentry / Datadog]
    end

    subgraph "DB / Local Infra"
        DB[(MySQL)]
        Flyway[Flyway V1~V23]
    end

    Web -- "JWT Bearer" --> API
    Mobile -- "JWT Bearer" --> API
    API --> Auth
    API --> Domain
    Domain --> Bridge
    Domain --> DB

    Web -. "presigned URL<br/>요청" .-> API
    API -. "발급" .-> FileSvc
    FileSvc --> S3
    Web == "직접 업로드<br/>(presigned)" ==> S3
    Web -. "업로드 후 URL<br/>BE 에 저장" .-> API

    Domain --> NotifySvc
    NotifySvc --> Email
    NotifySvc --> SMS

    Domain --> AiSvc
    AiSvc --> Claude

    Domain --> SttSvc
    SttSvc --> Whisper

    Domain --> PdfSvc
    PdfSvc -. "byte[] 응답" .-> API
    API -. "PDF download" .-> Web

    Domain --> PaySvc
    PaySvc --> PortOne

    Auth -. "OAuth callback" .-> OAuth
    OAuth -. "code 반환" .-> Auth

    Domain -. "metrics / errors" .-> Monitor

    style FileSvc fill:#dae8fc,stroke:#6c8ebf
    style NotifySvc fill:#dae8fc,stroke:#6c8ebf
    style AiSvc fill:#dae8fc,stroke:#6c8ebf
    style SttSvc fill:#dae8fc,stroke:#6c8ebf
    style PdfSvc fill:#dae8fc,stroke:#6c8ebf
    style PaySvc fill:#dae8fc,stroke:#6c8ebf
    style S3 fill:#fff2cc,stroke:#d6b656
    style Email fill:#fff2cc,stroke:#d6b656
    style Claude fill:#fff2cc,stroke:#d6b656
```

## 3. File Upload 흐름 (presigned URL — 유일한 예외)

```mermaid
sequenceDiagram
    participant FE as bkurs-fe<br/>(Browser)
    participant BE as bkurs-be
    participant S3 as S3

    Note over FE,BE: 학생이 모의면접 영상 업로드 예시

    FE->>BE: POST /api/v1/files/upload-url<br/>{type: "interview-video", contentType: "video/mp4"}
    activate BE
    BE->>BE: 권한 검증 + uuid 생성
    BE->>S3: pre-sign PUT URL (만료 10분)
    S3-->>BE: presigned URL + key
    BE-->>FE: {uploadUrl, finalUrl, key}
    deactivate BE

    FE->>S3: PUT {uploadUrl}<br/>Body: video bytes
    activate S3
    Note right of S3: 직접 업로드<br/>BE bandwidth 절약
    S3-->>FE: 200 OK
    deactivate S3

    FE->>BE: POST /api/v1/students/me/interviews/{iid}/respond<br/>{content, audioUrl: null, videoUrl: finalUrl}
    activate BE
    BE->>BE: 검증 + InterviewResponse upsert
    BE-->>FE: 201
    deactivate BE
```

**왜 이 패턴?**
- 동영상/음성 = 큰 파일 (10MB-100MB+). BE proxy 통과 시 bandwidth 2배.
- presigned URL = 10분 만료, 1회용. Client 가 영구 credentials 보유 안 함.
- BE 가 발급 시점에 권한 검증 (학생 본인의 interview 만 업로드 가능).

## 4. AI 자동 평가 흐름 (BE-managed)

```mermaid
sequenceDiagram
    participant FE as bkurs-fe
    participant BE as bkurs-be
    participant Claude as Claude API
    participant Bridge as RecordBridge

    Note over FE,BE: 강사가 수행평가 채점 + AI 코멘트 보조

    FE->>BE: POST /performance-submissions/{sid}/grade<br/>{score: 85, feedback: "수정 필요"}
    activate BE
    BE->>BE: 권한 검증
    BE->>BE: submission.grade(85, "수정 필요")

    opt AI 코멘트 보조 (optional)
        BE->>Claude: 프롬프트 (학생 답안 + 점수 + 강사 코멘트)
        activate Claude
        Note right of Claude: secret key = BE 환경변수<br/>FE 노출 없음
        Claude-->>BE: "구체적 예시 3개 보강하면 90+"
        deactivate Claude
        BE->>BE: submission.aiComment = ...
    end

    BE->>Bridge: PerformanceGradedEvent
    activate Bridge
    Bridge->>BE: StudentRecordEntry upsert (SUBJECT)
    deactivate Bridge

    BE-->>FE: 200 OK<br/>{score, feedback, aiComment, recordCreated: true}
    deactivate BE
```

**왜 BE 직접?**
- Claude API key 는 server secret. Client 노출 시 비용 abuse 위험.
- AI 응답을 record 생성 트랜잭션에 포함 — 일관성.
- AI 호출 실패 시 retry / fallback 을 BE 단에서 표준화.
- 비즈니스 로직 ("AI 코멘트는 강사 채점 후 보조용") 이 server 정의.

## 5. STT (음성 전사) 흐름

```mermaid
sequenceDiagram
    participant FE as bkurs-fe
    participant BE as bkurs-be
    participant S3 as S3
    participant Whisper as Whisper API

    FE->>BE: 모의면접 영상 업로드 (위 #3 흐름)
    BE->>S3: presigned URL 발급
    FE->>S3: 영상 직접 업로드
    FE->>BE: POST /interviews/{iid}/respond<br/>{audioUrl: s3-url}
    activate BE
    BE->>BE: InterviewResponse 저장
    BE->>BE: SttJob async enqueue (audioUrl)
    BE-->>FE: 201 (즉시 응답)
    deactivate BE

    Note over BE,Whisper: async worker

    BE->>S3: audio bytes download
    BE->>Whisper: transcribe(audio bytes)
    activate Whisper
    Whisper-->>BE: "안녕하세요 저는 ..."
    deactivate Whisper
    BE->>BE: InterviewResponse.transcript = ...

    Note over FE,BE: 클라이언트 polling 또는 WebSocket

    FE->>BE: GET /interviews/{iid}/responses/{rid}
    BE-->>FE: {..., transcript: "안녕하세요..."}
```

**왜 BE 직접 호출?**
- Whisper API key 보호.
- 전사 결과 가 검색 (B10) 대상 → 저장이 비즈니스 로직.
- async 처리로 학생 응답 latency 영향 없음.

## 6. Notification (이메일/SMS) 흐름

```mermaid
sequenceDiagram
    participant Owner as 학원장
    participant BE as bkurs-be
    participant Teacher as 강사 (예비)
    participant Notify as SendGrid<br/>(또는 Twilio)

    Owner->>BE: POST /academies/me/teacher-invites<br/>{email, expiresIn: 7d}
    activate BE
    BE->>BE: TeacherInvite 생성 (32-byte token)
    BE->>Notify: send_email(<br/>  to: email,<br/>  subject: "강사 초대",<br/>  body: "토큰: {token} ..."<br/>)
    activate Notify
    Notify-->>Teacher: 이메일 도착
    deactivate Notify
    Notify-->>BE: messageId (track용)
    BE-->>Owner: 201 (token 은 응답 안 함)
    deactivate BE

    Teacher->>BE: POST /auth/teacher-signup<br/>{token, password}
    BE->>BE: token consume + Account.role=TEACHER
    BE-->>Teacher: JWT
```

**현재 vs 변경 후:**
- **현재 (Mock)**: token 이 응답 JSON 으로 노출 → 학원장이 직접 이메일/SMS forward.
- **변경 후**: BE 가 직접 이메일 발송 → 학원장은 email 만 입력하면 됨.

## 7. 인터페이스 + Mock 패턴

모든 외부 서비스는 인터페이스를 통해 추상화. 비즈니스 로직은 인터페이스만 참조.

```
com.bkurs.{domain}.gateway/
  ├── XService.java          // interface (BE-internal 계약)
  ├── MockXService.java      // 기본 구현 (개발/테스트 용)
  └── RealXService.java      // 실제 SaaS 구현 (별도 PR 에서 추가)
```

**Bean 선택**: `application-{profile}.yml` 의 `bkurs.external.{service}.impl=mock|real` property + `@ConditionalOnProperty`.

**예 — PaymentGateway (이미 구현됨, B4C)**:
```java
public interface PaymentGateway {
    ChargeResult charge(ChargeRequest req);
    RefundResult refund(String gatewayTxId);
    String name();
}

@Component
public class MockPaymentGateway implements PaymentGateway { ... }

// 후일:
// @Component
// @ConditionalOnProperty(name = "bkurs.external.payment.impl", havingValue = "portone")
// public class PortOnePaymentGateway implements PaymentGateway { ... }
```

## 8. 추상화 추가 계획 (B17~B21)

| PR | 인터페이스 | 통합 위치 | 후일 실제 구현 |
|---|---|---|---|
| B17 | `FileStorageService` | 신규 endpoint `POST /api/v1/files/upload-url` | S3 / GCS SDK |
| B18 | `NotificationService` | `TeacherInviteService.create()` 자동 호출 | SendGrid / SES / Twilio / NCP |
| B19 | `AiAssistant` | `PerformanceEvaluationService.gradeByTeacher()` 보조 호출 | Claude API |
| B20 | `SttService` | `ResponseService.submit()` async trigger | Whisper / NCP Clova STT |
| B21 | `PdfGenerator` | 신규 endpoint `GET /api/v1/students/me/record.pdf` | PDFBox / OpenPDF |

각 PR 은 **인터페이스 + Mock impl + 비즈니스 통합 + 회귀 테스트**.
실제 SaaS 구현은 사용자가 별도 PR 에서 추가.

## 9. Secret 관리 (modify 대상)

| 항목 | 현재 (commit 됨) | 변경 후 |
|---|---|---|
| DB password (local) | `application-local.yml` 평문 `'13950991*'` | `${DB_PASSWORD}` env var, `.env.example` 가이드 |
| DB password (prod) | 없음 | `application-prod.yml` + secret manager (AWS SSM / Vault) |
| JWT secret | `${JWT_SECRET:bkurs-local-dev-secret-...}` default 약함 | local default 유지, prod 필수 env |
| docker-compose | `MYSQL_PASSWORD: '13950991*'` | `MYSQL_PASSWORD: ${MYSQL_PASSWORD:-changeme}` |
| Claude API key | 없음 | `${CLAUDE_API_KEY}` (B19 추가 시) |
| Whisper API key | 없음 | `${WHISPER_API_KEY}` (B20) |
| SendGrid API key | 없음 | `${SENDGRID_API_KEY}` (B18) |
| S3 access key | 없음 | `${AWS_ACCESS_KEY_ID}` + `${AWS_SECRET_ACCESS_KEY}` (B17) |
| PortOne secret | 없음 | `${PORTONE_API_SECRET}` (B22+) |

**git history**: 평문 비번 commit 3개 (`e39c560`, `777eb2e`, `4138a18`) 는 push 됨.
- 옵션 1: `git filter-repo` 로 rewrite + force-push (파괴적, 협업자 영향).
- 옵션 2: 비번 rotate (DB user `mw` 비번 변경) — 권장.

## 10. company-infra 분리

별도 repo `company-infra` 로 이전:
- `docker-compose.yml` (현재 bkurs-be 루트)
- Kubernetes manifests (향후)
- Terraform / Pulumi IaC (향후)
- 환경 별 secret 관리 (Vault config, AWS SSM tree)

`bkurs-be` 에는 남김:
- `application.yml` / `application-local.yml` (env var 참조만)
- `.env.example`
- `Dockerfile` (앱 빌드)
- `.github/workflows/ci.yml` (앱 단위 CI)

## 관련

- [[bkurs-be|bkurs-be hub]] — 도메인 + abstraction 진행 로그
- [[../bkurs-fe/bkurs-fe|bkurs-fe hub]] — 짝 프론트엔드
- [[decisions/decision-record-bridge|B9 bridge 결정]] — same-TX 일관성 예시
- 후속: company-infra repo 신설 / 실 SaaS plug-in PR (B22+)
