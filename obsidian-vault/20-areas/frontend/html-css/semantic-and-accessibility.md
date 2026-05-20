---
title: "Semantic HTML & 접근성"
kind: knowledge
project: frontend
status: current
tags: [frontend, html, accessibility, a11y, semantic]
---

# Semantic HTML & 접근성

**[[html-css|↑ HTML/CSS hub]]**

> 의미 있는 태그 사용 + 키보드 / 스크린 리더 사용자 배려.

---

## 1. Semantic 이란

```html
<!-- ❌ 의미 없음 -->
<div>제목</div>
<div>
  <div>...</div>
</div>

<!-- ✅ 의미 있음 -->
<h1>제목</h1>
<main>
  <article>...</article>
</main>
```

같은 화면이라도 의미 있는 태그가:
- 검색엔진 (SEO) 친화
- 스크린 리더 친화 (시각 장애)
- 코드 가독성 좋음

---

## 2. 페이지 골격 (landmark)

```html
<body>
  <header>
    <h1>사이트 이름</h1>
    <nav>
      <a href="/">Home</a>
      <a href="/about">About</a>
    </nav>
  </header>

  <main>
    <article>
      <h2>글 제목</h2>
      <p>본문...</p>
    </article>
    <aside>
      <h3>관련 글</h3>
    </aside>
  </main>

  <footer>
    <p>© 2026</p>
  </footer>
</body>
```

### 각 태그
- `<header>` — 페이지 / 섹션 머리 (로고, 메인 메뉴)
- `<nav>` — 주 네비게이션
- `<main>` — 본문 영역 (페이지에 1개)
- `<article>` — 독립적 콘텐츠 (블로그 글, 카드)
- `<section>` — 의미 있는 섹션
- `<aside>` — 사이드 보조 정보
- `<footer>` — 페이지 / 섹션 꼬리

---

## 3. heading 위계

```html
<h1>페이지 제목 (1개만)</h1>
  <h2>큰 섹션</h2>
    <h3>서브 섹션</h3>
      <h4>더 작은</h4>
```

### 규칙
- 페이지에 `<h1>` 1번만
- 순서대로 (h1 → h3 점프 금지)
- 스크린 리더가 heading 으로 페이지 구조 읽음

---

## 4. ARIA — 보완 속성

HTML 만으로 의미 표현 부족할 때 보완.

### aria-label — 텍스트 없는 요소의 라벨
```html
<button aria-label="검색">🔍</button>
<a href="/" aria-label="홈으로">
  <img src="logo.png" alt="">
</a>
```

### aria-hidden — 스크린 리더에서 숨김
```html
<span class="icon" aria-hidden>✓</span>  <!-- 장식 -->
```

### aria-current — 현재 위치 표시
```html
<a href="/about" aria-current="page">About</a>
```

본인 hompage 의 nav 항목이 사용.

### role — 역할 강제 지정
```html
<div role="button" tabindex="0">눌러</div>
```
(보통 `<button>` 쓰면 됨 — role 은 비표준 요소를 강제로 의미부여할 때)

### aria-live — 동적 알림
```html
<div aria-live="polite">로딩 중...</div>
```

값 바뀔 때 스크린 리더가 읽음.

---

## 5. 키보드 접근

### 자연스러운 tab 순서
HTML 순서 = tab 순서. 시각적 위치 (CSS 로 이동) 과 다르면 혼란.

### tabindex
```html
<div tabindex="0">tab 으로 접근 가능</div>
<div tabindex="-1">JS 로만 focus 가능 (tab 으로는 안 됨)</div>
<div tabindex="3">tab 순서 강제 (남용 금지)</div>
```

### focus 가시화 (focus-visible)
```css
:focus-visible {
  outline: 2px solid #4ade80;
  outline-offset: 2px;
}
```

→ Tab 키로 이동 시만 outline. 마우스 클릭 시엔 안 보임.

### 키보드 단축키
```html
<button accesskey="s">제출 (Alt+S)</button>
```

---

## 6. Skip link

긴 페이지에서 본문으로 바로 점프:
```html
<a href="#main" class="skip-link">본문으로 건너뛰기</a>
...
<main id="main">...</main>
```

```css
.skip-link {
  position: absolute;
  top: -100px;       /* 평소엔 화면 밖 */
  background: #4ade80;
  color: white;
  padding: 8px 16px;
}
.skip-link:focus {
  top: 0;            /* tab 으로 focus 시 보임 */
}
```

