---
title: "Clickjacking — UI redress 공격"
kind: knowledge
project: security
agent: security-review
status: current
created_at: 2026-06-15
tags: [security, web-security, clickjacking, iframe, csp, http-headers]
home_hub: "20-areas/security/web-security"
related: ["[[web-security]]", "[[security-misconfiguration]]", "[[csrf]]"]
---

# Clickjacking — UI redress 공격

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | security-review | 최초 — 원리/frame-ancestors/X-Frame-Options |

**[[web-security|↑ web-security]]**

> 피해자가 보이지 않는 iframe 위에서 "다른 것을 클릭한다고 착각"하게 만들어, 의도치 않은 행동을 유도.

## 1. 원리

Clickjacking(UI redress)은 공격자 페이지가 **타깃 사이트를 투명 iframe으로 위에 겹쳐** 놓고, 그 아래 보이는 영역에 미끼 UI("경품 받기", "동영상 재생")를 배치하는 공격이다. 사용자는 화면에 보이는 미끼 버튼을 누른다고 생각하지만, 마우스 클릭은 **맨 위에 떠 있는 투명 iframe**으로 전달되어 타깃 사이트의 실제 버튼(예: "결제", "팔로우", "권한 승인", "계정 삭제")이 눌린다.

핵심은 iframe을 `opacity: 0`(또는 `0.0001`)으로 완전히 투명하게 만들고, `z-index`로 미끼 위에 올린 뒤, 타깃 버튼이 정확히 미끼 버튼 위치에 오도록 `position`/`top`/`left`로 픽셀 단위 정렬하는 것이다.

### 1.1. CSRF와의 결정적 차이

[[csrf]]는 **요청 자체를 위조**한다 — 그래서 SameSite 쿠키나 CSRF 토큰으로 "이 요청이 우리 폼에서 나온 게 맞나"를 검증해 막을 수 있다. 반면 clickjacking은 **진짜 사용자가 진짜로 자기 손으로 클릭**한다. 세션도 진짜, 클릭도 진짜이므로 서버 입장에서는 정상 요청과 구분할 수 없다.

- CSRF 토큰: 우회됨. iframe 안에서 정상 페이지가 렌더되니 토큰도 정상 포함됨.
- SameSite=Strict/Lax: 우회됨. top-level 탐색이 아니라 사용자가 자기 세션으로 자기 사이트를 클릭하는 것.
- 따라서 **방어 지점이 다르다** — clickjacking은 "내 페이지가 남의 iframe에 들어가는 것 자체"를 막아야 한다.

### 1.2. 변형

- **likejacking** — 페이스북 "좋아요" 버튼을 투명하게 겹쳐 무단 좋아요/공유 유도.
- **cursorjacking** — 실제 커서를 숨기고 가짜 커서를 그려, 클릭 지점에 대한 인지를 왜곡.
- **드래그 앤 드롭 탈취** — 사용자의 드래그 동작을 가로채 토큰/텍스트를 외부로 옮김.
- **tapjacking** — 모바일에서 투명 오버레이로 탭을 가로챔(안드로이드 toast/overlay 권한 악용).

### 1.3. 공격 HTML 예시

```html
<!doctype html>
<html>
  <head>
    <style>
      /* 미끼: 사용자가 보는 화면 */
      #decoy {
        position: absolute;
        top: 300px;
        left: 240px;
        z-index: 1;
        font-size: 24px;
      }
      /* 타깃 사이트: 투명하게 위에 겹침 */
      iframe {
        position: absolute;
        top: 0;
        left: 0;
        width: 800px;
        height: 600px;
        opacity: 0;          /* 완전 투명 */
        z-index: 2;          /* 미끼보다 위 */
        filter: alpha(opacity=0);
      }
    </style>
  </head>
  <body>
    <button id="decoy">🎁 무료 경품 받기</button>
    <!-- 실제로 클릭되는 것은 이 iframe 안의 "결제" 버튼 -->
    <iframe src="https://bank.example.com/transfer?to=attacker&amount=1000000"></iframe>
  </body>
</html>
```

사용자는 "무료 경품 받기"를 누른다고 믿지만, 클릭은 `z-index`가 더 높은 투명 iframe으로 전달되어 송금 버튼이 눌린다.

## 2. 방어

### 2.1. (4구조) 왜 `Content-Security-Policy: frame-ancestors`가 근본 방어인가

