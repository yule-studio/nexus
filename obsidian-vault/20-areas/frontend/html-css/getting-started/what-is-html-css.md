---
title: "HTML/CSS 가 뭐예요? — 첫 이해"
kind: knowledge
project: frontend
status: current
tags: [frontend, html, css, beginner, introduction]
---

# HTML/CSS 가 뭐예요?

**[[../html-css|↑ HTML/CSS hub]]**

---

## 1. 웹 페이지의 3 요소

| 종류 | 역할 | 비유 |
| --- | --- | --- |
| HTML | 구조 (뼈대) | 건물의 골조 |
| CSS | 스타일 (옷) | 건물의 페인트·인테리어 |
| JavaScript | 동작 (반응) | 건물의 전기·엘리베이터 |

3 개 다 알아야 완전한 웹사이트. 처음엔 HTML + CSS 만으로도 정적 페이지 가능.

---

## 2. HTML — HyperText Markup Language

### 뭐 하는 거?
"이 글은 제목이다", "이 글자는 굵게다", "여기는 사진이다" 같은 **의미 / 구조** 를 표시.

### 가장 단순한 HTML
```html
<h1>안녕하세요</h1>
<p>여기는 첫 페이지입니다.</p>
<img src="cat.jpg" alt="고양이">
```

브라우저가 위를 읽고:
- `<h1>` 은 큰 제목으로
- `<p>` 는 단락으로
- `<img>` 는 그림으로

표시.

### "태그" 라는 단어
`<h1>` 처럼 `<` `>` 으로 감싼 게 **태그**. 시작 태그 `<h1>` + 내용 + 닫는 태그 `</h1>`.

```html
<h1>제목</h1>
└┬─┘└┬┘└┬───┘
 │   │  └─ 닫는 태그
 │   └─ 내용
 └─ 시작 태그
```

일부 태그는 닫는 짝이 없음 (self-closing):
```html
<img src="cat.jpg">     <!-- 닫는 게 없음 -->
<input type="text">
<br>
```

---

## 3. CSS — Cascading Style Sheets

### 뭐 하는 거?
HTML 요소에 **어떻게 보이는지** 를 지정. 색 / 폰트 / 여백 / 위치.

### 가장 단순한 CSS
```css
h1 {
  color: red;
  font-size: 48px;
}
```

해석: `h1` 요소를 빨강 / 48px 글씨로.

### HTML 과 연결
```html
<head>
  <style>
    h1 { color: red; }
  </style>
</head>
<body>
  <h1>안녕</h1>     <!-- 자동으로 빨강 -->
</body>
```

또는 외부 `.css` 파일로:
```html
<link rel="stylesheet" href="style.css">
```

---

## 4. 가장 작은 완전한 페이지

이 코드를 `hello.html` 로 저장 후 더블클릭 → 브라우저에서 열림.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>내 첫 페이지</title>
  <style>
    body { background: #f0f0f0; font-family: sans-serif; }
    h1 { color: #2563eb; }
    p { color: #555; line-height: 1.6; }
  </style>
</head>
<body>
  <h1>안녕하세요! 👋</h1>
  <p>이건 제가 처음 만든 웹 페이지예요.</p>
</body>
</html>
```

저장 → 더블클릭 → 회색 배경에 파란 제목.

### 각 줄 의미
- `<!DOCTYPE html>` — "이 문서는 HTML5 입니다" 라고 브라우저에 알림 (필수)
- `<html lang="ko">` — 한국어 페이지
- `<head>` — 메타 정보 (제목 / 인코딩 / 스타일). 브라우저 화면엔 안 보임
- `<meta charset="UTF-8">` — 한글 깨짐 방지 (반드시)
- `<title>` — 브라우저 탭 제목
- `<style>` — CSS 자리
- `<body>` — 실제 화면 내용

---

## 5. 왜 굳이 HTML/CSS 를 따로?

### 분리의 이유
- 같은 HTML 에 다른 CSS 만 바꾸면 다른 디자인 (테마 변경)
- HTML 은 의미, CSS 는 외관 → 검색엔진 / 스크린리더 친화

### 예: 같은 HTML 두 가지 디자인
```html
<h1>제목</h1>
<p>내용</p>
```

CSS 1:
```css
h1 { color: black; font-family: serif; }
```

CSS 2:
```css
h1 { color: red; font-family: monospace; text-transform: uppercase; }
```

HTML 그대로, CSS 만 교체.

---

## 6. 다음 단계

1. **[[first-page]]** — 첫 페이지 직접 만들기
2. **[[../html-tags-essentials]]** — 자주 쓰는 태그 20개

---

연결: [[../html-css|↑ hub]] · [[first-page]]
