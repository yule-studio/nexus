---
title: "HTML 태그 — 자주 쓰는 20 개"
kind: knowledge
project: frontend
status: current
tags: [frontend, html, beginner, tags, reference]
---

# HTML 태그 — 자주 쓰는 20 개

**[[html-css|↑ HTML/CSS hub]]**

> 외울 필요 X. 필요할 때 여기 와서 복사.

---

## 1. 텍스트

### `<h1>` ~ `<h6>` — 제목 (Headings)
```html
<h1>가장 큰 제목 (페이지 1개에 1번만)</h1>
<h2>섹션 제목</h2>
<h3>서브 섹션</h3>
<h4>작은 제목</h4>
```

### `<p>` — 단락 (paragraph)
```html
<p>이건 한 단락입니다. 여러 줄이라도 한 단락.</p>
<p>다음 단락.</p>
```

### `<strong>` / `<em>` — 강조
```html
<p>이건 <strong>굵게 (중요)</strong> 표시.</p>
<p>이건 <em>기울임 (강조)</em> 표시.</p>
```

### `<br>` — 줄바꿈
```html
<p>한 줄.<br>다음 줄.</p>
```

### `<hr>` — 가로줄 (구분선)
```html
<p>위 단락</p>
<hr>
<p>아래 단락</p>
```

---

## 2. 링크와 그림

### `<a>` — 링크
```html
<a href="https://google.com">구글로</a>
<a href="https://google.com" target="_blank" rel="noreferrer">새 탭에서 열기</a>
<a href="mailto:hello@example.com">메일 보내기</a>
<a href="tel:+821012345678">전화 걸기</a>
<a href="#section2">같은 페이지의 #section2 로 점프</a>
```

### `<img>` — 그림
```html
<img src="cat.jpg" alt="고양이 사진">
<img src="https://example.com/cat.jpg" alt="외부 사진" width="200">
```

- `src` — 사진 경로 (필수)
- `alt` — 대체 텍스트 (필수, 접근성)

---

## 3. 리스트

### `<ul>` + `<li>` — 순서 없는 목록 (bullet)
```html
<ul>
  <li>Python</li>
  <li>TypeScript</li>
  <li>Go</li>
</ul>
```
화면: `• Python`

### `<ol>` + `<li>` — 순서 있는 목록 (번호)
```html
<ol>
  <li>커피 마시기</li>
  <li>코딩</li>
  <li>잠</li>
</ol>
```
화면: `1. 커피 마시기`

---

## 4. 컨테이너

### `<div>` — 의미 없는 박스 (가장 자주 씀)
```html
<div class="card">
  <h2>제목</h2>
  <p>내용</p>
</div>
```

### `<span>` — 인라인 (한 줄 안) 박스
```html
<p>가격은 <span class="price">$9.99</span> 입니다.</p>
```

`<div>` 는 박스 (줄 바꿈), `<span>` 은 한 줄 안 일부.

---

## 5. 의미 있는 컨테이너 (semantic)

### `<header>` — 페이지 / 섹션 머리
```html
<header>
  <h1>사이트 제목</h1>
</header>
```

### `<nav>` — 네비게이션 (메뉴)
```html
<nav>
  <a href="/">Home</a>
  <a href="/about">About</a>
</nav>
```

### `<main>` — 페이지 본문 (1번만)
```html
<main>
  <h2>본문</h2>
</main>
```

### `<section>` — 의미 있는 섹션
```html
<section>
  <h2>요즘 관심</h2>
  <p>...</p>
</section>
```

### `<article>` — 독립 콘텐츠 (블로그 글 등)
```html
<article>
  <h2>블로그 제목</h2>
  <p>본문...</p>
</article>
```

### `<aside>` — 사이드 (보조 정보)
```html
<aside>
  <h3>관련 글</h3>
</aside>
```

### `<footer>` — 페이지 / 섹션 꼬리
```html
<footer>
  <p>© 2026</p>
</footer>
```

