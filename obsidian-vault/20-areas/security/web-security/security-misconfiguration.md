---
title: "Security Misconfiguration (OWASP A05) — 보안 헤더/CORS/기본값"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, misconfiguration, http-headers, cors, csp, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[injection]]", "[[ssrf]]"]
---

# Security Misconfiguration (OWASP A05) — 보안 헤더/CORS/기본값

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — 기본값/헤더/CORS 오설정 |

**[[web-security|↑ web-security]]**

> OWASP A05. 코드가 아니라 "설정"이 만드는 구멍 — 기본값·과노출·헤더 누락.

## 1. 위험한 기본값 / 과노출

Security Misconfiguration 은 애플리케이션 로직의 버그가 아니라 **배포 환경의 설정값**이
안전하지 않은 채로 노출되는 결함이다. 코드 리뷰로는 안 잡히고, 인프라/런타임 설정에서 샌다.

- **기본 자격증명**: `admin/admin`, `root/root`, 라우터/DB/관리 콘솔의 출고 기본 비밀번호. 가장
  단순하지만 봇 스캐너가 가장 먼저 시도한다.
- **관리 콘솔 / 액추에이터 공개**: Spring Boot `/actuator/env`, `/actuator/heapdump`,
  `/admin`, phpMyAdmin, Kibana 등이 인증 없이 외부 노출. `/actuator/heapdump` 는 메모리 덤프 →
  평문 자격/토큰 유출로 직결된다.
- **디렉터리 리스팅**: 웹 서버가 인덱스 파일 없는 경로에서 파일 목록을 그대로 출력 → 백업 파일
  (`.env.bak`, `db.sql`), 소스 노출.
- **디버그 모드 prod 활성**: Django `DEBUG=True`, Flask debugger, `app.run(debug=True)` 가
  운영에 살아있으면 예외 화면에서 환경변수·설정·코드까지 노출되고, 일부는 원격 코드 실행(RCE)까지 간다.
- **상세 스택트레이스 / 에러 노출**: 500 응답에 내부 파일 경로, 프레임워크 버전, 실패한 SQL 쿼리가
  찍히면 공격자에게 정찰 정보를 무료로 제공한다. 예: `at com.app.UserRepo.find(UserRepo.java:88)`,
  `MySQL ... near 'WHERE id =' ` 같은 메시지는 [[injection]] 공격의 단서가 된다.
- **안 쓰는 포트/기능**: 기본 활성화된 샘플 앱, 사용 안 하는 API endpoint, 열린 관리 포트(예: 9200
  Elasticsearch)는 공격 표면만 키운다.
- **퍼블릭 S3 버킷**: `s3:GetObject` 를 `Principal: "*"` 로 열어둔 버킷은 역대 최대 규모 유출의
  단골 원인이다. ACL `public-read` + 버킷 정책 미검토 → 고객 PII·백업 전량 인터넷 공개.

```text
# 위험 — 누구나 객체 읽기 가능
{ "Effect": "Allow", "Principal": "*",
  "Action": "s3:GetObject", "Resource": "arn:aws:s3:::my-bucket/*" }
```

## 2. 보안 응답 헤더

브라우저에게 "이 사이트를 어떻게 다뤄야 하는지"를 지시하는 응답 헤더. 누락은 곧 클라이언트 측 방어 공백이다.

- **`Content-Security-Policy`** — 로드 가능한 리소스 출처를 화이트리스트로 제한해 [[injection]]
  계열 XSS 를 심층 방어. `script-src 'self'` 면 인라인/외부 임의 스크립트 실행을 차단한다.
- **`Strict-Transport-Security`(HSTS)** — 이후 모든 요청을 HTTPS 로 강제. 평문 HTTP 로 다운그레이드
  시키는 **SSL stripping** MITM 공격을 막는다.
- **`X-Content-Type-Options: nosniff`** — 브라우저가 `Content-Type` 을 무시하고 내용을 추측
  (MIME sniffing)하는 것을 차단. 이미지로 업로드된 파일이 HTML/JS 로 해석돼 실행되는 것을 막는다.
- **`X-Frame-Options: DENY`** / CSP **`frame-ancestors`** — 내 페이지를 다른 사이트가 `<iframe>`
  으로 감싸 투명 레이어로 클릭을 가로채는 **clickjacking** 을 방어.
- **`Referrer-Policy: no-referrer-when-downgrade`** — Referer 헤더로 내부 URL·토큰이 외부로
  새는 것을 제한.
- **`Permissions-Policy`** — 카메라/마이크/geolocation 등 강력한 브라우저 기능 사용을 출처별로 제한.

```nginx
# nginx — server 블록
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; frame-ancestors 'none'" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```

```js
// Express — helmet 미들웨어가 위 헤더 다수를 기본 주입
import helmet from "helmet";
app.use(helmet({
  contentSecurityPolicy: { directives: { defaultSrc: ["'self'"], scriptSrc: ["'self'"], frameAncestors: ["'none'"] } },
  hsts: { maxAge: 63072000, includeSubDomains: true, preload: true },
}));
```

## 3. CORS 오설정

