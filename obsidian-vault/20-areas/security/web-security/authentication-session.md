---
title: "인증·세션 취약점 (OWASP A07) — JWT/세션/크리덴셜"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, authentication, session, jwt, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[broken-access-control]]", "[[csrf]]"]
---

# 인증·세션 취약점 (OWASP A07) — JWT/세션/크리덴셜

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — JWT 함정/세션/크리덴셜 스터핑 |

**[[web-security|↑ web-security]]**

> 인증(누구인가)과 세션(인증 유지)의 실패. 인가(broken-access-control)와 구분.

OWASP A07(Identification and Authentication Failures)은 "이 요청 주체가 진짜
누구인지" 검증하는 단계와, 한 번 검증한 신원을 후속 요청 동안 유지하는 단계가
무너지는 결함을 묶는다. **인가(authorization, [[broken-access-control]])는 "이
주체가 이 리소스를 만질 권한이 있는가"** 를 다루는 별개 축이다 — 인증이 뚫리면
공격자가 임의 신원으로 들어오고, 인가가 뚫리면 정당한 신원이 권한 밖 리소스를
만진다. 둘을 섞어 생각하면 방어 지점을 잘못 잡는다.

## 1. JWT 함정

JWT(JSON Web Token)는 서버가 상태를 들고 있지 않아도 토큰 자체의 서명으로
신원을 검증하는 stateless 토큰이다. 검증 책임이 서버 코드에 있기 때문에 검증
로직의 작은 빈틈이 곧 신원 위조가 된다.

### 1.1 `alg: none` 공격

JWT 헤더에는 서명 알고리즘이 적혀 있고, 검증 라이브러리가 **헤더의 alg 값을
그대로 신뢰**해 서명 방식을 고르면 공격자가 `none`을 박아 서명 검증 자체를
건너뛰게 만들 수 있다.

```text
# 공격자가 위조한 토큰 (서명부가 비어 있음)
header  = {"alg":"none","typ":"JWT"}
payload = {"sub":"admin","role":"admin","exp":9999999999}
token   = base64url(header) + "." + base64url(payload) + "."
```

서버가 `none`을 허용하면 임의 payload(예: `role:admin`)가 그대로 통과한다.
방어는 **검증 시 알고리즘을 코드에서 화이트리스트로 고정**하는 것이다 — 헤더가
무엇을 주장하든 서버는 `HS256`만 받는다고 못 박는다.

### 1.2 HS256 / RS256 confusion

- **HS256**: 대칭. 같은 비밀키로 서명·검증.
- **RS256**: 비대칭. 개인키로 서명, 공개키로 검증.

서버가 "RS256 공개키"를 검증에 쓰는데, 검증 루틴이 헤더 alg를 신뢰해 **HS256
모드로 빠지면**, HS256은 검증키 = 서명키이므로 서버는 *공개적으로 알려진
공개키를 HS256 비밀키로 오인*해 검증한다. 공격자는 그 공개키로 HS256 서명을
직접 만들어 위조 토큰을 통과시킨다.

```text
# 서버 의도: RS256 공개키로 검증
# 공격자: alg를 HS256으로 바꾸고, 공개키(공개 정보)를 HMAC 키로 써서 서명
forged = HMAC-SHA256(key=server_rsa_public_key_PEM, data=header.payload)
```

방어는 동일하다 — **검증 알고리즘을 RS256으로 못 박고 헤더 alg를 신뢰하지
않는다**. 키 타입과 알고리즘을 분리해서 받지 않는다.

### 1.3 약한 비밀키

HS256 secret이 사전 단어/짧은 문자열/기본값이면 `hashcat` 류로 오프라인
brute-force가 가능하고, 키를 복구하면 임의 토큰을 위조한다. 실무에서 가장 흔한
운영 사고는 **prod에 dev 기본 시크릿이 그대로 나가는 것**이다.

우리 [[broken-access-control]] 계열 교훈과 같은 맥락으로, bkurs는
`application.yml`에서 로컬 기본값을 둔다.

```yaml
# application.yml (로컬 편의용 기본값)
jwt:
  secret: ${JWT_SECRET:bkurs-local-dev-secret-do-not-use-in-prod}
```

이 형태는 로컬 개발은 편하지만 **prod에서는 강한 `JWT_SECRET`을 env / secret
manager로 강제 주입하고, 누락 시 기동 실패(fail-fast)** 해야 한다. 기본값이
prod까지 따라가면 공격자가 소스에서 그대로 시크릿을 읽어 토큰을 위조한다.
"기본값이 있으니 안 죽고 뜬다"가 곧 사고다.

### 1.4 만료·폐기 부재

- **exp 없음/너무 김**: 토큰이 탈취되면 노출창이 무한대 또는 수일.
- **revocation 불가**: stateless 특성상 서버가 "이 토큰 무효" 라고 말할 곳이
  없다. 로그아웃·비번 변경·탈취 탐지 후에도 발급된 토큰이 만료까지 살아 있다.

이 둘 때문에 access token은 짧게(분 단위) + refresh token으로 재발급하고,
폐기가 필요한 토큰은 서버측 폐기 목록(deny list)으로 보완한다(아래 2.4 참조).

## 2. 세션 취약점

세션은 인증 결과를 후속 요청 동안 유지하는 서버측 상태(보통 쿠키의 세션 ID로
참조)다.

- **Session fixation**: 로그인 *전*에 공격자가 심어둔 세션 ID를 피해자가 그대로
  쓰게 만든 뒤, 피해자가 로그인하면 같은 세션 ID가 인증된 세션이 된다. **로그인
  성공 시 세션 ID를 회전(rotate)하지 않으면** 공격자가 알고 있던 ID로 인증
  세션에 올라탄다.
