---
title: "OAuth2 / OIDC 흔한 오용 — redirect_uri/state/PKCE"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, oauth2, oidc, pkce, authentication]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[authentication-session]]", "[[csrf]]"]
---

# OAuth2 / OIDC 흔한 오용 — redirect_uri/state/PKCE

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — redirect_uri/state/PKCE/토큰 검증·저장 |

**[[web-security|↑ web-security]]**

> OAuth2=위임 인가, OIDC=그 위의 인증 계층(id_token). 프로토콜은 맞는데 "검증 한 군데"를 빠뜨려 계정 탈취로 이어지는 패턴.

## 1. OAuth2 와 OIDC 의 구분

OAuth2 는 **인가(authorization)** 프로토콜이다. "이 앱이 사용자의 리소스(이메일·캘린더·결제 등)에 사용자를 대신해 접근해도 되는가"를 위임하는 것이고, 그 결과물은 리소스 서버에 제시할 `access_token` 이다. OAuth2 자체는 "이 사용자가 누구인가"를 보장하지 않는다 — access_token 은 단지 권한 증표이지 신원 증표가 아니다. OIDC(OpenID Connect)는 OAuth2 **위에 얹은 인증(authentication) 계층**으로, `id_token`(서명된 JWT)을 추가로 발급해 "로그인한 주체가 누구인지(sub)"를 RP(Relying Party)에게 검증 가능한 형태로 전달한다. 즉 access_token 으로 "무엇을 할 수 있는가"를, id_token 으로 "누구인가"를 표현한다. 이 둘을 혼동해 access_token 만 받고 신원으로 쓰거나(인증 우회), id_token 검증을 생략하는 것이 사고의 출발점이다. 이 노트는 모두 **Authorization Code Flow** 를 기준으로 한다(브라우저가 인가 코드를 받고, 백채널에서 토큰으로 교환).

## 2. 흔한 오용 (공격 → 방어)

### 2.1 redirect_uri 검증이 느슨함

인가 서버는 클라이언트가 보낸 `redirect_uri` 를 **사전 등록된 값과 정확 일치(exact match) allowlist** 로 검증해야 한다. prefix 매칭, 하위 경로 허용, 와일드카드 서브도메인 같은 느슨한 비교를 쓰면 공격자가 자신이 통제하는 목적지로 인가 코드/토큰을 흘려보낼 수 있다.

```
# 등록된 값:   https://app.example.com/callback
# 공격자 시도 (느슨한 검증 시 통과):
https://app.example.com.attacker.com/callback     # suffix 트릭
https://app.example.com/callback/../redirect?to=… # path traversal + open redirect
https://app.example.com@attacker.com/callback      # userinfo 트릭
```

피해자가 정상 인가에 동의하면 `code` 가 공격자 콜백으로 전달되고, 공격자가 이를 토큰으로 교환해 **계정 탈취**가 성립한다. 정상 도메인 안에 open redirect 가 하나라도 있으면 exact match 를 통과하면서도 코드를 외부로 튕겨낼 수 있다. 버그바운티 단골 패턴이다.

### 2.2 state 누락

`state` 는 인가 요청과 콜백을 **묶는 CSRF 방어**다. RP 가 인가 요청 시 예측 불가능한 랜덤 값을 생성해 세션에 저장하고, 콜백에서 돌아온 state 와 대조한다. 없거나 검증하지 않으면 공격자가 자신의 인가 코드를 피해자의 콜백에 주입(login CSRF / code injection)해 피해자를 **공격자 계정으로 로그인**시킬 수 있다 — 피해자가 입력한 데이터가 공격자 계정에 쌓인다. 상세 메커니즘은 [[csrf]] 참조.

```
GET /authorize?response_type=code&client_id=...
    &redirect_uri=https://app.example.com/callback
    &scope=openid%20email
    &state=8d3f1c9a2b...   # 세션 바인딩 랜덤, 콜백에서 동일성 검증
```

### 2.3 공개 클라이언트에 PKCE 미적용

SPA·모바일 앱은 **공개 클라이언트(public client)** 다. 바이너리/번들에 박힌 `client_secret` 은 디컴파일·DevTools 로 추출되므로 비밀이 아니다. 이때 **PKCE(Proof Key for Code Exchange)** 가 코드 가로채기(authorization code interception, 예: 모바일 커스텀 스킴 하이재킹)를 막는다. 클라이언트가 랜덤 `code_verifier` 를 만들고 그 해시인 `code_challenge` 를 인가 요청에 실어 보낸 뒤, 토큰 교환 시 원본 verifier 를 제시한다. 코드를 가로채도 verifier 를 모르면 교환이 불가능하다. 현재는 공개·기밀 클라이언트 **모두** PKCE 권장이다.

```
# 인가 요청
GET /authorize?response_type=code&client_id=spa-1
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
    &redirect_uri=https://app.example.com/callback&state=...

# 토큰 교환
POST /token
grant_type=authorization_code&code=AUTH_CODE
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

### 2.4 Implicit Flow 사용

Implicit Flow 는 access_token/id_token 을 **URL fragment(`#access_token=...`)로 브라우저에 직접** 반환한다. 토큰이 브라우저 히스토리·Referer 헤더·로그·확장 프로그램에 노출되고, 백채널 교환이 없어 출처 검증도 약하다. 현재는 **폐기 권고(deprecated)** 이며, **Auth Code Flow + PKCE** 로 대체한다(코드만 fragment 가 아닌 정상 파라미터로 받고, 토큰은 백채널에서 교환).

