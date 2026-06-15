---
title: "troubleshooting · CDN 캐시 포이즈닝 / 캐시 디셉션"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, cdn, cache, cache-poisoning, cache-deception, security]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[../../20-areas/security/web-security/security-misconfiguration|security-misconfiguration]]"]
---

# CDN 캐시 포이즈닝 / 캐시 디셉션

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — unkeyed 입력/포이즈닝/디셉션 + 키 정규화 |

**[[web|↑ web]]**

CDN/리버스 프록시 캐시는 "한 번 계산한 응답을 여러 사용자에게 재사용"하는 게 본질이다. 이 재사용의 경계를 정하는 게 **캐시 키(cache key)** — 보통 `(method, host, path, 일부 쿼리)` 로 구성된다. 키에 들어가지 않는 입력이 응답을 바꾸거나, 키 판정이 실제 콘텐츠와 어긋나면 "다른 사용자에게 보여서는 안 될 응답"이 캐시를 타고 광범위하게 퍼진다. 여기서는 서로 다른 두 공격을 구분해 다룬다.

## 1. 캐시 포이즈닝 (Cache Poisoning)

### 증상
- 특정 페이지가 **모든 사용자에게 동일하게 오염**돼 보인다 — 임의 스크립트(XSS) 삽입, 외부 도메인으로 리다이렉트, 깨진 콘텐츠.
- 공격자가 한 번 요청 → 캐시에 독성 응답 저장 → 이후 정상 사용자 다수에게 그대로 서빙된다. 피해 범위가 "캐시 TTL 동안 그 키로 들어온 모두"라 한 번의 요청이 대규모로 번진다.
- 캐시를 비우면 일시적으로 정상화됐다가, 공격자가 다시 한 방 넣으면 재오염된다.

### 원인 / 메커니즘
CDN/캐시가 **캐시 키에 포함하지 않는 입력(unkeyed input)** — 대표적으로 `X-Forwarded-Host`, `X-Forwarded-Scheme`, `X-Forwarded-Port`, `X-Host`, 그 밖의 디버그/리라이트 헤더 — 을 애플리케이션이 응답에 **반사(reflect)** 하는 게 핵심이다.

애플리케이션은 절대 URL을 만들 때 흔히 "요청이 들어온 호스트"를 신뢰한다. 그 호스트를 프록시 헤더에서 읽으면, 헤더는 캐시 키에 없으므로 공격자가 값을 조작해도 **정상 키로 오염 응답이 저장**된다. James Kettle의 "Practical Web Cache Poisoning" 연구가 이 클래스를 체계화한 출처다.

```http
GET / HTTP/1.1
Host: www.example.com
X-Forwarded-Host: evil.com
```

애플리케이션이 `X-Forwarded-Host`로 스크립트 베이스 URL을 만들면 응답은 이렇게 나간다.

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=300
X-Cache: miss
Content-Type: text/html

<script src="//evil.com/static/app.js"></script>
```

캐시 키는 `Host: www.example.com` + path `/` 뿐이므로, 이 오염 응답이 정상 키로 저장된다. 이후 정상 사용자는 `X-Forwarded-Host`를 보내지도 않았는데 `//evil.com/app.js`를 그대로 받는다.

```http
GET / HTTP/1.1
Host: www.example.com

HTTP/1.1 200 OK
X-Cache: hit              # 캐시에서 서빙 — evil.com 스크립트 그대로
<script src="//evil.com/static/app.js"></script>
```

리다이렉트(`Location` 헤더에 반사), 이메일 안의 비밀번호 재설정 링크, OG 이미지 URL 등도 같은 경로로 오염된다.

## 2. 캐시 디셉션 (Web Cache Deception)

### 증상
- **로그인한 사용자의 개인정보 페이지가 정적 파일처럼 캐시**되어, 다른 사람이 그 URL로 회수해 노출된다.
- 사용자가 무심코 연 `…/account/foo.css` 같은 링크 한 번에 자기 계정 페이지(이름·이메일·토큰)가 캐시에 박힌다.

### 원인 / 메커니즘
백엔드 라우팅과 CDN 캐시 판정의 **기준이 다른 게** 원인이다. Omer Gil의 "Web Cache Deception" 연구가 출처다.

```http
GET /account/nonexistent.css HTTP/1.1
Host: site.example.com
Cookie: session=<피해자 세션>
```

- 백엔드: 라우터가 "뒤의 잔여 경로 무시" 또는 trailing-path 매칭으로 동작해 `/account`(인증된 개인 페이지)를 렌더한다.
- CDN: URL이 `.css`로 끝나니 **확장자만 보고 "정적 자원"이라 판단**, `Cache-Control`을 무시하고 캐시한다.

```http
HTTP/1.1 200 OK
Cache-Control: private, no-store     # 백엔드는 막으려 했지만…
Content-Type: text/html
X-Cache: stored                      # …CDN이 확장자 규칙으로 덮어써 저장

<h1>홍길동 님의 계정</h1> <span>token=...</span>
```

