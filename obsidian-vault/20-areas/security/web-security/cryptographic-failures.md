---
title: "Cryptographic Failures (OWASP A02) — 전송/저장/키 관리"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, cryptography, tls, hashing, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[authentication-session]]", "[[security-misconfiguration]]"]
---

# Cryptographic Failures (OWASP A02) — 전송/저장/키 관리

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — 전송/저장/해시/키 관리 실패 |

**[[web-security|↑ web-security]]**

> OWASP A02(구 Sensitive Data Exposure). 암호화의 부재·약함·오용으로 민감 데이터가 노출되는 결함.

## 1. 개요

A02는 "노출된 결과"가 아니라 **원인**을 가리키도록 이름이 바뀐 항목이다. 구 명칭 Sensitive Data Exposure는 증상(데이터가 새어나감)을 말했지만, 현재 명칭 Cryptographic Failures는 그 증상을 만드는 근본 원인 — 암호화를 안 했거나(부재), 약한 알고리즘을 썼거나(약함), 올바른 알고리즘을 잘못 운용한(오용) — 세 가지를 직접 지목한다.

판단 기준은 단순하다. **"이 데이터가 평문으로 노출되면 피해가 큰가?"** 가 yes면 그 데이터의 전송 경로·저장소·키 관리 전부가 검토 대상이다. 민감 데이터 분류(비밀번호, 카드번호, 주민번호, 건강정보, 세션 토큰 등)가 선행되지 않으면 어디를 암호화해야 하는지조차 정의되지 않는다.

## 2. 전송 중(in transit) 실패

평문 또는 약한 채널로 데이터가 네트워크를 건너가는 경우다.

- **HTTP 평문** — 로그인 폼·API 호출이 HTTP면 같은 네트워크의 공격자가 패킷을 그대로 읽는다(중간자, MITM).
- **약한 TLS** — SSLv3, TLS 1.0/1.1은 알려진 취약점이 있다. **POODLE**은 SSLv3의 CBC 패딩을 악용해 암호문에서 평문 바이트를 추출하고, 약한 RC4·export cipher는 통계 공격에 노출된다.
- **다운그레이드 공격 / SSL stripping** — 공격자가 핸드셰이크를 가로채 약한 프로토콜로 협상을 끌어내리거나, HTTPS 링크를 HTTP로 바꿔치기해 평문으로 떨어뜨린다.
- **인증서 검증 생략** — 클라이언트(특히 모바일·서버 간 호출)가 인증서 체인·호스트명 검증을 끄면, 공격자의 자가서명 인증서를 그대로 신뢰해 MITM이 성립한다.

방어:

```nginx
# TLS 1.2+ 강제, 약한 프로토콜·cipher 제거, HSTS 적용
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_prefer_server_ciphers on;

# HSTS: 브라우저가 이후 접속을 HTTPS로 고정 → SSL stripping 차단
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

인증서 검증은 **절대 끄지 않는다.** 클라이언트 라이브러리의 `verify=False`, `InsecureSkipVerify: true` 같은 옵션은 운영 코드에서 금지한다.

## 3. 저장 시(at rest) 실패 — 비밀번호

비밀번호를 복원 가능한 형태로 저장하면, DB 한 번 유출로 전 사용자 계정이 위험해진다.

- **평문 저장** — 유출 즉시 전체 노출.
- **salt 없는 MD5/SHA-1** — 빠른 해시는 GPU로 초당 수십억 회 시도가 가능하고, salt가 없으면 레인보우 테이블(미리 계산된 해시 사전)로 일괄 역산된다.

실제 사례:

- **LinkedIn 2012** — salt 없는 SHA-1로 저장한 해시 약 650만 건이 유출됐다. salt가 없어 같은 비밀번호는 항상 같은 해시였고, 공개된 레인보우 테이블로 단시간에 대량 크랙됐다.
- **Adobe 2013** — 비밀번호를 **해시가 아니라** 단일 키 ECB 모드 대칭 암호로 "암호화"하고, 사용자가 입력한 **비밀번호 힌트를 평문으로 같이** 저장했다. ECB는 같은 평문 블록이 항상 같은 암호문 블록이 되므로, 같은 비밀번호를 쓴 계정끼리 암호문이 똑같이 보였다. 공격자는 동일 암호문 그룹을 묶고, 그 그룹에 딸린 평문 힌트들을 모아 원래 비밀번호를 추론했다 — 복호화 키 없이도 패턴만으로 깨졌다.

방어: 비밀번호는 **암호화하지 않고**, **느린(메모리·연산 비용이 큰) 해시** + 사용자별 **salt** 로 저장한다.

```python
# argon2id — 메모리-하드 KDF, 사용자별 salt 자동 생성/포함
from argon2 import PasswordHasher

