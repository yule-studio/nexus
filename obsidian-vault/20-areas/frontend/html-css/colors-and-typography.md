---
title: "색과 폰트 (Colors & Typography)"
kind: knowledge
project: frontend
status: current
tags: [frontend, css, beginner, color, font, typography]
---

# 색과 폰트

**[[html-css|↑ HTML/CSS hub]]**

---

## 1. 색 표현법

```css
color: red;                      /* 이름 (140 종) */
color: #4ade80;                  /* HEX 6 자리 — 가장 자주 */
color: #4ade80aa;                /* HEX 8 자리 (마지막 2 자리 = 투명도) */
color: #4d8;                     /* HEX 3 자리 (단축) */
color: rgb(74, 222, 128);        /* RGB 0~255 */
color: rgba(74, 222, 128, 0.5);  /* RGB + alpha 0~1 */
color: hsl(141, 71%, 58%);       /* Hue, Saturation, Lightness */
color: hsla(141, 71%, 58%, 0.5);
```

### 색 고르는 도구
- **VSCode**: HEX 코드 위에 마우스 → 색 picker
- **Tailwind 컬러**: https://tailwindcss.com/docs/customizing-colors — 500개 검증된 색
- **Coolors**: https://coolors.co — 팔레트 자동 생성

### 자주 쓰는 톤
```css
/* 다크 톤 — 본인 hompage 스타일 */
--bg: #0b0d0c;
--card: #0f1311;
--text: #e6ece8;
--accent: #4ade80;

/* 라이트 톤 */
--bg: #f7f8f6;
--card: #ffffff;
--text: #161a18;
--accent: #16a34a;

/* 따뜻한 톤 */
--bg: #fef3e0;
--text: #2a1810;
--accent: #d97706;

/* 차가운 톤 */
--bg: #0a0e1a;
--text: #e0eaf5;
--accent: #60a5fa;
```

---

## 2. 폰트 패밀리

### 시스템 폰트 (가장 빠름, 추천)
```css
font-family:
  -apple-system, BlinkMacSystemFont, "Segoe UI",
  Roboto, "Helvetica Neue", Arial,
  "Apple SD Gothic Neo", "Pretendard", sans-serif;
```

OS 가 제공하는 기본 폰트 사용 — 로딩 0초.

### Google Fonts (외부 폰트)
```html
<!-- index.html <head> -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet"
  href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap">
```

```css
body { font-family: "Inter", sans-serif; }
```

### 한국어 폰트
**Pretendard** (한국 권장):
```html
<link rel="stylesheet"
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@latest/dist/web/variable/pretendardvariable.css">
```
```css
body { font-family: "Pretendard Variable", sans-serif; }
```

**Noto Sans KR** (Google):
```html
<link rel="stylesheet"
  href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;500;700&display=swap">
```

### 모노 폰트 (코드용)
```css
font-family: "JetBrains Mono", ui-monospace, Menlo, monospace;
```

---

## 3. 폰트 크기

```css
font-size: 16px;     /* 픽셀 (가장 자주) */
font-size: 1rem;     /* 루트 (html) 기준 — 보통 1rem = 16px */
font-size: 1.5em;    /* 부모 기준 1.5 배 */
font-size: 1.5vw;    /* 뷰포트 너비의 1.5% */
font-size: clamp(14px, 2vw, 18px);   /* 최소/이상/최대 */
```

### 자주 쓰는 크기 (참고)
```css
/* 본인 hompage 의 사이즈 */
--xs:   11px;     /* chip / eyebrow */
--sm:   12.5px;   /* sub / 작은 글 */
--md:   15px;     /* 본문 / title */
--lg:   17px;     /* 큰 글 */
--xl:   20px;     /* h2 */
--2xl:  26px;     /* page title */
--3xl:  34px;     /* hero */

/* Tailwind 기본 */
--text-xs:    12px;
--text-sm:    14px;
--text-base:  16px;
--text-lg:    18px;
--text-xl:    20px;
--text-2xl:   24px;
--text-3xl:   30px;
--text-4xl:   36px;
```

---

## 4. font-weight (굵기)

```css
font-weight: 100;    /* Thin */
font-weight: 300;    /* Light */
font-weight: 400;    /* Regular (= normal) */
font-weight: 500;    /* Medium */
font-weight: 600;    /* SemiBold */
font-weight: 700;    /* Bold */
font-weight: 900;    /* Black */
```

