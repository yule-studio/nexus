---
title: "CSRF — Cross-Site Request Forgery"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, csrf, cookie, samesite]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[authentication-session]]", "[[security-misconfiguration]]"]
---

# CSRF — Cross-Site Request Forgery

| 버전 | 날짜 | 작성 | 변경 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — 원리/SameSite/토큰 |

**[[web-security|↑ web-security]]**

> 브라우저가 자동으로 붙이는 쿠키(ambient authority)를 악용해, 사용자가 모르게 상태 변경 요청을 위조.

## 1. 개요

CSRF(Cross-Site Request Forgery)는 **이미 로그인된 사용자의 브라우저를 속여,
사용자가 의도하지 않은 상태 변경 요청을 신뢰받는 사이트로 대신 보내게** 만드는
공격이다. 핵심은 단 하나 — 브라우저는 같은 출처(origin)의 쿠키를 요청에 **자동으로**
첨부한다는 사실이다. 이 "암묵적 권한(ambient authority)"을 공격자가 빌려 쓴다.

전형적 시나리오:

1. 피해자가 `bank.example.com`에 로그인 → 세션 쿠키가 브라우저에 저장됨.
2. 같은 브라우저로 공격자 페이지 `evil.example.com`를 방문.
3. 그 페이지가 `bank.example.com`으로 송금 `POST` 요청을 자동 발사.
4. 브라우저는 목적지가 `bank.example.com`이므로 **세션 쿠키를 자동 첨부**.
5. 서버는 "유효한 세션이 보낸 정상 요청"으로 보고 송금을 실행.

인증은 멀쩡히 통과한다. 위조된 것은 인증이 아니라 **"사용자의 의도"**다.

## 2. 원리 — 인증이 아니라 "의도"가 위조된다

```
피해자 브라우저 (bank 세션 쿠키 보유)
   │  ① evil.example.com 방문
   ▼
[evil 페이지] ──② POST bank.example.com/transfer 자동 발사──▶ bank 서버
                     (브라우저가 bank 쿠키 자동 첨부)            │
                                                                 ▼
                                        ③ 정상 세션으로 인식 → 송금 실행
```

CSRF가 성립하는 전제는 다음 세 가지가 모두 참일 때다.

- **쿠키 기반 세션** — 요청에 자격 증명이 자동으로 실린다(브라우저가 알아서 붙임).
- **예측 가능한 요청** — 공격자가 요청 형식(URL/파라미터)을 미리 안다.
- **부작용 있는 요청** — 그 요청이 서버 상태를 바꾼다(송금/비번 변경/권한 부여 등).

XSS와의 구분이 중요하다.

- **XSS** — 공격자가 **피해자 사이트 안에서 코드(JS)를 실행**시킨다. 토큰을
  훔치고 DOM을 읽는 등 사실상 무엇이든 한다. 출처 안쪽에서의 침투다.
- **CSRF** — 공격자는 **코드를 실행하지 못한다.** 응답 본문을 읽을 수도 없다.
  단지 **요청을 위조해 발사**할 뿐이다. 출처 바깥에서 쿠키만 빌린다.

그래서 XSS가 있으면 CSRF 방어(토큰 포함)는 대부분 무력화된다 — 토큰조차 읽어
가기 때문이다. CSRF 방어는 XSS 방어를 전제로 한다.

## 3. 공격 예시

상태 변경을 GET으로 처리하는 서버는 `<img>` 하나로도 뚫린다.

```html
<!-- GET 부작용: 페이지를 여는 순간 이미지 로드로 송금 요청 발사 -->
<img src="https://bank.example.com/transfer?to=attacker&amount=1000000"
     width="1" height="1" style="display:none">
```

POST 요청도 숨은 자동제출 폼으로 위조된다.

```html
<!-- evil.example.com 안의 숨은 폼 — onload로 자동 제출 -->
<body onload="document.forms[0].submit()">
  <form action="https://bank.example.com/transfer" method="POST">
    <input type="hidden" name="to" value="attacker">
    <input type="hidden" name="amount" value="1000000">
  </form>
</body>
```

`fetch`로도 가능하지만 자격 증명을 포함시켜야 한다.

```js
// credentials:'include' 가 있어야 크로스사이트로 쿠키가 실린다.
// 단, CORS 때문에 응답은 못 읽는다 — 그래도 "발사"는 된다(부작용 발생).
fetch("https://bank.example.com/transfer", {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: "to=attacker&amount=1000000",
});
```

## 4. 방어

### 4.1 SameSite 쿠키 (Lax/Strict) — 1차 방어

- **① 왜 필요**: CSRF의 뿌리는 "크로스사이트 요청에도 쿠키가 자동으로 붙는 것"이다.
  `SameSite=Lax`/`Strict`는 다른 출처에서 시작된 요청에 쿠키를 **아예 안 붙여** 위조
  자체를 차단한다. 모던 브라우저는 미지정 쿠키를 기본 `Lax`로 취급한다.
- **② 안 하면**: `SameSite=None`(또는 미지정 + 구형 브라우저)이면 evil 페이지의
  자동 폼/이미지에 세션 쿠키가 그대로 실려 2~3절 공격이 성립한다.
- **③ 대안**: CSRF 토큰(4.2), Origin/Referer 검증(4.3). SameSite는 가장 적은 코드로
  가장 넓게 막는 1차 방어이지만, 단독으로는 부족하다.