ph = PasswordHasher()           # time_cost / memory_cost / parallelism 튜닝 가능
digest = ph.hash("user-password")   # salt가 digest 문자열에 인코딩됨
ph.verify(digest, "user-password")  # 검증
```

비밀번호 해시의 파라미터 튜닝·peppering 등 깊은 내용은 [[authentication-session]]에 위임한다.

## 4. 저장 시 실패 — 일반 민감 데이터

비밀번호 외의 민감 데이터(카드번호, 주민번호, 토큰)도 같은 오용 패턴에 노출된다.

- **평문 DB 저장** — 카드/주민번호를 그대로 컬럼에 넣음.
- **ECB 모드** — 블록 패턴이 보존돼 구조가 누출된다(3절 Adobe 참조).
- **IV/nonce 재사용** — CTR/GCM 계열에서 같은 키+같은 nonce를 두 번 쓰면 두 평문의 XOR이 드러나 기밀성이 무너진다.
- **인증 없는 암호화** — CBC만 쓰면 기밀성은 있어도 무결성이 없어, 공격자가 암호문을 조작(비트 플립·패딩 오라클)할 수 있다.

방어: **인증된 암호화(AEAD)** 인 AES-GCM을 쓰고, nonce는 매 암호화마다 유일하게 생성하며, **꼭 필요한 데이터만 저장(데이터 최소화)** 한다.

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

key = AESGCM.generate_key(bit_length=256)   # KMS에서 받아오는 것이 이상적
aesgcm = AESGCM(key)
nonce = os.urandom(12)                       # 매 암호화마다 새 nonce (재사용 금지)
ct = aesgcm.encrypt(nonce, b"4111-1111-1111-1111", b"card:user42")  # AAD로 맥락 바인딩
pt = aesgcm.decrypt(nonce, ct, b"card:user42")  # 변조 시 예외 → 무결성 검증
```

## 5. 키 관리 실패

알고리즘이 안전해도 키 관리가 무너지면 전부 무의미하다.

- **하드코딩 키** — 소스에 키 문자열을 박으면 바이너리·저장소 접근만으로 노출.
- **git 커밋된 키** — 한 번 커밋되면 히스토리에 영구 잔존(rebase로도 완전 제거 어려움). 즉시 **회전(폐기 후 재발급)** 필요.
- **키 회전 부재** — 키가 한 번 새면 과거·미래 데이터 전부 위험. 회전 정책이 있어야 노출 영향 범위가 한정된다.
- **약한 키 생성** — `random()`·시간 기반 시드 등 예측 가능한 난수로 키를 만들면 brute-force 없이도 키가 추측된다.

방어: 키는 **KMS / secret manager** 에 두고, 주기적으로 **회전** 하며, **CSPRNG**(암호학적 난수: `os.urandom`, `secrets`)로만 생성한다. CI/배포 환경에서는 환경변수·secret store 주입으로 코드와 키를 분리한다.

## 6. 자작 암호 금지 (Don't roll your own crypto)

