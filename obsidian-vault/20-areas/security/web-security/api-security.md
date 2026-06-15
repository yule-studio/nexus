---
title: "API 보안 (OWASP API Top 10) — BOLA/Mass Assignment/과다노출"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, api-security, bola, mass-assignment, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[broken-access-control]]", "[[authentication-session]]"]
---

# API 보안 (OWASP API Top 10) — BOLA/Mass Assignment/과다노출

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — BOLA/BFLA/mass assignment/과다 노출 |

**[[web-security|↑ web-security]]**

> 웹 OWASP Top 10 과 별도의 OWASP API Security Top 10. API 는 UI 게이트 없이 객체/기능을 직접 노출해 인가 결함이 더 치명적.

## S1. 개요 — API 가 웹 화면과 다른 점

전통적 웹 애플리케이션은 서버가 HTML 을 렌더해 내려주고, 사용자는 화면에
보이는 버튼/링크만 누른다. 즉 **UI 자체가 1차 게이트** 역할을 했다. 관리자
버튼이 화면에 없으면 일반 사용자는 그 동작을 알기도 어려웠다. API 는 이
중간 게이트가 사라진다. 다음 세 가지가 본질적으로 다르다.

- **객체 ID/필드를 클라이언트에 직접 노출**한다. `GET /api/orders/123`
  처럼 자원 식별자가 URL 에 그대로 드러나고, 응답 JSON 의 필드명도 그대로
  보인다. 추측·열거가 쉽다.
- **클라이언트가 곧 신뢰 못할 환경**이다. 모바일 앱·SPA·서드파티 통합이
  API 를 직접 호출한다. 요청은 Burp/curl 로 얼마든지 조작된다. "프론트가
  검증하니까" 는 성립하지 않는다.
- **기능 단위가 엔드포인트로 1:1 노출**된다. `DELETE /api/users/{id}`
  같은 관리 동작이 URL 하나로 호출 가능하다.

결과적으로 인가(누가 무엇을 해도 되는가)와 데이터 노출(무엇을 내려줘도
되는가) 결함이 웹보다 더 흔하고 더 크게 터진다. 그래서 OWASP 는 웹
Top 10 과 **별도로** API Security Top 10 을 둔다.

## S2. OWASP API Security Top 10 (2023)

전체를 나열하되, 실무에서 가장 자주 터지는 4개를 깊게 다룬다.

### S2-1. API1 BOLA (Broken Object Level Authorization)

API 1위 항목. 객체 단위 소유권 검증이 빠진 결함으로, 웹의 IDOR 의 API
판이다. (자세한 원리는 [[broken-access-control]] 의 IDOR 절과 동일.)

```bash
# 내 주문 (정상)
curl https://shop.example.com/api/orders/123 \
  -H "Authorization: Bearer <user-A-token>"

# {id} 만 +1 → user-B 의 주문이 응답되면 BOLA
curl https://shop.example.com/api/orders/124 \
  -H "Authorization: Bearer <user-A-token>"
```

서버가 "124 의 소유자가 요청자 user-A 인가?" 를 검증하지 않으면, 토큰은
유효하므로(인증 OK) 그대로 남의 객체가 내려간다.

### S2-2. API3 객체 속성 레벨 인가 — 과다 노출 + Mass Assignment

같은 뿌리(객체의 *필드* 단위 인가 누락)에서 출력 방향과 입력 방향으로
갈라진다.

**과다 데이터 노출(출력)** — 서버가 객체 전체를 직렬화해 내려주고
"민감 필드는 클라가 화면에 안 그리겠지" 라고 가정한다. raw 응답에는
그대로 다 들어있다.

```json
// GET /api/users/42 — 클라는 name 만 쓰지만 응답 raw 에 전부 노출
{
  "id": 42,
  "name": "최유리",
  "email": "yuri@example.com",
  "passwordHash": "$2b$12$9aF...",
  "isAdmin": false,
  "internalRiskScore": 0.81
}
```

**Mass assignment(입력)** — 요청 JSON 을 모델 객체에 통째로 바인딩하면,
사용자가 보내선 안 되는 필드(`role`/`isAdmin`/`balance`)를 주입해 권한을
끌어올린다.

```json
// PATCH /api/users/42 — 의도는 name 수정뿐인데 role 을 끼워 넣음
{ "name": "최유리", "role": "admin" }
```

취약 코드는 입력을 엔티티에 직결한다.

```python
# 취약 — 요청 body 전체를 모델에 바인딩
user.update(**request.json)          # role=admin 까지 그대로 반영
return user.to_dict()                # 출력도 전체 직렬화 → 과다 노출
```

안전 코드는 **입력/출력 모두 명시 필드만** 통과시킨다.

```python
# 안전 — 입력 allowlist + 출력 DTO
ALLOWED_IN = {"name", "displayName"}
patch = {k: v for k, v in request.json.items() if k in ALLOWED_IN}
user.update(**patch)

def to_public_dto(u):                # 출력 allowlist
    return {"id": u.id, "name": u.name, "email": u.email}
return to_public_dto(user)
```

### S2-3. API5 BFLA (Broken Function Level Authorization)

객체가 아니라 **기능/동작** 단위 인가가 빠진 결함. 일반 사용자가 관리자
전용 엔드포인트나 메서드를 호출한다. 흔히 HTTP 메서드를 바꿔 우회한다.

