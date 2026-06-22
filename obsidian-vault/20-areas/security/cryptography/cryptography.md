---
title: "암호학 응용"
kind: knowledge
project: security
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - security
  - cryptography
---

# 암호학 응용

> 정의: 기밀성·무결성·인증을 보장하기 위해 대칭/비대칭 암호, 해시, 디지털 서명, 키 관리를 실제 시스템(TLS, JWT, 저장 데이터 등)에 안전하게 적용하는 분야.

**[[../security|↑ security]]**

## 1. 무엇인가

응용 암호학은 암호 알고리즘 자체를 설계하는 학문이 아니라, 검증된 프리미티브를 "올바르게 조합해" 실제 보안 속성을 달성하는 공학이다. 직접 알고리즘을 만들지 않고 표준 라이브러리를 정확히 쓰는 것이 원칙이다.

암호가 제공하는 보안 속성:
- 기밀성(Confidentiality) — 권한 없는 자가 내용을 못 본다(암호화).
- 무결성(Integrity) — 변조를 탐지한다(해시, MAC).
- 인증(Authentication) — 출처를 확인한다(디지털 서명, HMAC).
- 부인 방지(Non-repudiation) — 서명자가 한 행위를 부인 못 한다.

핵심 구성요소:
- 대칭 암호(AES-GCM, ChaCha20-Poly1305) — 빠르고, 같은 키로 암복호.
- 비대칭 암호(RSA, ECC) — 공개/개인키 쌍, 키 교환·서명.
- 해시(SHA-256, SHA-3) — 단방향 지문.
- 비밀번호 해시(bcrypt, scrypt, Argon2) — 의도적으로 느리게.
- 키 관리(KMS, HSM) — 생성/저장/회전/폐기.

## 2. 왜 / 언제 필요한가

(1) 왜 필요한가 — 데이터는 전송 중(in transit)과 저장 중(at rest) 모두 노출 위험이 있다. 네트워크는 도청 가능하고 디스크/백업은 탈취될 수 있다. 암호는 "데이터를 가졌어도 키 없이는 쓸모없게" 만든다.

(2) 안 하면 무슨 문제가 생기나 — 구체적 사고 예시:
- 평문 비밀번호 저장: 유출 시 즉시 계정 탈취 + 비밀번호 재사용으로 다른 서비스까지 도미노. 2012년 LinkedIn은 솔트 없는 SHA-1로 저장해 대량 크래킹당했다.
- 약한/직접 구현 암호: MD5/SHA-1 충돌 공격, ECB 모드의 패턴 노출, 자체 구현(roll your own crypto)의 미묘한 결함.
- TLS 미적용/약한 설정: 중간자 공격(MITM)으로 세션 탈취. 2014년 Heartbleed(OpenSSL CVE-2014-0160)는 서버 메모리에서 개인키를 유출시켰다.
- 키 하드코딩: 소스에 박힌 키가 깃 히스토리에 남아 영구 노출.

(3) 대안/다른 접근과 비교 — 무결성만 필요하면 암호화 대신 HMAC으로 충분하다. 키 교환은 정적 키 공유보다 ECDH 같은 키 합의가 안전하다. RSA 대비 ECC는 같은 보안 강도에서 키가 짧아 모바일/IoT에 유리하다. 양자 위협에 대비한 PQC(Post-Quantum Cryptography) 전환도 진행 중이다.

(4) 트레이드오프 — 강한 암호는 성능 비용이 있고(Argon2는 의도적으로 느림), 키 관리 복잡도가 보안의 진짜 병목이다. 키를 잃으면 데이터를 영영 못 푼다(복구 불가). 알고리즘 민첩성(crypto-agility)을 확보하면 전환은 쉬워지나 시스템 복잡도가 올라간다.

## 3. 핵심 개념 / 어떻게

| 용도 | 권장 | 피해야 할 것 |
| --- | --- | --- |
| 대칭 암호화 | AES-256-GCM, ChaCha20-Poly1305 | DES, ECB 모드 |
| 비밀번호 저장 | Argon2id, bcrypt, scrypt | MD5/SHA로 직접 해시 |
| 무결성/인증 | HMAC-SHA256 | 단순 해시 비교 |
| 키 교환 | ECDH(X25519) | 정적 키 공유 |
| 서명 | Ed25519, RSA-PSS | RSA PKCS#1 v1.5(신규) |
| 전송 보안 | TLS 1.3 | TLS 1.0/1.1, SSLv3 |

핵심 개념:
- AEAD(Authenticated Encryption with Associated Data): 암호화 + 무결성을 동시에. GCM/Poly1305가 표준.
- 솔트(Salt) + 페퍼(Pepper): 같은 비밀번호도 다른 해시가 되게 해 레인보우 테이블 무력화.
- JWT: 서명(HS256=HMAC, RS256=RSA)으로 무결성 보장. 단, `alg:none` 취약점·과도한 페이로드 주의.
- 키 회전(rotation): 키 노출 시 폭발 반경을 시간으로 제한.
- 난수: 반드시 CSPRNG(`/dev/urandom`, `secrets` 모듈) 사용. `Math.random()`/`rand()` 금지.

원칙: 직접 구현하지 말고 libsodium, Web Crypto, OpenSSL 같은 검증된 라이브러리를 쓴다. NIST FIPS 140 검증 모듈이 규제 환경 기준이다.

## 4. 적용 맥락 / 경계

- cryptography: 암호 프리미티브의 "올바른 선택과 적용". 알고리즘·키·모드 수준.
- [[../network-security/network-security|network-security]]: 암호를 전송 경로에 적용(TLS/mTLS/VPN). crypto가 엔진이면 network는 그 엔진을 차에 장착.
- [[../application-security/application-security|application-security]]: A02 Cryptographic Failures처럼 코드에서 암호를 잘못 쓰는 결함.

경계 예시: AES 모드 선택은 crypto, HTTPS 강제는 network, JWT를 코드에서 검증 누락한 버그는 AppSec다. 시작점은 OWASP Cryptographic Storage Cheat Sheet 기준선 적용이다.

## 5. 관련

- [[../security|↑ security]]
- [[../network-security/network-security|네트워크 보안]]
- [[../application-security/application-security|앱 보안 / AppSec]]
- [[../../00-inbox/links/links|↗ 외부 reference]]
- 외부: NIST FIPS 140-3, NIST SP 800-57(키 관리), OWASP Cryptographic Storage Cheat Sheet, IETF RFC 8446(TLS 1.3), libsodium 문서
