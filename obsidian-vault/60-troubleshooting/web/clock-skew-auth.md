---
title: "troubleshooting · 클록 스큐로 깨지는 인증 (JWT/TLS/TOTP)"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, clock-skew, ntp, jwt, tls, totp, auth]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[../../20-areas/security/web-security/authentication-session|authentication-session]]"]
---

# 클록 스큐로 깨지는 인증 (JWT/TLS/TOTP)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — 시계 드리프트/JWT leeway/NTP |

**[[web|↑ web]]**

방금 발급한 토큰이 "만료"라고 거부되고, 기간이 멀쩡한 인증서가 "아직 유효하지 않음"으로 떨어지고, 분명히 맞는 OTP 코드가 "잘못된 코드"로 튕긴다. 코드도 설정도 안 바꿨는데 어느 날 갑자기 인증이 깨진다면, 80%는 **양쪽 시계가 어긋난 것**이다. 보안 토큰·프로토콜은 거의 다 "발급자와 검증자의 시계가 같다"는 전제 위에 서 있기 때문이다.

## 1. 증상 (grep용 에러 모음)

같은 뿌리에서 나오는데 표면 에러는 계층마다 다르다. 이 문자열들이 한 시기에 산발적으로 보이면 클록 스큐를 의심하라.

- **JWT** — 발급 직후인데도:
  ```
  JWT expired at 2026-06-15T09:00:00Z. Current time: 2026-06-15T08:58:30Z
  io.jsonwebtoken.ExpiredJwtException: JWT expired
  JWTClaimsSet validation failed: token used before issued (iat)
  jwt.exceptions.ImmatureSignatureError: The token is not yet valid (nbf)
  ```
- **TLS** — 인증서 유효기간은 정상인데 클라이언트가 거부:
  ```
  x509: certificate has expired or is not yet valid
  SSL: CERTIFICATE_VERIFY_FAILED certificate is not yet valid
  curl: (60) SSL certificate problem: certificate is not yet valid
  ```
- **TOTP / 2FA** — 정확한 6자리 코드인데 계속 "잘못된 코드". 사용자는 "비밀번호 앱이 고장났다"고 신고하지만 실제로는 30초 윈도우가 어긋난 것.
- **Kerberos / Active Directory** — 가장 노골적인 에러:
  ```
  KRB_AP_ERR_SKEW: Clock skew too great
  ```
  AD 기본 허용 오차는 **5분**. 이걸 넘으면 도메인 로그인·SSO가 통째로 막힌다.
- **분산 환경** — 한 노드만 시간이 어긋나면 로드밸런서가 그 노드로 보낸 요청만 실패한다. "10번에 1번 로그인 실패" 같은 **간헐 장애**로 보여서 재현이 들쭉날쭉하고, 단일 노드에서 curl 테스트하면 멀쩡해 헛다리를 짚는다.

## 2. 원인 (root cause + 메커니즘)

이 토큰·프로토콜은 모두 **시각 비교**로 유효성을 판정한다. 양쪽 시계가 맞다는 가정이 깨지면 정상 토큰도 거부된다.

- **JWT** — 검증 시 세 클레임을 검증 서버의 *현재 시각*과 비교한다.
  - `exp`(만료): `now > exp` 면 만료. 검증 서버 시계가 빠르면 살아있는 토큰도 만료로 본다.
  - `nbf`(not before): `now < nbf` 면 아직 미활성.
  - `iat`(issued at): `iat > now` 면 "미래에 발급된 토큰" → 일부 라이브러리가 거부.

  발급 서버보다 검증 서버 시계가 몇 초~몇 분 빠르면 → "방금 발급됐는데 아직 유효하지 않음(nbf)" 또는 "이미 만료(exp)"라는 모순된 결과가 난다.
- **TLS** — 인증서의 `notBefore`/`notAfter`를 *클라이언트 로컬 시각*과 비교한다. 클라 시계가 과거로 점프하면 `not yet valid`, 미래로 점프하면 `expired`. 서버·CA는 멀쩡하고 **클라이언트 시계만** 문제인 경우가 많다.
- **TOTP** — 코드는 `HOTP(secret, floor(unixtime / 30))` 로 생성한다. 즉 30초 단위 스텝 카운터가 입력이다. 클라(앱)와 서버의 unixtime 이 30초 이상 어긋나면 **서로 다른 스텝**을 계산해 코드가 안 맞는다.

**뿌리는 NTP 동기화 실패**다. 대표 트리거:
- VM 일시정지 후 재개 / 라이브 마이그레이션 후 시계가 멈췄다 점프.
- 컨테이너가 호스트 시계를 그대로 쓰는데 호스트 NTP 가 죽음.
- 방화벽이 NTP(UDP 123)를 막아 동기화 자체가 안 됨.
- 가상화 환경의 시계 점프(steal time, TSC 불안정).

## 3. 해결

### 3.1 시계 동기화 강제 (NTP/chrony) — 근본 처방

```
# 현재 동기화 상태 (System clock synchronized: yes 확인)
timedatectl status

# chrony 추적 상태 — Last offset / RMS offset 가 핵심 지표
chronyc tracking
chronyc sources -v          # 상위 NTP 소스와 reach 확인

# 드리프트가 크면 즉시 보정 (slew 대신 step)
sudo chronyc makestep

# 노드 간 시각 직접 비교 (분산 디버깅)
date -u
```