```bash
# 일반 사용자 토큰으로 관리자 전용 동작 호출
curl -X DELETE https://shop.example.com/api/users/77 \
  -H "Authorization: Bearer <normal-user-token>"

# GET 만 막고 POST 는 안 막은 경우 — 메서드 변경으로 우회
curl -X POST https://shop.example.com/api/admin/feature-flags \
  -H "Authorization: Bearer <normal-user-token>" \
  -d '{"flag":"free_premium","enabled":true}'
```

라우트 prefix(`/admin`)로 추측 가능한 데다, 메서드별 인가를 따로 깜빡하면
열린다.

### S2-4. API4 Unrestricted Resource Consumption

레이트 리밋·쿼터·페이지 크기 상한이 없으면 한 클라이언트가 자원을
독식한다. 비용 폭증(과금 API/LLM 호출)이나 DoS 로 직결된다.

```bash
# page 크기 상한이 없으면 한 방에 전수 덤프 + DB 부하
curl "https://shop.example.com/api/orders?limit=10000000"
```

### S2-5. 나머지 항목 (짧게)

- **API2 Broken Authentication** — 토큰/세션 검증 결함. 상세는
  [[authentication-session]].
- **API6 Unrestricted Access to Sensitive Business Flows** — 민감
  비즈니스 플로우(티켓 구매·포인트 적립)를 봇으로 자동화·악용.
- **API7 SSRF** — 서버가 클라가 준 URL 을 그대로 호출. 상세는 [[ssrf]].
- **API8 Security Misconfiguration** — CORS 와일드카드·디버그 모드·기본
  자격. 상세는 [[security-misconfiguration]].
- **API9 Improper Inventory Management** — shadow/zombie API. 옛 `/v1`
  을 폐기 안 해 패치 안 된 엔드포인트가 살아있다.
- **API10 Unsafe Consumption of APIs** — 다운스트림(서드파티) API 응답을
  무검증으로 신뢰해 소비.

## S3. 실제 사례

- **Optus 2022 (호주)** — 인증이 걸리지 않은 API 엔드포인트 + **열거
  가능한 고객 ID** 조합으로 약 1천만 명 개인정보가 유출됐다. "ID 만 +1
  하면 다음 사람" 이라는, BOLA + 추측 가능 ID 의 교과서적 결합이다.
- **Parler 2021** — 게시물 ID 가 순차 증가라 ID 를 1씩 늘리며 전체
  게시물을 스크래핑했다. 동일한 결(열거 가능 식별자)에서 나온 사고다.

## S4. 방어

### S4-1. (왜) 모든 요청에서 객체 소유권을 서버가 재검증한다 (BOLA)

- **왜 필요한가** — API 는 UI 게이트가 없어 ID 추측이 곧 접근이다. 핸들러
  안에서 "이 객체의 소유자가 요청자인가?" 를 매번 판정해야만 한다.
- **안 하면 무엇이 문제인가** — Optus 처럼 ID 를 순회하며 전 사용자
  데이터가 전수 유출된다. 토큰은 유효하므로 인증 로그만 봐선 정상으로
  보인다.
- **대안은** — 추측 불가 ID(UUID/ULID)는 *열거 난이도*만 올릴 뿐 소유권
  검증을 대체하지 못한다. 유출된 UUID 하나면 그대로 뚫린다. 보조 수단일
  뿐이다.
- **트레이드오프** — 매 핸들러에 검증을 넣으면 보일러플레이트가 는다.
  정책 미들웨어/공통 가드(`require_owner(resource)`)로 한 곳에 모아
  상쇄한다.

### S4-2. (왜) 입력/출력 모두 명시적 필드 allowlist(DTO/serializer)

- **왜 필요한가** — 입력 allowlist 는 mass assignment 를(보내선 안 되는
  필드 차단), 출력 DTO 는 과다 노출을(내려선 안 되는 필드 차단) 막는다.
  같은 "필드 단위 인가" 문제의 양방향이다.
- **안 하면 무엇이 문제인가** — 입력이 새면 `isAdmin` 주입으로 권한
  상승, 출력이 새면 `passwordHash`/PII 노출. 한쪽만 막아도 절반은 열려
  있다.
- **대안은** — 블록리스트(민감 필드만 제외)는 새 필드가 추가될 때마다
  빠뜨리기 쉬워 안전하지 않다. allowlist 가 기본이어야 한다.
- **트레이드오프** — DTO/serializer 매핑 코드가 늘어난다. 하지만 "엔티티
  를 응답·요청에 직결" 하는 편의의 대가가 권한 상승/유출이므로, 매핑
  수고가 정상적인 비용이다.

### S4-3. 보조 통제

- 레이트 리밋·쿼터·페이지 크기 상한(API4/API6 대응).
- API 게이트웨이 + **인벤토리 관리** — 폐기된 `/v1` 같은 shadow/zombie
  API 를 목록화·차단(API9).
- 함수 레벨 인가 — 라우트별·메서드별로 역할을 명시 검증(API5/BFLA).

## S5. 관련 노트

- [[broken-access-control]] — BOLA 는 곧 IDOR(객체 소유권 검증).
- [[authentication-session]] — API 인증/토큰 검증(API2).
- [[security-misconfiguration]] — CORS/기본 자격/디버그 노출(API8).
- [[ssrf]] — 서버측 요청 위조(API7).
- [[../security|↑ security]]