- **① 왜 필요한가** — clickjacking의 전제는 "타깃 사이트가 공격자 페이지의 iframe 안에 렌더된다"는 것이다. `frame-ancestors`는 **누가 내 페이지를 iframe/frame/embed/object로 감쌀 수 있는지를 서버가 응답 헤더로 선언**한다. 허용 목록에 없는 출처가 임베드하려 하면 브라우저가 렌더 자체를 거부한다 → 공격의 전제가 무너진다.
- **② 안 하면 무슨 문제** — 헤더가 없으면 임의의 악성 도메인이 내 결제/관리 페이지를 투명 iframe으로 감쌀 수 있다. 진짜 사용자의 진짜 클릭이므로 CSRF 토큰·SameSite로도 사후에 못 막는다. "정상 요청"으로 처리되어 송금·권한 변경·계정 삭제까지 이어진다.
- **③ 대안은** — 과거에는 프레임 버스팅 JavaScript(`if (top !== self) top.location = self.location`)를 썼다. 하지만 이는 우회 수단이 많다(아래 2.3). 헤더 기반(`frame-ancestors` / `X-Frame-Options`)이 브라우저 레벨 차단이라 근본적이다.
- **④ 트레이드오프** — `frame-ancestors`는 CSP를 지원하는 비교적 최신 브라우저에서만 동작한다. 구형 브라우저 커버를 위해 `X-Frame-Options`를 병행해야 한다(중복 헤더 관리 비용). 또한 결제 위젯·SSO·소셜 임베드처럼 **정당한 임베드**가 있으면 화이트리스트를 정확히 유지·갱신해야 하며, 목록이 늘수록 관리·검토 비용이 커진다(허술하면 다시 구멍).

### 2.2. `X-Frame-Options` (레거시지만 병행 권장)

```
X-Frame-Options: DENY        # 어떤 출처도 임베드 금지
X-Frame-Options: SAMEORIGIN  # 동일 출처만 임베드 허용
```

- 오래됐지만 구형 브라우저까지 **폭넓게 지원**된다. 현재 베스트 프랙티스는 `frame-ancestors`와 `X-Frame-Options`를 **둘 다** 보내는 것.
- 둘의 차이: `X-Frame-Options`는 다중 도메인 화이트리스트를 표현할 수 없다(`ALLOW-FROM`은 사실상 폐기/미지원). 여러 신뢰 출처를 허용하려면 `frame-ancestors 'self' https://a.example https://b.example`처럼 CSP로만 표현 가능하다.
- 두 헤더가 충돌하면 CSP `frame-ancestors`가 지원 브라우저에서 우선한다.

### 2.3. 프레임 버스팅 JS는 약하다

`if (top !== self) { top.location = self.location; }` 같은 JS 방어는 우회 경로가 많아 신뢰할 수 없다.

- 공격자가 iframe에 `sandbox` 속성을 주면, allow-scripts는 켜되 allow-top-navigation은 빼서 **top 접근/탐색을 차단**할 수 있다 → 버스팅 코드가 무력화.
- `onbeforeunload` 가로채기, 다중 중첩 프레임, 302 트릭 등 추가 우회도 존재.
- 결론: JS는 보조일 뿐, **응답 헤더가 정답**이다.

### 2.4. 민감 액션 추가 확인

송금·권한 부여·계정 삭제 같은 고위험 액션은 헤더 방어에 더해 **명시적 의도 확인**을 둔다.

- 재인증(비밀번호/2FA 재입력), 명시적 확인 모달, 핵심 값(금액·수신자) 사용자 재입력.
- 헤더가 뚫리는 예외 상황에서도 단일 클릭만으로 돌이킬 수 없는 결과가 나지 않도록 하는 심층 방어(defense in depth).

## 3. 실제 헤더 설정 예시

기본(임베드 전면 차단):

```nginx
# nginx
add_header Content-Security-Policy "frame-ancestors 'none'" always;
add_header X-Frame-Options "DENY" always;
```

```js
// Express (helmet 미사용 시 직접)
app.use((req, res, next) => {
  res.setHeader("Content-Security-Policy", "frame-ancestors 'none'");
  res.setHeader("X-Frame-Options", "DENY");
  next();
});
```

특정 신뢰 출처에만 임베드 허용(결제 위젯 등):

```nginx
add_header Content-Security-Policy "frame-ancestors 'self' https://trusted.example.com" always;
# X-Frame-Options 는 다중 화이트리스트 표현 불가 → SAMEORIGIN 으로 보수적 병행
add_header X-Frame-Options "SAMEORIGIN" always;
```

응답 검증:

```
$ curl -sI https://app.example.com | grep -iE 'frame'
content-security-policy: frame-ancestors 'none'
x-frame-options: DENY
```

## 4. 관련 노트

- [[security-misconfiguration]] — 보안 응답 헤더(CSP/HSTS/X-Frame-Options) 누락·오설정 전반.
- [[csrf]] — 대비 개념: CSRF는 "의도(요청) 위조", clickjacking은 "실제 클릭 탈취". 방어 지점이 다름.
- [[web-security]] — 웹 보안 영역 허브.
- [[../security|↑ security]]
