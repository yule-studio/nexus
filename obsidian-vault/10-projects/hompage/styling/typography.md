---
title: "hompage styling — Typography (폰트 / 크기 / 한국어)"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:10:00+09:00
tags: [project, hompage, styling, typography, korean]
home_hub: hompage
related:
  - "[[tokens]]"
  - "[[../hompage]]"
---

# Typography

**[[../hompage|↑ hompage hub]]**

> 폰트 패밀리 변경, 크기 조정, 한국어 줄바꿈 / descender 잘림 처리.

---

## 0. 어디서 폰트를 정하는가

| 항목 | 파일 | 토큰 |
| --- | --- | --- |
| 폰트 패밀리 (sans / mono) | `src/styles/tokens.css` | `--font-sans`, `--font-mono` |
| 크기 (xs / sm / md / lg / xl / 2xl / 3xl) | `tokens.css` | `--text-xs` ~ `--text-3xl` |
| Line-height | `tokens.css` | `--lh-tight`, `--lh-snug`, `--lh-normal` |
| Letter-spacing | `tokens.css` | `--tracking-mono`, `--tracking-label` |
| 개별 요소 폰트 | `src/styles/layout.css` | 각 컴포넌트 클래스 |

원칙: **tokens.css 만 바꾸면 사이트 전역 영향**. 개별 컴포넌트만 바꾸려면 layout.css.

---

## 1. 폰트 패밀리

### 현재
```css
--font-sans:
  "Inter", system-ui, -apple-system, "Segoe UI", Roboto,
  "Helvetica Neue", Arial, "Apple SD Gothic Neo", "Pretendard",
  "Noto Sans KR", sans-serif;
--font-mono:
  "JetBrains Mono", ui-monospace, SFMono-Regular, "SF Mono",
  Menlo, Consolas, "Liberation Mono", monospace;
```

- 영문: Inter
- 한국어: Apple SD Gothic Neo (Mac 기본) → Pretendard → Noto Sans KR fallback
- 코드/숫자: JetBrains Mono

### 한글 폰트를 Pretendard 로 강제
```css
--font-sans:
  "Pretendard Variable", "Pretendard",
  "Inter", system-ui, -apple-system, sans-serif;
```

Pretendard 가 설치돼 있어야 함. 웹폰트로 가져오려면 `index.html` 에:
```html
<link rel="stylesheet"
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@latest/dist/web/variable/pretendardvariable.css">
```

### Noto Sans KR (Google Fonts)
```css
--font-sans:
  "Noto Sans KR", -apple-system, sans-serif;
```

`index.html`:
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet"
  href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@300;400;500;600;700&display=swap">
