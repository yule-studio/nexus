---
title: "앱 보안 / AppSec"
kind: knowledge
project: security
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - security
  - application-security
---

# 앱 보안 / AppSec

> 정의: 애플리케이션의 소스 코드, 의존성, 런타임 동작에서 발생하는 취약점을 설계·개발·테스트·운영 전 단계에 걸쳐 식별하고 줄이는 보안 분야.

**[[../security|↑ security]]**

## 1. 무엇인가

AppSec(Application Security)은 웹/모바일/API 등 애플리케이션 계층에서 발생하는 취약점을 다룬다. 네트워크나 OS가 아니라 "코드가 데이터를 어떻게 처리하는가"가 핵심이다.

핵심 구성요소:
- 입력 검증 / 출력 인코딩 — 신뢰할 수 없는 입력이 코드/쿼리/마크업으로 해석되는 것을 차단.
- 인증(Authentication) / 인가(Authorization) — 누구인지 확인하고, 무엇을 할 수 있는지 통제.
- 세션·토큰 관리 — 세션 고정, 토큰 탈취 방지.
- 의존성·공급망 보안(SCA) — 서드파티 라이브러리의 알려진 취약점(CVE) 관리.
- 시크릿 관리 — 하드코딩된 키/비밀번호 제거.
- 보안 테스트 자동화 — SAST / DAST / IAST / SCA를 CI에 통합.

## 2. 왜 / 언제 필요한가

(1) 왜 필요한가 — 공격 표면의 대부분이 애플리케이션 계층에 있다. 방화벽이 80/443을 열어두는 순간, 그 포트로 들어온 요청의 안전은 전적으로 애플리케이션 코드가 책임진다. 네트워크 보안만으로는 SQL Injection 한 줄을 막을 수 없다.

(2) 안 하면 무슨 문제가 생기나 — 구체적 사고 예시:
- SQL Injection: 2017년 Equifax 사고는 Apache Struts 취약점(CVE-2017-5638)을 패치하지 않아 1.47억 명 개인정보가 유출됐다. 의존성 관리(SCA) 실패의 전형.
- XSS: 사용자 입력을 인코딩 없이 출력하면, 공격자가 심은 스크립트가 다른 사용자의 세션 쿠키를 탈취한다.
- Broken Access Control(IDOR): `/api/orders/1234`의 ID만 바꿔 타인의 주문을 조회. OWASP Top 10 2021에서 1위.
- 시크릿 노출: AWS 키가 깃 커밋에 들어가면 수 분 내 봇이 스캔해 암호화폐 채굴 인스턴스를 띄운다.

(3) 대안/다른 접근과 비교 — WAF(Web Application Firewall)로 외부에서 패턴을 차단하는 방법이 있으나, 이는 증상 완화일 뿐 근본 코드 결함은 남는다. 우회 가능한 미봉책이므로, 코드 수준 수정(secure coding)이 1차이고 WAF는 보완 계층이다.

(4) 트레이드오프 — SAST는 false positive가 많아 개발자 피로를 유발하고, 모든 입력 검증을 엄격히 하면 개발 속도가 떨어진다. 보안 게이트를 너무 빡빡하게 잡으면 빌드가 자주 깨져 개발팀이 우회하려 든다. "shift-left"(개발 초기 단계 보안)와 개발 생산성 사이의 균형이 핵심이다.

## 3. 핵심 개념 / 어떻게

OWASP Top 10 (2021)이 가장 널리 쓰이는 위협 분류다.

| 순위 | 카테고리 | 대표 방어 |
| --- | --- | --- |
| A01 | Broken Access Control | 서버측 인가 검증, deny-by-default |
| A02 | Cryptographic Failures | TLS 강제, 약한 해시(MD5/SHA1) 제거 |
| A03 | Injection (SQLi/XSS 등) | Prepared Statement, 출력 인코딩 |
| A05 | Security Misconfiguration | 기본 설정 변경, 불필요한 기능 비활성 |
| A06 | Vulnerable Components | SCA로 CVE 추적, 정기 업데이트 |
| A07 | Identification/Auth Failures | MFA, 안전한 세션 관리 |

보안 테스트 도구 유형:
- SAST(정적 분석): 코드 자체를 스캔. 예) Semgrep, SonarQube, CodeQL. 배포 전 조기 발견.
- DAST(동적 분석): 실행 중인 앱을 외부에서 공격. 예) OWASP ZAP. 런타임 결함 발견.
- SCA(의존성 분석): 라이브러리 CVE 스캔. 예) Dependabot, Trivy, OWASP Dependency-Check.
- IAST: 런타임 계측으로 SAST/DAST 결합.

표준 방어 패턴: Parameterized Query(Injection 방어), Content-Security-Policy 헤더(XSS 완화), CSRF 토큰, 입력은 화이트리스트 검증, 출력은 맥락별 인코딩.

## 4. 적용 맥락 / 경계

- application-security: 코드와 앱 런타임의 취약점. "내가 짠 로직이 안전한가."
- [[../network-security/network-security|network-security]]: 트래픽·전송 경로의 안전. TLS/방화벽/세그멘테이션.
- [[../cloud-security/cloud-security]]: 인프라 설정·IAM·클라우드 리소스. "S3 버킷이 public인가."

경계 예시: SQLi는 AppSec, 데이터센터 간 트래픽 암호화는 network-security, IAM 역할 과대 권한은 cloud-security다. 시작점은 보통 OWASP Top 10 자가 점검과 CI에 SCA/SAST를 붙이는 것이다.

## 5. 관련

- [[../security|↑ security]]
- [[../cryptography/cryptography|암호학 응용]]
- [[../pentesting/pentesting|침투 테스트]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
- 외부: OWASP Top 10, OWASP ASVS(Application Security Verification Standard), OWASP Cheat Sheet Series, CWE Top 25, MITRE ATT&CK