`bold` = 700, `normal` = 400.

### 폰트가 weight 를 지원해야 함
Inter / Pretendard 같은 variable font 는 100~900 다 지원. 일부 OS 기본 폰트는 400/700 만.

---

## 5. line-height (줄 간격)

```css
line-height: 1.5;     /* 무단위 — 글자 크기의 1.5 배 (권장) */
line-height: 24px;    /* 절대 — 비추천 */
line-height: 150%;    /* 글자 크기의 150% */
```

### 자주
- 한국어 본문: **1.5 ~ 1.7** (descender 공간)
- 영문 본문: 1.4 ~ 1.6
- 제목: 1.1 ~ 1.3

```css
body  { line-height: 1.6; }
h1    { line-height: 1.2; }
```

---

## 6. letter-spacing (자간)

```css
letter-spacing: 0.05em;     /* 글자 크기의 5% 만큼 더 띄움 */
letter-spacing: 1px;
letter-spacing: -0.02em;    /* 더 좁게 (제목용) */
```

### 자주
- UPPERCASE 라벨: `0.05~0.1em` 띄움
- 큰 제목: `-0.01 ~ -0.02em` 좁힘 (큰 글씨는 자동으로 헐겁게 보임)

---

## 7. text-align

```css
text-align: left;       /* 기본 */
text-align: center;
text-align: right;
text-align: justify;    /* 좌우 끝 맞춤 (한국어엔 부자연) */
```

---

## 8. text-decoration

```css
text-decoration: none;        /* 밑줄 없앰 (링크에 자주) */
text-decoration: underline;
text-decoration: line-through; /* 가운데 줄 */

text-decoration: underline wavy red;  /* 짧게 */
text-decoration-thickness: 1px;
text-underline-offset: 3px;           /* 밑줄과 글자 거리 */
```

---

## 9. text-transform

```css
text-transform: uppercase;    /* 다 대문자 */
text-transform: lowercase;
text-transform: capitalize;   /* 단어 첫 글자만 대문자 */
text-transform: none;
```

---

## 10. white-space / word-break (줄바꿈 제어)

```css
white-space: nowrap;        /* 줄바꿈 안 됨 (한 줄로 강제) */
white-space: pre;           /* 공백 / 줄바꿈 그대로 */
white-space: pre-wrap;      /* 공백 그대로 + 너무 길면 자동 줄바꿈 */
white-space: normal;        /* 기본 */

word-break: break-all;      /* 영어 단어 중간에서도 잘림 */
word-break: keep-all;       /* 한국어 어절 단위 줄바꿈 (한국어 필수) */
overflow-wrap: anywhere;    /* 긴 영문 / URL 강제 줄바꿈 */
```

### 한국어 본문에 좋은 조합
```css
body, p, h1, h2, h3 {
  word-break: keep-all;
}
```

---

## 11. ellipsis (말줄임표 …)

### 한 줄 …
```css
.title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 200px;
}
```

### 여러 줄 …
```css
.desc {
  display: -webkit-box;
  -webkit-line-clamp: 3;     /* 3 줄 까지 */
  -webkit-box-orient: vertical;
  overflow: hidden;
  text-overflow: ellipsis;
}
```

---

## 12. 자주 만드는 패턴

### 깔끔한 본문
```css
body {
  font-family: "Inter", -apple-system, sans-serif;
  font-size: 16px;
  line-height: 1.6;
  color: #333;
  background: #fff;
}
```

### 카드 제목 + 설명
```css
.card-title {
  font-size: 18px;
  font-weight: 600;
  line-height: 1.25;
  margin-bottom: 8px;
}
.card-sub {
  font-size: 14px;
  color: #666;
  line-height: 1.5;
}
```

### 모노 라벨 (대문자 + 자간)
```css
.label {
  font-family: "JetBrains Mono", monospace;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #888;
}
```

### 헤더 강조 (큰 + 좁은 자간)
```css
.hero h1 {
  font-size: clamp(40px, 6vw, 64px);
  font-weight: 700;
  line-height: 1.1;
  letter-spacing: -0.02em;
}
```

---

## 13. 다음 단계

1. **[[layout/flexbox]]** — Flexbox 1차원
2. **[[layout/grid]]** — Grid 2차원
3. **[[responsive/responsive-design]]** — 반응형

---

연결: [[html-css|↑ hub]] · [[css-basics]] · [[layout/flexbox]]
