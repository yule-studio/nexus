---
title: "Broken Access Control (OWASP A01) — 인가 우회"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, authorization, idor, access-control, owasp]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[ssrf]]", "[[authentication-session]]"]
---

# Broken Access Control (OWASP A01) — 인가 우회

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — IDOR/함수레벨 인가/헤더 신뢰 + 방어 |

**[[web-security|↑ web-security]]**

> OWASP Top 10 2021 1위. "인증된 사용자가 자기 권한 밖 자원/기능에 접근".

## S1. 개요 — 인증과 인가의 분리

Broken Access Control(이하 BAC)은 **인증(authentication)은 통과했지만
인가(authorization)가 무너진** 상태를 말한다. 이 둘은 다른 질문에 답한다.

- 인증: "너는 누구냐?" → 로그인으로 신원을 확립한다.
- 인가: "그 신원으로 *이 자원/이 동작*을 해도 되느냐?" → 매 요청마다
  판정해야 한다.

BAC의 본질은 인가 판정이 **누락되거나, 잘못된 입력으로 판정되는** 것이다.
릴스에서 본 "관리자 페이지에 특정 헤더만 붙여 바로 들어간 사건"은 이
범주의 전형이다. 서버가 "이 요청자가 관리자인가?"를 신뢰할 수 없는
입력(헤더)으로 판정했기 때문에 인증을 통과한 누구나 관리자 행세를 했다.

OWASP가 2021년 BAC를 5위에서 1위로 올린 이유는 빈도 때문이다. 검사한
애플리케이션의 약 94%에서 어떤 형태든 접근 통제 결함이 발견됐고, 평균
발생률이 다른 어떤 카테고리보다 높았다. 코드 결함이라기보다 **설계
누락**(검증 지점을 빠뜨림)인 경우가 많아서, 단일 라이브러리 패치로
막히지 않는다는 점이 위험하다.

## S2. 공격 유형

### S2-1. IDOR (Insecure Direct Object Reference)

식별자를 그대로 노출하고 **소유권을 검증하지 않을 때** 발생한다. URL이나
바디의 ID만 바꿔 남의 자원에 접근한다.

```bash
# 내 주문 (정상)
curl https://shop.example.com/api/orders/123 \
  -H "Authorization: Bearer <user-A-token>"

# ID만 +1 → 남(user-B)의 주문이 그대로 응답되면 IDOR
curl https://shop.example.com/api/orders/124 \
  -H "Authorization: Bearer <user-A-token>"
```

서버가 "orders/124 조회 요청자가 user-A인데, 124의 소유자도 user-A인가?"를
묻지 않고 단지 "로그인했는가?"만 봤기 때문이다. 순차 정수 ID는 열거
(enumeration)를 쉽게 만든다. 다음처럼 전 범위를 긁을 수 있다.

```bash
for id in $(seq 1 10000); do
  curl -s -o "dump_$id.json" \
    "https://shop.example.com/api/orders/$id" \
    -H "Authorization: Bearer <user-A-token>"
done
```

쓰기 IDOR은 더 위험하다. `PATCH /api/users/124/email`이 소유권 검증
없이 동작하면 타인 계정 이메일을 바꿔 탈취로 이어진다.

### S2-2. Missing function-level authorization (함수 레벨 인가 누락)

특정 기능(관리자 엔드포인트 등)이 화면에서 링크를 숨겼을 뿐 서버에서
role 검사를 하지 않는 경우다. URL만 알면 누구나 접근한다(forced
browsing). **릴스 예시가 정확히 여기에 속한다.**

```http
GET /admin/users HTTP/1.1
Host: app.example.com
Authorization: Bearer <일반-사용자-토큰>
```

UI는 일반 사용자에게 `/admin`을 보여주지 않지만, 핸들러가
`requireRole("admin")`을 호출하지 않으면 200을 돌려준다. "메뉴에 안
보이니까 못 들어온다"는 보안이 아니다(security through obscurity).
HTTP 메서드 단위 누락도 흔하다 — `GET /admin/users`는 막혀 있는데
`DELETE /admin/users/5`는 검사를 빠뜨리는 식이다.

### S2-3. 클라이언트 공급 값을 권한 신호로 신뢰

가장 직접적으로 릴스 사건과 닮은 유형. 서버가 **자기가 발급하지 않은
값**을 권한 판정에 쓴다.

```http
# 클라이언트가 임의로 붙인 헤더를 서버가 그대로 신뢰
GET /admin/dashboard HTTP/1.1
Host: app.example.com
X-User-Role: admin
Cookie: session=<평범한-사용자-세션>
```

변형들:

- **JWT 무검증 신뢰**: `alg: none` 토큰을 받아들이거나, payload의
  `"role":"admin"`을 서명 검증 없이 디코드해 그대로 쓴다.
- **hidden form field / 바디 파라미터**: `<input type="hidden"
  name="role" value="user">`를 `value="admin"`으로 조작해 전송.