### 2.5 토큰 검증 누락

OIDC RP 는 받은 `id_token` 의 다음을 **전부** 검증해야 한다.

- **서명(signature)** — 발급자의 JWKS 공개키로 검증(키 회전 대응).
- **iss(issuer)** — 신뢰하는 발급자와 정확히 일치.
- **aud(audience)** — 내 client_id 가 대상에 포함. **token substitution** 차단의 핵심.
- **exp / iat** — 만료·발급 시각.
- **nonce** — 인가 요청에 심은 값과 동일(재생 방지).

이 중 하나라도 빠지면 **다른 앱용으로 발급된 토큰**이나 위조 토큰을 그대로 수용한다(token substitution / IdP mix-up). JWT 의 `alg` 혼동(`none`, RS256↔HS256 키 혼용) 같은 알고리즘 함정은 [[authentication-session]] 으로 위임한다.

### 2.6 토큰 저장 위치

브라우저에서 access/refresh 토큰을 어디 두느냐는 **트레이드오프**다. `localStorage` 는 XSS 한 번이면 통째로 탈취되고, 쿠키는 CSRF 표면이 생긴다. 현재 권장은 **BFF(Backend-for-Frontend)** 패턴 — 토큰은 서버(BFF)에 보관하고, 브라우저에는 `httpOnly`·`Secure`·`SameSite` 세션 쿠키만 노출한다. JS 가 토큰에 직접 닿지 않으므로 XSS 토큰 탈취 표면이 사라지고, 쿠키 기반 CSRF 는 SameSite + state/CSRF 토큰으로 막는다.

## 3. 방어 — 핵심 2개의 "왜"

### 3.1 왜 redirect_uri 는 정확 일치 allowlist 인가 (4구조)

- **왜 필요** — 인가 서버가 코드/토큰의 **목적지를 사전에 못 박아** 두어야, 공격자가 임의 목적지로 자격증명을 빼돌릴 수 없다. redirect_uri 는 "이 자격증명을 누구에게 줄 것인가"를 결정하는 마지막 관문이다.
- **안 하면 문제** — 한 글자(`.com.attacker.com`)나 도메인 내부 open redirect 한 개로 인가 코드가 공격자에게 가고, 곧장 **계정 탈취**로 이어진다.
- **대안** — 패턴/prefix 매칭(우회 표면 넓음), 동적 클라이언트 등록(검증·신뢰 부담 전가). 둘 다 공격 표면을 키운다.
- **트레이드오프** — dev/staging/prod·미리보기 배포마다 정확한 URL 을 전부 등록해야 하는 운영 수고가 든다. 그러나 이 수고가 곧 보안 경계이므로 와일드카드로 우회하지 않는다.

### 3.2 왜 공개 클라이언트에 PKCE + Auth Code Flow 인가 (4구조)

- **왜 필요** — 공개 클라이언트는 client_secret 을 숨길 수 없으므로, **secret 없이도** 코드 가로채기를 막을 수단이 필요하다. PKCE 는 verifier 를 아는 주체만 코드를 토큰으로 교환할 수 있게 해 이 빈틈을 메운다.
- **안 하면 문제** — 모바일 커스텀 스킴 하이재킹·중간자가 인가 코드를 가로채면 그대로 토큰 교환에 성공해 **코드 탈취=토큰 탈취**가 된다.
- **대안** — implicit flow(토큰 직접 노출, 폐기), client_secret 클라이언트 내장(추출되어 무의미).
- **트레이드오프** — verifier 생성·challenge 전달·토큰 교환의 구현 한 단계가 추가되지만, 대부분의 OAuth 라이브러리/SDK 가 자동 처리하므로 실비용은 낮다.

## 4. 체크리스트가 아니라 원칙

OAuth2/OIDC 보안은 항목 나열이 아니라 **"자격증명이 어디로 가고 누가 검증하는가"** 한 줄로 환원된다.

- redirect_uri 는 **정확 일치 allowlist** — 목적지를 못 박는다.
- 모든 인가 요청에 **state + nonce** — 요청·콜백·토큰을 세션에 묶는다.
- 모든 클라이언트에 **PKCE**, 항상 **Auth Code Flow**(implicit 금지).
- id_token 의 **서명·iss·aud·exp·nonce 5필드** 를 전부 검증한다.
- 토큰은 **짧은 수명 + 회전(refresh rotation)**, 브라우저는 **BFF + httpOnly 쿠키**.

목적지를 못 박고, 요청을 세션에 묶고, 받은 토큰을 끝까지 검증하면 위 오용은 대부분 동시에 닫힌다.

## 5. 관련 노트

- [[authentication-session]] — JWT 서명/alg 함정, 세션 토큰 관리
- [[csrf]] — state 가 막는 콜백 CSRF
- [[broken-access-control]] — scope/aud 미검증이 곧 권한 경계 붕괴
- [[api-security]] — access_token 을 받는 리소스 서버 측 검증
- [[../security|↑ security]]
