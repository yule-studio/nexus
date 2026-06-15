---
title: "웹 보안 — OWASP Top 10 허브"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-05-13T04:00:00+09:00
tags:
  - area
  - index
  - security
  - web-security
  - owasp
---

# 웹 보안 — OWASP Top 10 허브

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-15 | security-review | OWASP Top 10 2021 허브로 재작성 + 깊이 노트 6종 결선 |

**[[../security|↑ security]]**

> 프레임워크 무관한 "웹이면 알아야 하는" 보안 지식. 특정 스택의 CVE/설정은
> `60-troubleshooting/<stack>/` 에, 여기엔 **공격 계열의 원리·방어**를 둔다.

## 이 허브를 만든 이유

vault 의 기존 보안 노트는 Next.js CVE, Spring Security 401 처럼 **프레임워크
기술 이슈** 중심이었다. 하지만 "헤더 하나로 관리자 페이지 접근"(인가 우회),
"서버가 내부망에 요청을 대신 보냄"(SSRF) 같은 문제는 **프레임워크가 아니라
웹 자체의 신뢰 경계** 문제다. 이 계층을 OWASP Top 10 골격으로 정리한다.

## OWASP Top 10 (2021) 카탈로그

| # | 항목 | 한 줄 | 깊이 노트 |
| --- | --- | --- | --- |
| A01 | Broken Access Control | 인증된 사용자가 권한 밖 자원/기능 접근 (IDOR·헤더 신뢰) | [[broken-access-control]] |
| A02 | Cryptographic Failures | 평문 전송/저장, 약한 해시, 키 관리 실패 | [[cryptographic-failures]] |
| A03 | Injection | 입력이 인터프리터에 명령으로 해석 (SQLi/XSS/명령) | [[injection]] |
| A04 | Insecure Design | 설계 결함 vs 구현 버그 / 위협 모델링 / 비즈니스 로직 | [[insecure-design]] |
| A05 | Security Misconfiguration | 기본값·과노출·보안 헤더 누락·CORS 오설정 | [[security-misconfiguration]] |
| A06 | Vulnerable & Outdated Components | 취약 의존성 방치 (npm audit / Dependabot) | [[vulnerable-components]] |
| A07 | Identification & Authentication Failures | JWT 함정·세션·크리덴셜 스터핑 | [[authentication-session]] |
| A08 | Software & Data Integrity Failures | 무결성 미검증 업데이트/역직렬화/CI 공급망 | [[insecure-deserialization]] |
| A09 | Logging & Monitoring Failures | 탐지·대응 불가 (감사 로그 부재) | [[security-logging-monitoring]] |
| A10 | Server-Side Request Forgery (SSRF) | 서버가 공격자 URL로 내부 자원에 도달 | [[ssrf]] |

> OWASP Top 10 에는 별도 번호가 없지만 웹 필수 지식인 **CSRF**([[csrf]])와
> **Clickjacking**([[clickjacking]])도 함께 둔다. CSRF 는 2017 판 A08 →
> 2021 판에선 SameSite 보급으로 강등됐고, Clickjacking 은 A05(보안 헤더)
> 방어와 맞물린다.

## 깊이 노트 (이 영역 직속)

| 노트 | 핵심 | 대표 실제 사례 |
| --- | --- | --- |
| [[broken-access-control]] | IDOR / 함수레벨 인가 / 클라이언트 값 신뢰 | Next.js 미들웨어 헤더 우회 (CVE-2025-29927) |
| [[ssrf]] | egress allowlist / 메타데이터 차단 / IMDSv2 | Capital One 2019 (1억+ 건 유출) |
| [[authentication-session]] | JWT alg/혼동/약한 시크릿 / 세션 회전 / 크리덴셜 스터핑 | dev 기본 시크릿 prod 유출 |
| [[injection]] | 파라미터화 / 출력 인코딩 + CSP | 고전 SQLi 대량 유출 |
| [[security-misconfiguration]] | 보안 헤더 / CORS allowlist / 기본값 하드닝 | 퍼블릭 S3 버킷 유출 |
| [[cryptographic-failures]] | TLS/HSTS / 느린 해시+salt / AES-GCM / 키 관리 | LinkedIn 2012, Adobe 2013 |
| [[vulnerable-components]] | SBOM / 의존성 스캔 / 패치 SLA | Log4Shell, Equifax 2017 |
| [[insecure-design]] | 설계 결함 vs 구현 버그 / 위협 모델링 / 비즈니스 로직 | — |
| [[insecure-deserialization]] | 네이티브 역직렬화 금지 / 데이터 포맷+스키마 / 산출물 서명 | ysoserial, SolarWinds 2020 |
| [[security-logging-monitoring]] | dwell time / SIEM·알림 / 로그에 PII 금지 | Target 2013, Marriott |
| [[csrf]] | SameSite 쿠키 / CSRF 토큰 / Bearer 트레이드오프 | — |
| [[clickjacking]] | frame-ancestors / X-Frame-Options | likejacking |
| [[api-security]] | BOLA / mass assignment / 과다 노출 / 레이트 리밋 | Optus 2022, Parler 2021 |
| [[oauth2-oidc-pitfalls]] | exact redirect_uri / state+nonce / PKCE / BFF | OAuth 계정 탈취 버그바운티 |

> **API & 인증 프로토콜**: [[api-security]]는 OWASP **API** Security Top 10
> (웹 Top 10 과 별도), [[oauth2-oidc-pitfalls]]는 위임 인가/OIDC 의 흔한 오용.
> 둘 다 A01(인가)·A07(인증)의 API/프로토콜 심화로 읽으면 좋다.

## 관통하는 원리 (모든 노트의 공통 뼈대)

1. **신뢰 경계 밖 입력을 권한/제어 신호로 쓰지 마라** — 헤더·쿼리·바디·정규식
   입력 전부 공격자 통제. (A01 헤더 신뢰, A03 주입, SSRF URL, ReDoS 입력)
2. **인가의 최종 판정은 데이터에 가장 가까운 곳에서, 검증된 신원으로** — 미들웨어/
   FE 가드/네트워크 위치는 우회 가능한 보조 계층.
3. **deny-by-default** — 명시 허용만 통과. (인가·CORS·egress·캐시 키 전부 동일 원리)
4. **심층 방어** — 단일 방어선(미들웨어·escaping·WAF)은 한 번 뚫리면 끝. 레이어를 겹친다.

## 실제 운영 장애로 이어진 사례 (cross-link)

웹 보안 결함은 종종 "보안"이 아니라 "가용성 장애"로 먼저 터진다.

- [[../../../60-troubleshooting/web/redos-regex-dos\|ReDoS]] — 정규식 한 줄이 Cloudflare 글로벌 장애 (2019)
- [[../../../60-troubleshooting/web/cdn-cache-poisoning\|CDN 캐시 포이즈닝]] — unkeyed 입력 반사로 전체 사용자 오염
- [[../../../60-troubleshooting/web/cache-stampede\|캐시 스탬피드]] / [[../../../60-troubleshooting/web/retry-storm-metastable\|재시도 폭풍]] — 부하 증폭 연쇄

## 관련

- [[../security|↑ security]]
- [[../../../60-troubleshooting/web/web|↗ 60-troubleshooting/web — 실전 장애 카탈로그]]
- [[../../computer-science/security-theory/security-theory|↗ CS 보안 이론]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