- **신뢰 헤더 위조**: 보통 프록시가 채우는 `X-Forwarded-For`,
  `X-User-Id` 등을 외부 요청이 직접 채워 신뢰 경계를 속인다.

실제 대표 사례가 **CVE-2025-29927**(Next.js 미들웨어 인가 우회)다.
공격자가 내부용 헤더 `x-middleware-subrequest`를 직접 주입하면
미들웨어 실행 자체를 건너뛰게 만들어, 미들웨어에 의존하던 인증/인가
검사를 통째로 우회했다. "프레임워크가 내부적으로만 채운다고 가정한
헤더"를 외부가 위조할 수 있다는 점에서 S2-3의 교과서적 사례다. 상세
재현/패치 노트:
[[../../../60-troubleshooting/frontend/nextjs-middleware-auth-bypass|nextjs-middleware-auth-bypass]].

### S2-4. 수직/수평 권한 상승

- **수평 권한 상승(horizontal)**: 같은 권한 등급의 *다른 사용자* 자원에
  접근. user-A가 user-B의 주문/프로필을 읽는 IDOR이 전형.
- **수직 권한 상승(vertical)**: 더 낮은 등급이 더 높은 등급의 권한을
  획득. 일반 사용자 → 관리자. S2-2/S2-3이 보통 수직 상승으로 귀결된다.

구분이 중요한 이유는 방어 지점이 다르기 때문이다. 수평은 **소유권
검증**(resource.owner == subject)으로, 수직은 **기능 권한
검증**(subject.role can perform action)으로 막는다. 둘 다 필요하다 —
하나만 있으면 다른 한쪽으로 뚫린다.

### S2-5. 메타데이터/경로 조작

- **Path traversal**: 파일 경로 파라미터에 `../`를 넣어 의도 밖 파일
  접근. `GET /files?name=../../etc/passwd`.
- **Forced browsing**: 인덱싱되지 않은 경로/백업 파일 추측 접근 —
  `/.git/config`, `/api/v1/internal/...`, `/admin.old`.
- **CORS 오설정**: `Access-Control-Allow-Origin`을 요청 Origin으로
  무조건 반영하면, 악성 사이트가 인증된 사용자 브라우저로 보호 자원을
  읽는다.

```bash
# traversal 시도
curl "https://app.example.com/files?name=../../../../etc/passwd" \
  -H "Authorization: Bearer <token>"
```

## S3. 방어

### S3-1. 핵심 원칙 — 인가 최종 판정은 서버에서 검증된 신원으로 (4구조 "왜")

가장 자주 어기는 원칙: **인가의 최종 판정을 클라이언트가 보낸 값이
아니라, 서버가 신뢰하는 신원(세션/검증된 토큰 claim)으로, 데이터에
가까운 레이어에서** 내려야 한다.

- ① 왜 필요한가: 인가의 신뢰는 "요청자가 진짜 누구인지"에 달려 있다.
  신원을 서버가 통제하는 곳(서버 세션, 서명 검증을 통과한 JWT claim)에서
  꺼내야만, 이후의 모든 권한 판정이 위조 불가능한 사실 위에 선다.
- ② 안 하면 무슨 문제: 클라이언트가 보낸 `X-User-Role`·hidden field·
  무검증 JWT를 권한 신호로 쓰면, 공격자가 그 값을 마음대로 바꿔
  즉시 관리자가 된다. 릴스 사건과 CVE-2025-29927이 정확히 이 실패다.
  게다가 판정을 UI/게이트웨이 같은 바깥 레이어에서만 하면, 우회 경로
  (직접 API 호출, 내부 서비스 콜)가 검사를 통째로 건너뛴다.
- ③ 대안: (a) 게이트웨이/프록시에서만 인가 → 우회 가능. (b) 프런트엔드
  라우팅 가드로만 보호 → 서버 직접 호출에 무력. (c) 서버 측에서, 핸들러
  진입마다, 검증된 신원으로 객체 소유권 + 기능 권한을 함께 판정 →
  유일하게 견고하다. 가능하면 ORM/데이터 접근 레이어에서 소유권을
  쿼리 조건으로 강제(`WHERE owner_id = :subject`)해 누락을 구조적으로
  막는다.
- ④ 트레이드오프: 데이터 레이어 강제는 모든 쿼리에 subject 컨텍스트를
  전파해야 하고(요청 컨텍스트 의존성 증가), 중앙 인가 게이트는 단일
  병목/단일 실패점이 될 수 있다. 그럼에도 "검사 누락 시 기본 거부"가
  "성능 약간"보다 압도적으로 우선이다. 비용은 row-level security나
  공통 미들웨어로 분산할 수 있다.

```sql
-- 소유권을 쿼리 조건으로 강제 → 핸들러가 깜빡해도 남의 행이 안 나옴
SELECT * FROM orders
WHERE id = :order_id
  AND owner_id = :authenticated_user_id;  -- 세션에서 온 신원
```