- **④ 트레이드오프**: `Strict`는 외부 링크로 진입한 첫 요청에도 쿠키를 안 붙여,
  "이메일 링크 클릭 → 로그인된 화면 기대"가 깨진다. `Lax`는 top-level GET 내비게이션은
  허용해 이를 완화하지만, **서드파티 임베드/결제 리다이렉트/iframe 위젯** 같은
  교차 출처 흐름이 깨진다. 구형 브라우저는 SameSite 미지원 → 토큰 병행이 필요하다.

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/
```

### 4.2 CSRF 토큰 (synchronizer / double-submit)

- **① 왜 필요**: 공격자는 evil 페이지에서 요청을 위조할 수 있어도, **세션마다 유일한
  비밀 토큰 값은 알 수 없다.** 서버가 모든 상태 변경 요청에 그 토큰을 요구하면,
  토큰을 채워 넣지 못하는 위조 요청은 거부된다.
- **② 안 하면**: SameSite 미지원 브라우저나 `SameSite=None` 흐름에서 방어선이 비어,
  쿠키 자동 첨부만으로 요청이 통과한다.
- **③ 대안 / 패턴**:
  - **Synchronizer token** — 서버가 세션에 토큰을 저장하고 폼에 hidden 필드로 심음.
    서버 측 세션 상태가 필요해 SPA/분산 환경에서 부담.
  - **Double-submit cookie** — 토큰을 쿠키 + 요청 헤더(또는 폼)에 **둘 다** 실어
    서버는 두 값이 일치하는지만 본다. 서버 상태가 불필요해 stateless에 적합.
  - **Custom header** — `X-CSRF-Token` 같은 커스텀 헤더는 단순 폼/이미지로는 못 붙이고
    크로스사이트 JS는 CORS preflight에 막힌다. 그 자체로 약한 CSRF 방어가 된다.
- **④ 트레이드오프**: SPA/마이크로서비스에서 토큰 발급·갱신·전파 관리가 복잡해진다.
  그래서 double-submit cookie + custom header 조합이 자주 쓰인다.

```html
<!-- synchronizer token: 서버가 세션 토큰을 hidden 필드로 주입 -->
<form action="/transfer" method="POST">
  <input type="hidden" name="csrf_token" value="9f8b2c...session-bound">
  <input name="amount" value="100">
</form>
```

```http
# double-submit: 쿠키 값과 헤더 값이 일치해야 통과
Cookie: csrf=9f8b2c...
X-CSRF-Token: 9f8b2c...
```

### 4.3 보조 방어 — Origin/Referer 검증, 안전 메서드, 재인증

- **Origin/Referer 검증** — 상태 변경 요청의 `Origin`/`Referer` 헤더가 자기 출처인지
  확인. 토큰의 보강 수단(헤더 누락/프록시 제거 케이스가 있어 단독 방어는 약함).
- **상태 변경은 GET 금지** — GET/HEAD는 **안전(safe) 메서드** 규약상 부작용이
  없어야 한다. 송금·삭제·변경은 반드시 `POST`/`PUT`/`DELETE`로. 그래야 `<img src>`
  GET 부작용 공격(3절)이 원천 차단되고 SameSite·토큰 방어와도 정합한다.
- **민감 작업 재인증** — 비밀번호·이메일 변경, 고액 송금 등은 비밀번호 재입력 또는
  step-up 인증을 요구. CSRF가 토큰을 우회해도 마지막 관문이 남는다.

### 4.4 Bearer 토큰 기반 SPA는 왜 CSRF에 강한가

- **① 왜 필요/강점**: 자격 증명을 쿠키가 아니라 `Authorization: Bearer <token>`
  헤더로 보내면, **브라우저가 자동으로 첨부하지 않는다.** 모든 요청에 JS가 명시적으로
  헤더를 붙여야 하므로, evil 페이지의 자동 폼/이미지로는 인증 요청을 만들 수 없다 —
  ambient authority가 사라지므로 CSRF의 전제 자체가 무너진다.
- **② 안 하면 / 함정**: 편의를 위해 **토큰을 쿠키에 저장**하면, 다시 브라우저가
  자동 첨부 → CSRF 노출로 되돌아간다. "Bearer라서 안전"이 아니라 "쿠키로 자동
  첨부되지 않아서 안전"임을 정확히 이해해야 한다.
- **③ 대안**: 쿠키 세션 + SameSite + 토큰(4.1~4.2). 어느 쪽이든 일관되게 적용.
- **④ 트레이드오프 (저장 위치)**: 토큰을 `localStorage`에 두면 자동 첨부는 없지만,
  **XSS가 발생하면 JS가 토큰을 읽어 탈취**한다(CSRF는 피하고 XSS에 노출). 토큰을
  `HttpOnly` 쿠키에 두면 XSS의 직접 탈취는 막지만 **CSRF에 다시 노출**된다. 즉
  **CSRF 노출(쿠키) ↔ XSS 탈취(localStorage)** 의 교환이며, 정답은 보통
  "쿠키 저장 + SameSite + CSRF 토큰" 또는 "단명 access 토큰 + 강한 XSS 방어"를
  명시적으로 선택하는 것이다.

## 5. 관련 노트

- [[authentication-session]] — 세션 쿠키 플래그(HttpOnly/Secure/SameSite)와 수명 관리
- [[security-misconfiguration]] — SameSite 기본값/CORS 설정 결함이 CSRF 표면을 넓힘
- [[injection]] — XSS와의 대비(코드 실행 vs 요청 위조), XSS는 CSRF 방어를 무력화
- [[../security|↑ security]]