→ 자세히 [[semantic-and-accessibility]]

---

## 6. 폼 (입력)

### `<input>` — 한 줄 입력
```html
<input type="text" placeholder="이름">
<input type="email" placeholder="이메일">
<input type="password" placeholder="비밀번호">
<input type="number" placeholder="나이">
<input type="date">
<input type="checkbox">
<input type="radio" name="color" value="red"> 빨강
<input type="radio" name="color" value="blue"> 파랑
```

### `<textarea>` — 여러 줄 입력
```html
<textarea rows="5" placeholder="여러 줄"></textarea>
```

### `<button>` — 버튼
```html
<button type="button">눌러</button>
<button type="submit">제출</button>
```

### `<select>` + `<option>` — 드롭다운
```html
<select>
  <option value="kr">한국</option>
  <option value="jp">일본</option>
</select>
```

### `<label>` — 라벨 (input 과 짝)
```html
<label for="name">이름</label>
<input type="text" id="name">
```

### `<form>` — 폼 묶음
```html
<form action="/submit" method="POST">
  <input type="text" name="username">
  <button type="submit">로그인</button>
</form>
```

→ 자세히 [[forms/forms-basics]]

---

## 7. 표

### `<table>` + `<tr>` + `<td>`
```html
<table>
  <thead>
    <tr><th>이름</th><th>나이</th></tr>
  </thead>
  <tbody>
    <tr><td>홍길동</td><td>25</td></tr>
    <tr><td>이순신</td><td>30</td></tr>
  </tbody>
</table>
```

- `<thead>` — 헤더, `<tbody>` — 본문
- `<tr>` — 한 행 (row), `<th>` — 헤더 셀, `<td>` — 데이터 셀

---

## 8. 미디어

### `<video>`
```html
<video src="movie.mp4" controls width="400"></video>
```

### `<audio>`
```html
<audio src="sound.mp3" controls></audio>
```

### `<iframe>` — 다른 페이지 끼워넣기
```html
<iframe src="https://example.com" width="400" height="300"></iframe>
```

---

## 9. 속성 (attributes)

태그에 추가 정보. `name="value"` 형태.

| 속성 | 의미 | 예 |
| --- | --- | --- |
| `id` | 페이지에서 유일한 ID | `<div id="header">` |
| `class` | CSS 적용용 그룹 | `<div class="card">` |
| `style` | 인라인 CSS | `<div style="color: red">` |
| `title` | 마우스 올림 시 툴팁 | `<a title="홈으로">` |
| `data-*` | 사용자 정의 데이터 | `<div data-id="42">` |

### id vs class
- **id** — 페이지에 1개만. 중복 금지. CSS `#header { ... }`
- **class** — 여러 개 OK. CSS `.card { ... }`. 여러 클래스 가능: `class="card primary large"`

---

## 10. 주석

```html
<!-- 이건 주석. 화면에 안 보임. -->
<p>이건 보임</p>
```

---

## 11. 의미 / 구조 vs 외관

옛 방식 (피하기):
```html
<font color="red" size="5">제목</font>     <!-- font 태그는 deprecated -->
<center>가운데</center>                     <!-- center 태그도 -->
```

새 방식:
```html
<h1 style="color: red; font-size: 28px; text-align: center;">제목</h1>
```

또는 더 좋은 방식 (CSS 분리):
```html
<h1 class="big-red-center">제목</h1>
```
```css
.big-red-center {
  color: red;
  font-size: 28px;
  text-align: center;
}
```

---

## 12. 다음 단계

1. **[[css-basics]]** — CSS selector / 속성 / 박스 모델
2. **[[colors-and-typography]]** — 색 / 폰트
3. **[[layout/flexbox]]** — 가로/세로 배치

---

연결: [[html-css|↑ hub]] · [[getting-started/first-page]] · [[css-basics]]