```python
# 매 요청: 기능 권한 + 객체 소유권을 함께 검증
def get_order(order_id, subject):           # subject = 검증된 세션 신원
    require_permission(subject, "order:read")          # 기능 권한
    order = repo.find(order_id)
    if order is None or order.owner_id != subject.id:  # 객체 소유권
        raise NotFound()                    # 존재 노출 회피 위해 404
    return order
```

### S3-2. Deny-by-default (명시 허용만 통과)

기본 정책을 "거부"로 두고, **명시적으로 허용된 것만** 통과시킨다.
새 엔드포인트를 추가했을 때 인가 데코레이터를 깜빡하면 열려버리는 구조
(allow-by-default)는 사고의 온상이다. 라우터에 "보호되지 않은 핸들러는
부팅 시 거부/에러"를 강제하면 누락이 런타임이 아니라 배포 전에 드러난다.

### S3-3. 두 축의 검증을 항상 함께

- **객체 소유권 검증**: 이 자원이 이 사용자 것인가 → 수평 상승 차단.
- **기능 권한 검증**: 이 사용자가 이 동작을 할 등급인가 → 수직 상승 차단.

둘 중 하나만으로는 부족하다. 소유권만 보면 일반 사용자가 자기
"주문"을 "환불 승인" 처리하는 관리자 기능을 호출할 수 있고, 기능 권한만
보면 관리자가 아닌 다른 사용자의 자원에 같은 등급으로 접근한다.

### S3-4. 클라이언트 입력 불신

- role/user-id/권한은 **요청 바디·헤더·hidden field에서 절대 읽지
  않는다**. 오직 서버 세션 또는 서명 검증을 통과한 토큰 claim에서만.
- JWT는 `alg` 고정(서버가 기대한 알고리즘만), 서명·만료·issuer·audience
  검증 후에만 claim을 신뢰.
- 프록시가 채우는 신뢰 헤더(`X-Forwarded-*`, 내부용 `x-middleware-*`)는
  **엣지에서 외부 입력을 스트립**해, 외부 요청이 위조할 수 없게 한다.

### S3-5. RBAC/ABAC + 중앙화된 인가 게이트

권한 규칙을 핸들러마다 흩뿌리지 말고 중앙 정책으로 모은다.

- **RBAC**(역할 기반): 사용자→역할→권한. 단순/감사 용이.
- **ABAC**(속성 기반): 주체·자원·환경 속성으로 정책 평가. "본인 부서
  문서만", "근무 시간 외 거부" 같은 세밀한 규칙에 적합.

중앙 인가 함수 하나(`can(subject, action, resource)`)를 통과시키면
규칙이 한곳에 모여 테스트·감사가 쉽다. 단, 이 중앙 게이트가 모든
경로에서 실제로 호출되는지를 보장하는 것(S3-2의 deny-by-default)이
전제다.

### S3-6. 예측 불가 식별자(UUID)는 보조책일 뿐

순차 ID 대신 UUID를 쓰면 열거가 어려워진다. 그러나 이는 **방어 심화
(defense in depth)일 뿐 소유권 검증의 대체가 아니다.**

- ① 왜(보조로): UUID는 추측·열거 비용을 높여 무차별 스캔을 둔화시킨다.
- ② 안 하면: 순차 ID는 `seq 1 10000` 한 줄로 전수 조사된다.
- ③ 대안: 소유권 검증(S3-1) — 이것만이 *정답을 아는* 공격자도 막는다.
- ④ 트레이드오프: UUID만 믿으면 식별자가 한 번이라도 새면(로그·Referer·
  공유 링크·캐시) 그 자원은 그대로 열린다. 즉 "추측 불가"는 "접근 통제"가
  아니다. UUID는 켜되, 소유권 검증을 그 이유로 생략하지 말 것.

## S4. 점검 관점 (테스트 시 확인)

- 모든 객체 접근 엔드포인트에서 ID를 타인 것으로 바꿔 200/데이터가
  나오는가(IDOR).
- 관리자/내부 엔드포인트에 일반 토큰으로 직접 호출 시 거부되는가.
- role/권한을 헤더·바디·hidden field로 주입하면 무시되는가.
- 프런트 가드를 우회한 직접 API 호출이 서버에서 막히는가.
- 새 엔드포인트가 인가 데코레이터 없이 배포되면 빌드/부팅이 실패하는가.

## 관련 노트

- [[../../../60-troubleshooting/backend/payment-amount-tampering|payment-amount-tampering]]
  — 클라이언트가 보낸 결제 금액을 서버가 신뢰한 사례. "클라이언트 입력
  불신"(S3-4)의 결제판.
- [[../../../60-troubleshooting/frontend/nextjs-middleware-auth-bypass|nextjs-middleware-auth-bypass]]
  — CVE-2025-29927, 헤더 위조로 미들웨어 인가 우회(S2-3).
- [[ssrf]] — 서버가 클라이언트 제어 입력을 신뢰해 내부 자원에 접근하는
  또 다른 신뢰 경계 붕괴.
- [[authentication-session]] — 인가의 전제인 신원 확립/세션 관리.
- [[../security|↑ security]]