- **예측 가능 세션 ID**: 순차/타임스탬프 기반 ID는 추측 가능. CSPRNG로 충분한
  엔트로피(128bit 이상)를 줘야 한다.
- **쿠키 플래그 누락**: `HttpOnly`(JS 접근 차단 → XSS로 세션 탈취 방지),
  `Secure`(HTTPS에서만 전송), `SameSite`([[csrf]] 완화) 누락.
- **로그아웃해도 서버 세션 유지**: 클라이언트 쿠키만 지우고 서버측 세션을 무효화
  하지 않으면, 탈취된 세션 ID가 계속 유효하다.

## 3. 크리덴셜 공격

- **Credential stuffing**: 다른 서비스에서 유출된 (아이디, 비번) 목록을 그대로
  대입. 비번 재사용 사용자가 다수라 성공률이 의외로 높다. rate limit / MFA가
  없으면 자동화 봇에 무방비.
- **Brute-force**: 한 계정에 비번을 사전/무차별 대입. 계정 락·rate limit 부재 시
  성립.
- **평문/MD5 저장**: DB가 유출되면 평문은 즉시, MD5/SHA1은 GPU로 초당 수십억
  해시 속도라 사실상 즉시 복구된다.

## 4. 방어

### 4.1 (4구조) 왜 로그인 시 세션을 회전하고 토큰은 짧게 + refresh로 가는가

- **① 왜 필요**: 인증 *전*에 노출됐던 세션 ID/토큰을 인증 *후*에 무효화해야
  fixation과 탈취 노출창을 줄인다. 로그인 시 세션 ID를 새로 발급하면 공격자가
  미리 알던 ID는 죽는다. access token을 5~15분으로 짧게 두면 탈취돼도 곧 만료
  된다.
- **② 안 하면 문제**: ID 미회전 시 fixation으로 인증 세션 탈취. 토큰이 길면
  한 번 새는 순간 며칠간 임의 요청이 가능하다(폐기도 안 되니 손쓸 방법 없음).
- **③ 대안**: 매 요청 재인증(보안은 최상이나 UX 파탄), 또는 IP/디바이스 바인딩
  으로 탈취 토큰의 재사용 자체를 막는 방식. 대안은 상호 보완으로 같이 쓴다.
- **④ 트레이드오프**: 짧은 만료 + refresh는 재발급 트래픽과 refresh 회전 로직
  복잡도, 그리고 만료 직전 UX 끊김(슬라이딩 세션으로 완화) 비용을 치른다.
  예: 5분 access면 평균 2.5분마다 silent refresh가 돌아 토큰 엔드포인트 부하가
  올라가지만, 탈취 노출창을 수일 → 수분으로 압축하는 대가로는 싸다.

### 4.2 (4구조) 왜 비밀번호는 bcrypt/argon2(느린 해시 + salt)인가

- **① 왜 필요**: 비번은 복호화할 일이 없으므로 단방향 해시로 저장하되, **의도적
  으로 느린** 해시(bcrypt/argon2)를 써서 공격자가 DB 유출 후 오프라인 대량
  크래킹을 하기 어렵게 만든다. salt는 같은 비번이라도 해시를 다르게 만들어
  rainbow table과 동일 해시 군집화를 무력화한다.
- **② 안 하면 문제**: MD5/SHA256 같은 빠른 해시는 GPU로 초당 수십억 시도가
  가능해, DB가 유출되면 약한 비번은 분 단위로 복구된다. salt 없으면 한 번 깬
  결과가 모든 동일 비번 계정에 그대로 적용된다.
- **③ 대안**: pepper(앱 시크릿을 추가로 섞어 DB만 털려선 못 깨게), 또는 비번
  자체를 없애는 passkey/WebAuthn. argon2id는 GPU·ASIC 저항을 위해 메모리
  비용까지 요구하는 최신 권장값이다.
- **④ 트레이드오프**: 느린 해시는 **로그인마다 서버 CPU/지연**을 쓴다. bcrypt
  cost를 너무 높이면(예: cost 14~15) 로그인 응답이 수백 ms로 늘고 로그인
  폭주 시 CPU가 병목이 된다 — 보안 강도와 처리량 사이에서 cost를 튜닝한다.

### 4.3 운영 체크리스트

- **알고리즘 화이트리스트 고정**: 검증 시 alg를 코드에서 못 박아 `none`/HS-RS
  confusion 차단. 헤더 alg를 신뢰하지 않는다.
- **강한 시크릿**: env/secret manager 주입, prod 기본값 금지, 누락 시
  fail-fast. (위 1.3)
- **MFA**: credential stuffing/brute-force가 비번 하나로는 안 뚫리게.
- **로그인 rate limit + 계정 락**: IP·계정 단위 시도 제한, 임계치 초과 시 임시
  락 + 알림.
- **쿠키 플래그**: `Secure` + `HttpOnly` + `SameSite`(Lax/Strict). [[csrf]]와
  XSS 세션 탈취를 동시에 줄인다.
- **서버측 세션/토큰 폐기**: 로그아웃·비번 변경·탈취 탐지 시 서버 세션 무효화 +
  토큰 deny list로 stateless 폐기 한계 보완.

## 5. 관련

- [[broken-access-control]] — 인가 축(권한). 인증과 구분되는 별개 결함.
- [[csrf]] — `SameSite` 쿠키 플래그가 연결 지점.
- [[../cryptography/cryptography|cryptography]] — 해시(argon2/bcrypt)·키·서명
  의 기초.
- [[../security|↑ security]]