```

### Mono 폰트 (코드/숫자) 만 바꾸기
```css
--font-mono: "Fira Code", "JetBrains Mono", monospace;
```

---

## 2. 텍스트 크기 사전

```css
--text-xs:   11px;    ← chip / eyebrow label (예: `/ BLOG · LATEST`)
--text-sm:   12.5px;  ← card-sub / mono faint / date
--text-base: 14px;    ← (현재 거의 미사용)
--text-md:   15px;    ← card-title / 본문
--text-lg:   17px;    ← Stack chip / 큰 카드 title
--text-xl:   20px;    ← project-detail h2
--text-2xl:  26px;    ← page-title (Projects / Skills 등)
--text-3xl:  34px;    ← (Home hero h1 은 별도 clamp)
```

### 어디서 어떤 크기 쓰이나
| 화면 영역 | 토큰 |
| --- | --- |
| Topbar handle (yuchan.oh) | `--text-sm` |
| Topbar status badge | `--text-xs` |
| Nav 항목 (Home, Projects ...) | `--text-sm` |
| Home hero 이름 (Oh Yuchan) | `clamp(34px, 6vw, 56px)` — 직접 정의 |
| Home hero 역할 / location | `--text-md` |
| Home hero CTA 버튼 | `--text-sm` |
| 각 카드 라벨 (`/ GITHUB`) | `--text-xs` |
| 카드 title (GitHub, Selected …) | `--text-md` |
| KPI 큰 숫자 (`42` REPOS) | `--text-xl` |
| 카드 sub 설명 | `--text-sm` |
| Page title (Projects, Skills) | `--text-2xl` |
| Page subtitle | `--text-sm` |
| Project detail H2 | `--text-xl` |

### 전체적으로 글씨 키우기
```css
--text-xs:   12px;
--text-sm:   13.5px;
--text-md:   16px;
--text-lg:   18px;
--text-xl:   22px;
--text-2xl:  28px;
```

### 전체적으로 줄이기 (밀도 ↑)
```css
--text-xs:   10px;
--text-sm:   11.5px;
--text-md:   13px;
--text-lg:   15px;
--text-xl:   18px;
--text-2xl:  22px;
```

---

## 3. Home hero 의 큰 이름 폰트

`src/styles/layout.css` 에서:
```css
.hero h1 {
  font-size: clamp(34px, 6vw, 56px);
  font-weight: 700;
  letter-spacing: -0.02em;
  line-height: 1.1;
}
```

### 더 크게 / 더 작게
```css
.hero h1 {
  font-size: clamp(28px, 5vw, 44px);  /* 더 작게 */
}
.hero h1 {
  font-size: clamp(40px, 7vw, 72px);  /* 더 크게 */
}
```

`clamp(min, ideal, max)` 는 반응형 — 화면이 작으면 min, 크면 max.

### Home 의 hero 만 별도로
Home.tsx 는 home-shell 안에서 hero 가 더 작음:
```css
.home-shell .hero h1 {
  font-size: clamp(36px, 7vw, 48px);
}
```

---

## 4. Line-height (한국어 핵심)

```css
--lh-tight:    1.25;   ← 제목 (card-title)
--lh-snug:     1.4;    ← 한 줄 텍스트
--lh-normal:   1.55;   ← 본문
```

### 한국어가 빡빡해 보이면
한국어 글자는 영문보다 위아래로 더 차지함. line-height 1.55 정도가 안전.

#### 글로벌 적용
```css
--lh-normal: 1.65;
```

#### 특정 영역만
`src/styles/layout.css` 에서 해당 클래스에:
```css
.card-sub {
  line-height: 1.6;
}
```

### 1줄 짜리 텍스트
한 줄로만 표시되는 텍스트 (예: card-title) 는 line-height 1.25 면 충분. 더 큰 폰트일수록 더 작은 line-height 가 자연스러움.

---

## 5. 한국어 줄바꿈 (word-break)

### 문제: 단어 중간에서 줄바꿈 됨
예: "스프링부트" → "스프링" / "부트" 로 잘림.

### 해결: word-break: keep-all
한국어는 어절 단위로 줄바꿈 해야 자연스러움.

#### 글로벌 적용 (모든 본문)
`src/styles/layout.css` 끝에:
```css
.card-sub,
.card-title,
p {
  word-break: keep-all;
}
```

#### 특정 클래스만
이미 적용된 곳:
- `.blog-featured-title` — Blog featured 제목
- `.blog-featured-desc` — Blog featured 설명
- `.markdown-body p, li` — README 본문

추가하고 싶으면 해당 selector 에 `word-break: keep-all;` 한 줄.

### 영문 긴 단어 잘릴 때
URL 같은 긴 영문이 카드 폭을 넘으면 `overflow-wrap: anywhere` 가 필요. 이미 markdown-body 에 적용됨.

---

## 6. 한국어 descender 잘림 (line-clamp 사용 시)

### 문제
`-webkit-line-clamp: 2` 로 2줄 자르면 마지막 줄의 한글 받침 (ㅁ ㄴ ㅏ 등 아래로 내려가는 획) 이 잘림.

### 해결
- **line-height ≥ 1.5** 확보 (descender 공간)
- `padding-bottom: 2px` 추가 (overflow:hidden 직전 마진)
- `word-break: keep-all` (단어 중간 안 끊김)

### 적용 예 (이미 layout.css 에 있음)
```css
.blog-featured-title {
  font-size: var(--text-md);
  font-weight: 600;
  white-space: normal;
  overflow: hidden;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  line-height: 1.5;     /* ← 핵심 */
  padding-bottom: 2px;  /* ← 핵심 */
  text-overflow: ellipsis;
  word-break: keep-all; /* ← 핵심 */
}
```

같은 패턴을 다른 곳에 적용하려면 위 5줄을 그대로 복사.

---

## 7. Letter-spacing (자간)

```css
--tracking-mono:   0.02em;   ← 코드/숫자 자간 (살짝 넓힘)
--tracking-label:  0.06em;   ← UPPERCASE 라벨 자간 (eyebrow `/ GITHUB`)
```

### 더 시원하게
```css
--tracking-label: 0.1em;
```

### 더 빡빡하게
```css
--tracking-label: 0.04em;
```

---

## 8. Font weight (굵기)

토큰으로 정의 안 됨 — 각 컴포넌트에 직접:
- `.brand` 600
- `.card-title` 600
- `.hero h1` 700
- `.hero-cta` 500
- 기본 400

### 전체적으로 더 굵게
`src/styles/layout.css` 에:
```css
body { font-weight: 500; }
.card-title { font-weight: 700; }
```

---

## 9. 글씨 너무 흐려 보일 때

```css
--text-muted: #b0b8b4;   /* 8a948f → 더 밝게 */
--text-faint: #80897e;   /* 5e6663 → 더 밝게 */
```

또는 카드 전체적으로:
```css
.card { color: var(--text); }       /* muted 가 아니라 text 로 */
```

---

## 10. 모노 폰트만 다른 폰트로 (KPI 숫자 등)

KPI 숫자 (`42 REPOS`) 는 mono 폰트:

### 더 두꺼운 sans 로 바꾸기
`src/styles/layout.css` 에:
```css
.kpi-value {
  font-family: var(--font-sans);
  font-weight: 700;
}
```

---

## 11. 코드 블록 (README 안)

```css
.markdown-body pre {
  padding: var(--space-4);
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  background: #07090a;
  color: #d3dcd7;
}
```

### 다른 색상 테마로
```css
.markdown-body pre {
  background: #1e1e2e;   /* Catppuccin Mocha base */
  color: #cdd6f4;
}
```

---

## 12. 변경 후 확인

```bash
cd ~/local-dev/hompage
npm run dev
# 브라우저에서 글씨 확인 → 한국어 / 영문 / 숫자 다 봄
```

특히 chip / KPI 숫자 / card-title / 본문 4 종류를 한 화면에서 같이 봐야 균형이 맞는지 보임.

---

연결: [[../hompage]] · [[tokens]] · [[colors]] · [[cards]]