- **컨테이너**는 자체 시계가 없고 호스트 시계를 공유한다 → 컨테이너 안에 ntpd 를 띄우는 게 아니라 **호스트 NTP** 를 고쳐야 한다.
- **클라우드**는 제공 NTP 를 써라(AWS: `169.254.169.123`, GCP: `metadata.google.internal`). 외부 pool.ntp.org 가 막혀도 동작하고 지연도 작다.

### 3.2 JWT 검증에 leeway (clock skew tolerance)

검증 시 허용 오차를 둬서 작은 드리프트를 흡수한다. 보통 **30~60초**.

```java
// jjwt
Jwts.parser()
    .verifyWith(key)
    .clockSkewSeconds(60)   // exp/nbf 비교 시 ±60초 허용
    .build()
    .parseSignedClaims(token);
```
```js
// jose (Node)
await jwtVerify(token, key, { clockTolerance: '60s' });
```

leeway 는 NTP 가 정상일 때 남는 수백 ms~수 초의 잔여 오차를 덮는 안전망이지, **NTP 부재의 대체가 아니다**(아래 4구조 참조).

### 3.3 TOTP 윈도우 ±1 스텝 허용

서버 검증 시 현재 스텝뿐 아니라 직전·직후 30초 스텝(`window = 1`)도 받아준다. 코드를 누른 직후 약간의 지연·드리프트를 흡수해 UX 를 살린다. window 를 2 이상으로 키우면 한 코드의 유효 시간이 길어져 재사용 공격 표면이 커지므로 **±1 이 균형점**.

### 3.4 인증서 만료 모니터링 + 자동 갱신

cert-manager(ACME)로 갱신을 자동화하고, 만료 D-30 알람을 건다. 갱신 절차는 [[../devops/tls-cert-renewal]] 참고.

## 4. 핵심 방어 — NTP 동기화 vs JWT leeway (4구조)

이 둘은 자주 혼동되는데, **역할이 완전히 다르다**. leeway 를 키우면 NTP 없이도 인증이 통과하니 같은 문제를 푸는 것처럼 보이지만, 보안적으로는 정반대 방향이다.

- **① 왜 NTP 동기화가 근본인가** — 모든 시각 기반 검증(JWT exp/nbf, TLS notAfter, TOTP 스텝, Kerberos)의 *공통 입력*이 "정확한 현재 시각"이다. NTP 로 시각을 맞추면 이 모든 계층이 동시에 정상화된다. leeway 는 JWT 하나에만 적용되고 TLS·TOTP·Kerberos 는 못 구한다.
- **② NTP 없이 leeway 만 키우면 무슨 문제가** — 드리프트가 몇 분까지 벌어지면 leeway 를 그만큼(예: 300초) 늘려야 통과한다. 그 순간 **만료된 토큰이 5분 더 살아남는다**. 탈취된 access token 의 유효창이 그만큼 길어지고, 짧은 만료로 피해를 줄이려던 설계가 무력화된다. TOTP·TLS 는 여전히 못 고치므로 장애도 일부만 사라진다.
- **③ 대안** — (a) NTP/chrony 로 시각을 맞추고 leeway 는 30~60초로 작게(권장). (b) leeway 없이 NTP 만(시각이 완벽하면 가능하나 부팅 직후 동기화 전 구간이 취약). (c) 토큰 만료를 길게 잡아 회피(보안 약화, 안티패턴).
- **④ 트레이드오프** — leeway 는 *가용성 ↔ 보안*의 다이얼이다. 크게 → 드리프트에 관대(장애 적음)하지만 만료·미활성 토큰을 오래 허용(보안 약화). 작게 → 보안은 타이트하지만 시계가 조금만 어긋나도 정상 토큰을 거부(가용성 악화). **정답은 다이얼을 만지는 게 아니라 NTP 로 드리프트 자체를 없애고 leeway 는 잔여 오차용으로 작게 유지**하는 것.

구체 예: access token 만료 5분 + leeway 300초로 NTP 부재를 덮은 서비스에서, 탈취된 토큰이 의도(5분)의 2배 가까이 유효해 로그아웃·rotation 으로도 즉시 무효화되지 않은 사례. NTP 를 붙이고 leeway 를 60초로 내리자 장애도 사라지고 유효창도 의도대로 복구됐다.

## 5. 재발 방지

- **전 노드 NTP 헬스 모니터링** — `chronyc tracking` 의 offset 을 메트릭으로 수집, 임계(예: 1초) 초과 시 알람. 분산 환경의 단일 노드 드리프트를 조기 포착.
- **인증서 만료 D-30 알람** + 자동 갱신(cert-manager/ACME).
- **이미지/VM 부팅 시 시간 동기 보장** — 부팅 직후 첫 인증 요청 전에 동기화가 끝나도록 startup 순서 고정(정지/마이그레이션 후 재개 포함).
- **TOTP 실패율 관측** — 특정 시각대에 실패율이 급증하면 서버 시계 드리프트 신호.

---

cross-link: [[web]] · [[../../20-areas/security/web-security/authentication-session|authentication-session]](JWT exp/세션) · [[../devops/tls-cert-renewal]](인증서 갱신) · [[../network/dns-resolve-fail]] · [[../troubleshooting|↑ 60-troubleshooting]]