본인 hompage 의 패턴.

---

## 7. 이미지 alt

```html
<img src="cat.jpg" alt="검은 고양이가 햇볕 쬐는 모습">    <!-- 내용 -->
<img src="logo.png" alt="회사 이름">                       <!-- 의미 있음 -->
<img src="decoration.svg" alt="">                          <!-- 장식 (빈 alt 필수) -->
```

### 규칙
- 모든 `<img>` 에 `alt` (없으면 screen reader 가 파일명 읽음)
- 내용 전달 → 설명
- 장식 → 빈 `alt=""` (생략 X)

---

## 8. 폼 접근성

### label 항상
```html
<!-- ❌ -->
<input placeholder="이메일">

<!-- ✅ -->
<label for="email">이메일</label>
<input id="email" type="email">
```

### required 명시
```html
<label for="email">이메일 *</label>
<input id="email" type="email" required aria-required="true">
```

### 에러 메시지 연결
```html
<label for="email">이메일</label>
<input id="email" type="email" aria-describedby="email-error">
<div id="email-error" role="alert">이메일 형식이 잘못됐습니다</div>
```

---

## 9. 색 대비 (contrast)

WCAG 권장 비율:
- 본문 (작은 글) — 4.5:1 이상
- 큰 글 / UI — 3:1 이상

### 검사 도구
- Chrome DevTools → Lighthouse → Accessibility 탭
- https://webaim.org/resources/contrastchecker/

### 너무 흐린 글자
```css
/* ❌ 안 보임 */
color: #d1d5db;
background: white;

/* ✅ */
color: #4b5563;
background: white;
```

---

## 10. 모션 / 깜박임

### 빠른 깜박임 금지
초당 3 번 이상 깜박이면 발작 유발 가능. 피하기.

### prefers-reduced-motion 존중
```css
.card {
  transition: transform 0.3s;
}
@media (prefers-reduced-motion: reduce) {
  .card { transition: none; }
}
```

사용자가 시스템 설정 (macOS: 시스템 설정 → 손쉬운 사용 → 동작 줄이기) 으로 모션 끄면 자동 비활성.

본인 hompage 도 적용:
```css
@media (prefers-reduced-motion: reduce) {
  .page-anim > * {
    animation: none;
    opacity: 1;
  }
}
```

---

## 11. dark / light 자동 대응

```css
:root {
  --bg: #fff;
  --text: #000;
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: #000;
    --text: #fff;
  }
}

body {
  background: var(--bg);
  color: var(--text);
}
```

사용자 시스템 다크 / 라이트 따라감.

본인 hompage 는 직접 toggle + localStorage. 자세히 [[../react/styling/]]

---

## 12. lang 속성

```html
<html lang="ko">     <!-- 한국어 페이지 -->
```

검색엔진 / 스크린 리더가 발음 / 언어 결정. 필수.

다른 언어 부분만:
```html
<p>한국어 본문. <span lang="en">English</span> 가 들어감.</p>
```

---

## 13. 자주 만나는 안티패턴

### ❌ div 로 버튼 흉내
```html
<div onclick="...">눌러</div>
```
키보드 / 스크린 리더로 접근 불가.

### ✅
```html
<button onclick="...">눌러</button>
```

### ❌ # 없는 a
```html
<a onclick="...">눌러</a>
```

### ✅
```html
<a href="#" onclick="...">눌러</a>     <!-- 또는 -->
<button onclick="...">눌러</button>
```

### ❌ placeholder 만으로 label
```html
<input placeholder="이메일">
```

### ✅
```html
<label>이메일 <input></label>
```

### ❌ table 로 레이아웃
1990 년대 패턴. 표 데이터에만 `<table>`.

### ✅
Flexbox / Grid 로 레이아웃.

---

## 14. 도구

- **Lighthouse** (Chrome DevTools) — 자동 검사
- **axe DevTools** 확장 — 더 정밀
- **NVDA** (Windows) / **VoiceOver** (Mac) — 스크린 리더 직접 사용해보기
- **Tab 키만으로 사이트 이용** — 키보드 접근성 직접 확인

---

## 15. 한국 표준

웹 접근성 인증마크 (WA) — 국가 표준. 공공기관 / 큰 사이트는 필수.
- 기준: KWCAG 2.1 (한국형 WCAG 2.1)
- 검사: https://www.wa.or.kr/

---

연결: [[html-css|↑ hub]] · [[forms/forms-basics]] · [[../react/react]]