공격자는 피해자가 그 링크를 한 번 열게 한 뒤, **쿠키 없이** 같은 `/account/nonexistent.css`를 요청해 캐시에서 피해자 개인정보를 회수한다.

## 3. 해결

1. **unkeyed 입력 반사 금지** — 응답 생성(절대 URL, `Location`, 스크립트 src)에 `X-Forwarded-Host` 같은 신뢰 못할 헤더를 쓰지 않는다. 호스트는 서버 설정의 고정 canonical 값에서 가져온다. 꼭 써야 하면 캐시 키에 포함하거나 화이트리스트로 정규화한다.
2. **캐시 키 정규화/명시** — "무엇으로 캐시할지"를 명시한다(경로 + 정규화된 쿼리 + 필요한 `Vary`). 엣지에서 신뢰 못할 헤더는 오리진 전달 전에 제거(strip)한다.
3. **확장자가 아니라 실제 Content-Type/경로 규칙으로 캐시 판단** — `.css`라고 무조건 캐시하지 않는다. `Content-Type: text/html`이거나 인증 컨텍스트면 캐시 금지. `Cache-Control: private`/`no-store`가 붙은 응답은 CDN이 절대 저장 못 하게 강제한다.
4. **개인화 응답에 no-store 강제 + Vary 정확히** — 세션/쿠키 의존 응답에는 `Cache-Control: private, no-store`. 캐시 가능한 응답이라도 응답을 가르는 헤더는 `Vary`에 정확히 선언한다.

### 핵심 방어 선택: "unkeyed 입력 제거" vs "캐시 키에 포함" (4구조)

unkeyed 헤더를 응답에 반사해야 하는 상황에서, 두 갈래의 방어가 있다. 어느 쪽을 택하느냐가 보안과 캐시 효율을 가른다.

- **① 왜 필요한가** — 응답을 바꾸는 입력이 캐시 키 밖에 있으면, 그 입력이 만든 응답이 "다른 입력으로 들어온 사용자"에게 재사용된다. 이 불일치를 없애야 오염이 원천 차단된다.
- **② 안 하면 무슨 문제** — 둘 다 안 하면(헤더를 반사하면서 키에도 안 넣으면) 1절의 포이즈닝이 그대로 성립한다. 한 번의 악성 요청이 TTL 동안 전 사용자에게 퍼진다.
- **③ 대안 두 가지**
  - *제거(권장 기본값)*: 응답 생성에서 unkeyed 헤더를 아예 안 쓴다. 호스트는 고정값. 가장 안전하다.
  - *키 포함*: 헤더를 캐시 키 구성요소로 넣어, 값이 다르면 다른 캐시 엔트리로 분리한다. 멀티테넌트처럼 호스트별 응답이 정말 달라야 할 때만.
- **④ 트레이드오프** — *제거*는 안전하지만 "요청 호스트에 따라 동적으로 달라지는 응답"을 못 만든다(기능 제약). *키 포함*은 유연하지만 키에 들어간 헤더 값의 가짓수만큼 캐시 엔트리가 쪼개져 **캐시 적중률이 떨어지고**, 공격자가 임의 헤더 값을 무한히 보내면 **키 폭발(cache key explosion)**로 캐시가 오히려 무력화·오리진 과부하를 부른다. 그래서 키 포함은 반드시 **유효 값 화이트리스트 정규화**와 함께여야 한다.

구체 예시: 멀티테넌트 SaaS가 `Host`별 브랜딩을 한다면 `Host`를 키에 넣되 허용 도메인 목록으로 정규화한다. 단일 사이트라면 그냥 canonical 호스트 고정(제거)이 정답이다.

## 4. 재발 방지

- **CDN 캐시 규칙 인벤토리** — "어떤 경로가 어떤 키로 캐시되는가"를 문서화해 정기 리뷰. 확장자 기반 규칙(`*.css`, `*.js`)이 인증 경로를 덮지 않는지 확인.
- **개인화 응답 no-store 강제** — 세션 의존 응답에 `Cache-Control: private, no-store`를 미들웨어 레벨에서 일괄 부여. 누락 시 빌드/배포 게이트에서 실패.
- **비정상 키 모니터링** — 캐시 적중 로그에서 예상 밖 키(예: 인증 경로의 `hit`, 비정상 확장자 조합) 알람.
- **포이즈닝 테스트** — `X-Forwarded-Host` 등 헤더 fuzz로 응답 반사·캐시 저장 여부를 회귀 검사에 포함.

cross-link: [[web]], [[../../20-areas/security/web-security/security-misconfiguration|security-misconfiguration]], [[../../20-areas/security/web-security/injection|injection]](XSS 반사 경로), [[../frontend/service-worker-stale-cache]](클라이언트 측 캐시 경계), [[../troubleshooting|↑ 60-troubleshooting]]