암호 프리미티브·프로토콜을 직접 구현하면 타이밍 누출, 패딩 오라클, nonce 오용 같은 미묘한 결함이 거의 항상 들어간다. **검증된 표준(AES-GCM, argon2, TLS)과 잘 감사된 라이브러리**만 쓴다. "약간만 바꿔서" 직접 만든 변형도 금지다.

## 7. 핵심 설계 "왜" (4구조)

### 7.1 왜 비밀번호는 "암호화"가 아니라 "느린 해시 + salt"인가

- **① 왜 필요** — 서버는 비밀번호 **원문을 알 필요가 없다.** 로그인은 "입력값을 같은 방식으로 처리해 저장값과 일치하는가"만 확인하면 되므로, 복원 불가능한 단방향 해시면 충분하다. salt는 사용자마다 해시를 다르게 만들어 레인보우 테이블·동일 비밀번호 노출을 막는다.
- **② 안 하면 문제** — 암호화는 복호화가 가능해 키 유출 시 전원 원문이 풀린다. salt 없는 빠른 해시는 GPU + 레인보우 테이블로 대량 역산된다(LinkedIn 2012). Adobe는 ECB "암호화" + 평문 힌트로 키 없이도 비밀번호가 추론됐다 — 암호화 경로의 전형적 실패.
- **③ 대안** — (a) 대칭 암호화: 키 관리 부담 + 복호화 위험, (b) 빠른 해시(SHA-256)+salt: 레인보우는 막지만 무차별 크랙 속도는 못 늦춤, (c) **느린 해시(bcrypt/argon2/scrypt)+salt**: 시도당 비용을 의도적으로 키워 대량 크랙을 비경제적으로 만듦 → 권장.
- **④ 트레이드오프** — 느린 해시는 **로그인마다 CPU/메모리 비용과 수십~수백 ms 지연**을 유발한다. 파라미터를 너무 높이면 정상 로그인이 느려지고 DoS 표면이 되며, 너무 낮추면 크랙 저항이 약해진다 — 서버 용량에 맞춰 튜닝해야 한다.

### 7.2 왜 "인증된 암호화(AES-GCM)"이고 ECB는 금지인가

- **① 왜 필요** — 저장/전송 데이터는 기밀성(못 읽게)만이 아니라 **무결성(변조 탐지)** 도 필요하다. AES-GCM은 한 연산으로 암호화 + 인증 태그를 함께 만들어 둘을 동시에 보장한다.
- **② 안 하면 문제** — ECB는 블록 단위로 같은 평문→같은 암호문이라 **패턴이 그대로 보존**된다(같은 비밀번호·같은 이미지 윤곽이 암호문에 드러남, Adobe 사례). 인증이 없는 CBC 단독은 공격자가 암호문을 조작해도 탐지 못 해 패딩 오라클·비트 플립에 노출된다.
- **③ 대안** — (a) ECB: 절대 금지, (b) CBC + 별도 HMAC(encrypt-then-MAC): 동작은 하나 두 단계를 직접 조합해야 해 실수 여지, (c) **AES-GCM(AEAD)**: 기밀성+무결성을 한 프리미티브로 → 권장. ChaCha20-Poly1305도 동급 대안.
- **④ 트레이드오프** — GCM은 **nonce 유일성에 안전성이 전적으로 의존**한다. 같은 키로 nonce를 재사용하면 인증 키가 유출돼 위조가 가능해진다 — nonce 카운터/랜덤 생성 관리가 추가 부담이고, 대량 메시지에서는 키 회전까지 고려해야 한다.

## 8. 관련 노트

- [[authentication-session]] — 비밀번호 해시 파라미터·세션 토큰 보호 깊이
- [[security-misconfiguration]] — HSTS·보안 헤더·TLS 설정 운영
- [[../cryptography/cryptography|cryptography]] — 대칭/비대칭·해시·KDF 암호 이론
- [[../../../60-troubleshooting/network/tls-handshake-fail|tls-handshake-fail]] — TLS 핸드셰이크 실패 트러블슈팅
- [[../security|↑ security]]
