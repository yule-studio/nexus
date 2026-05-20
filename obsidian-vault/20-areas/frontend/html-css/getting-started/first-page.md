---
title: "첫 HTML 페이지 만들기 — Step by step"
kind: knowledge
project: frontend
status: current
tags: [frontend, html, css, beginner, tutorial]
---

# 첫 HTML 페이지 만들기

**[[../html-css|↑ HTML/CSS hub]]**

> 아무 도구 없이 노트패드 / VSCode 만으로 첫 페이지 띄우기.

---

## 1. 준비

1. **VSCode 설치** (https://code.visualstudio.com)
2. (선택) VSCode 좌측 사이드바 → "Extensions" → "Live Server" 검색 → 설치

---

## 2. 폴더 만들기

바탕화면에 `my-first-site` 폴더 만들기.

```
my-first-site/
└─ (비어 있음)
```

---

## 3. index.html 작성

VSCode 에서 `my-first-site` 폴더 열기 → `index.html` 새 파일.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>내 첫 사이트</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <header>
    <h1>안녕, 나는 오유찬</h1>
    <p>백엔드 개발자 · DevOps 엔지니어</p>
  </header>

  <main>
    <section>
      <h2>요즘 관심</h2>
      <ul>
        <li>LLM 에이전트</li>
        <li>홈랩 자동화</li>
        <li>Kubernetes</li>
      </ul>
    </section>

    <section>
      <h2>연락</h2>
      <p>이메일: <a href="mailto:hello@example.com">hello@example.com</a></p>
      <p>GitHub: <a href="https://github.com/codwithyc" target="_blank">@codwithyc</a></p>
    </section>
  </main>

  <footer>
    <p>© 2026 — 첫 사이트</p>
  </footer>
</body>
</html>
```

저장.

---

## 4. style.css 작성

같은 폴더에 `style.css` 새 파일.

```css
/* 1. 전체 리셋 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* 2. 본문 기본 */
body {
  font-family: -apple-system, "Apple SD Gothic Neo", sans-serif;
  background: #0b0d0c;
  color: #e6ece8;
  line-height: 1.6;
  padding: 40px 20px;
}

/* 3. 컨테이너 (가운데 정렬) */
header, main, footer {
  max-width: 720px;
  margin: 0 auto;
}

/* 4. Header */
header {
  border-bottom: 1px solid #1c211f;
  padding-bottom: 24px;
  margin-bottom: 32px;
}
h1 {
  font-size: 36px;
  margin-bottom: 8px;
}
header p {
  color: #8a948f;
}

/* 5. Section */
section {
  margin-bottom: 24px;
}
h2 {
  font-size: 20px;
  color: #4ade80;
  margin-bottom: 12px;
}

ul {
  list-style: none;
}
li {
  padding: 8px 0;
  border-bottom: 1px dashed #1c211f;
}

/* 6. Link */
a {
  color: #4ade80;
  text-decoration: none;
}
a:hover {
  text-decoration: underline;
}

/* 7. Footer */
footer {
  margin-top: 48px;
  padding-top: 16px;
  border-top: 1px solid #1c211f;
  color: #5e6663;
  font-size: 12px;
  text-align: center;
}
```

저장.

---

## 5. 열기

### 방법 A — 더블클릭
`index.html` 더블클릭 → 브라우저 자동 열림.

### 방법 B — Live Server (저장 시 자동 새로고침)
VSCode 에서 `index.html` 우클릭 → "Open with Live Server".
→ 코드 수정하면 브라우저가 자동 새로고침.

---

## 6. 결과

```
┌──────────────────────────────────────┐
│  안녕, 나는 오유찬                    │
│  백엔드 개발자 · DevOps 엔지니어     │
│  ───────────────────────────────    │
│                                       │
│  요즘 관심                            │
│  · LLM 에이전트                       │
│  · 홈랩 자동화                        │
│  · Kubernetes                         │
│                                       │
│  연락                                 │
│  이메일: hello@example.com           │
│  GitHub: @codwithyc                   │
│                                       │
│  ───────────────────────────────    │
│  © 2026 — 첫 사이트                  │
└──────────────────────────────────────┘
```

---

## 7. 다음 변경 연습

### A. 색 바꾸기
`style.css` 의 `body { background: #0b0d0c }` 를:
```css
body { background: #ffffff; color: #333; }
```
→ 흰 배경 라이트 모드.

### B. 글씨 키우기
```css
h1 { font-size: 48px; }
```

### C. 항목 추가
`index.html` 의 `<ul>` 안에:
```html
<li>새 항목</li>
```

### D. 사진 추가
같은 폴더에 `me.jpg` 두고:
```html
<img src="me.jpg" alt="내 사진" style="width: 100px; border-radius: 50%;">
```

---

## 8. 함정 (자주 만나는 실수)

| 증상 | 원인 |
| --- | --- |
| 한글이 깨짐 | `<meta charset="UTF-8">` 누락 또는 파일 인코딩이 UTF-8 아님 |
| CSS 가 안 먹음 | `<link rel="stylesheet" href="style.css">` 경로 / 파일명 오타 |
| 태그가 닫혀있지 않음 → 화면 깨짐 | 모든 `<x>` 는 `</x>` 짝 필요 |
| 사진 안 보임 | `src` 경로 오타 또는 파일이 다른 폴더 |

---

## 9. 다음 단계

1. **[[../html-tags-essentials]]** — 자주 쓰는 태그 20 개
2. **[[../css-basics]]** — CSS selector / 속성 / 박스 모델
3. **[[../colors-and-typography]]** — 색 / 폰트 / 크기

---

연결: [[../html-css|↑ hub]] · [[what-is-html-css]]
