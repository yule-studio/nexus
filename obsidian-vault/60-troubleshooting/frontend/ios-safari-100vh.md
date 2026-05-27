---
title: "troubleshooting · iOS Safari 100vh — viewport 가 잘림"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, css, mobile, safari, viewport]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# iOS Safari 100vh — viewport URL bar 포함 계산

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- `height: 100vh` 한 footer / hero section 이 iOS Safari 에서 화면 밖
- 스크롤 시 URL bar 가 사라지면서 element 크기 변함 (jiggle)
- Android Chrome 도 비슷 (정도 약함)

## 원인

iOS Safari 의 `100vh` 는 **URL bar 가 숨어 있을 때 기준** (큰 viewport). 처음 로드 시 URL bar 보이므로 실제 visible 영역은 100vh 보다 작음.

## 해결

### `100dvh` (Dynamic Viewport Unit) — modern
```css
.hero {
  height: 100dvh;   /* dynamic — URL bar 변화에 따라 자동 조정 */
}
```

지원: Safari 15.4+ / Chrome 108+. fallback 필요:

```css
.hero {
  height: 100vh;     /* fallback */
  height: 100dvh;    /* 지원 브라우저 */
}
```

또는 `100svh` (small viewport) / `100lvh` (large viewport).

### CSS custom property + JS
```css
.hero {
  height: calc(var(--vh, 1vh) * 100);
}
```

```js
function setVh() {
  document.documentElement.style.setProperty('--vh', `${window.innerHeight * 0.01}px`);
}
setVh();
window.addEventListener('resize', setVh);
window.addEventListener('orientationchange', setVh);
```

### `-webkit-fill-available` (Safari 만)
```css
.hero {
  height: 100vh;
  height: -webkit-fill-available;
  min-height: -webkit-fill-available;
}
```

→ Safari 에서만 작동, 다른 브라우저 무시.

### Flexbox / grid 로 우회
```css
html, body { height: 100%; margin: 0; }
.app {
  height: 100%;
  display: flex;
  flex-direction: column;
}
.main { flex: 1; }
.footer { /* fixed height */ }
```

→ vh 안 쓰고 부모-자식 100% chain. 단 `html / body height: 100%` 필수.

## 안전 영역 (Notch / Home indicator)

iPhone notch / 홈 인디케이터 영역:

```css
.app {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}

/* meta tag 필요 */
```

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

## 진단

```
# Safari iOS — 개발자 도구
1. Mac Safari → Preferences → Advanced → "Show Develop menu"
2. iPhone 연결 + Develop → 기기 선택
3. Inspector 로 element 크기 측정
```

## 재발 방지

- `100dvh` + fallback 패턴 표준
- Storybook 의 mobile viewport preset 으로 사전 검증
- e2e (Playwright iOS WebKit) 에 viewport 회귀 테스트

## 관련

- [[frontend]] / [[css-grid-blowout]]