CORS 는 **브라우저가** 교차 출처 응답을 자바스크립트에 노출할지 결정하는 메커니즘이다.
서버 인가(authorization)가 아니다 — `curl`/Postman/서버 간 요청에는 적용되지 않으므로, CORS 를
접근 제어로 착각하면 안 된다. 실제 인가는 [[csrf]] 토큰·세션·서버 권한 검사로 해야 한다.

위험한 조합:

```http
# 위험 — 모든 출처에 자격증명 포함 응답 허용 (브라우저가 거부하는 조합이지만,
#        구현이 Origin 을 그대로 reflect 하면 실질적으로 임의 출처 + 쿠키 노출)
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

- **Origin 무검증 reflect**: 요청의 `Origin` 헤더를 검증 없이 응답 ACAO 로 되돌리면(`evil.com` →
  `Access-Control-Allow-Origin: evil.com`) 사실상 모든 출처를 허용한 것이다. 피해자의 쿠키로 인증된
  민감 응답을 공격 사이트가 읽어간다.
- **`null` origin 허용**: `Origin: null` 은 sandbox iframe·로컬 파일에서 온다. 허용하면 우회 경로가 된다.

올바른 allowlist 기반:

```js
const ALLOW = new Set(["https://app.example.com", "https://admin.example.com"]);
app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin && ALLOW.has(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin); // 화이트리스트에 있을 때만 reflect
    res.setHeader("Vary", "Origin");                       // 캐시 오염 방지
    res.setHeader("Access-Control-Allow-Credentials", "true");
  }
  next();
});
```

preflight(OPTIONS) 가 막혀 디버깅이 필요할 땐
[[../../../60-troubleshooting/frontend/cors-preflight-blocked|cors-preflight-blocked]] 참조.

## 4. 방어 — 베이스라인 하드닝

### 4.1 왜 "하드닝된 기본(deny/최소 노출) 베이스라인 + 환경별 분리"인가

- **① 왜 필요** — 안전한 기본값(deny-by-default, 디버그 off, 액추에이터 비공개)이 깔려 있어야,
  나중에 누군가 설정을 깜빡 빠뜨려도 시스템이 닫힌 상태로 남는다. 보안이 "추가 작업"이 아니라 "기본 상태"가 된다.
- **② 안 하면** — 기본이 open 이면 새 환경/새 서비스마다 누군가 명시적으로 잠가야 하고, 한 번만
  빠뜨려도(예: 새 staging 에 `DEBUG=True` 그대로) 즉시 구멍. 실수 한 번의 비용이 유출이다.
- **③ 대안** — 매 배포마다 체크리스트로 수동 점검, 또는 외부 스캐너로 사후 탐지. 둘 다 사람/타이밍에
  의존해 누락이 생긴다.
- **④ 트레이드오프** — 초기 설정 비용↑, 개발 편의↓(로컬에서 헤더·CORS 때문에 막힘). → 환경별 프로파일
  (`dev`/`prod`)을 분리해 prod 만 엄격하게 두면 개발 마찰을 최소화한다.

예: prod 이미지에서는 `MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info` 로만 노출하고,
`/actuator/env`·`heapdump` 는 기본 비공개로 둔다.

### 4.2 왜 prod 에러를 일반화(사용자엔 generic, 로그엔 상세)해야 하는가

- **① 왜 필요** — 사용자에게는 "요청을 처리할 수 없습니다 (trace: 8f3a...)" 같은 generic 메시지만
  주고, 스택트레이스·쿼리·경로는 서버 로그에만 남겨 공격자의 정찰을 돕지 않는다.
- **② 안 하면** — 에러 화면이 프레임워크 버전·내부 경로·실패한 SQL 을 노출 → 공격자가 취약 버전을
  특정하고 [[injection]] payload 를 정밀하게 다듬는다.
- **③ 대안** — 에러를 통째로 숨기기. 그러면 운영 디버깅이 불가능해 장애 대응이 마비된다.
- **④ 트레이드오프** — 디버깅 난이도↑. → 구조화 로깅 + **추적 ID(trace id)** 로 상쇄: 사용자에게 준
  ID 로 로그에서 전체 컨텍스트를 즉시 찾는다.

### 4.3 보조 통제

- **IaC 로 설정 코드화 + 드리프트 점검**: 헤더·CORS·버킷 정책을 Terraform/Helm 에 선언하고, 콘솔
  수동 변경(drift)을 `terraform plan`·Config 규칙으로 탐지한다.
- **헤더 자동 주입**: helmet / nginx 공통 conf / API 게이트웨이 미들웨어로 모든 응답에 강제 — 라우트별
  누락을 원천 차단.
- **관측**: `securityheaders.com`, Mozilla Observatory 로 외부 관점 정기 스캔.
- **기본 자격 제거**: 프로비저닝 시 기본 계정 삭제/비밀번호 강제 변경을 부트스트랩에 포함.

## 관련

- [[injection]] — CSP 가 심층 방어하는 XSS, 에러 누출이 돕는 SQLi.
- [[ssrf]] — 노출된 액추에이터/메타데이터 endpoint 가 SSRF 와 결합되는 경로.
- [[csrf]] — CORS/SameSite 경계, "브라우저 보호 vs 서버 인가" 구분.
- [[../../../60-troubleshooting/frontend/cors-preflight-blocked|cors-preflight-blocked]]
- [[../security|↑ security]]
