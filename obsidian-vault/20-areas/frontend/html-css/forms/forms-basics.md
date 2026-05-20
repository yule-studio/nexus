---
title: "Forms 기초 — input / button / form"
kind: knowledge
project: frontend
status: current
tags: [frontend, html, css, beginner, forms]
---

# Forms 기초

**[[../html-css|↑ HTML/CSS hub]]**

---

## 1. `<form>`

```html
<form action="/submit" method="POST">
  <input type="text" name="username">
  <button type="submit">로그인</button>
</form>
```

- `action` — 제출할 URL (서버)
- `method` — `GET` / `POST`
- 버튼 클릭 시 자동 제출 (페이지 새로 고침)

### JS / React 에서는 보통 `action` 안 쓰고 `onSubmit` 으로 직접 처리.

---

## 2. `<input>` 종류

```html
<input type="text">         <!-- 일반 텍스트 -->
<input type="email">         <!-- 이메일 형식 검증 -->
<input type="password">      <!-- 점으로 가려짐 -->
<input type="number">        <!-- 숫자 (+ 스피너) -->
<input type="tel">           <!-- 전화 (모바일 키패드) -->
<input type="url">           <!-- URL -->
<input type="search">        <!-- 검색 (x 버튼 자동) -->
<input type="date">          <!-- 달력 picker -->
<input type="time">          <!-- 시간 picker -->
<input type="datetime-local"><!-- 날짜 + 시간 -->
<input type="color">         <!-- 색 picker -->
<input type="range" min="0" max="100">  <!-- 슬라이더 -->
<input type="file">          <!-- 파일 업로드 -->
<input type="checkbox">      <!-- 체크박스 -->
<input type="radio">         <!-- 라디오 (하나만 선택) -->
<input type="hidden">        <!-- 안 보이는 데이터 -->
```

---

## 3. 자주 쓰는 속성

```html
<input
  type="text"
  name="username"             <!-- 서버로 전송될 이름 -->
  id="username"               <!-- label 과 짝 -->
  value="홍길동"               <!-- 초기값 -->
  placeholder="이름 입력"      <!-- 비어 있을 때 회색 글 -->
  required                    <!-- 필수 -->
  disabled                    <!-- 비활성 -->
  readonly                    <!-- 읽기 전용 -->
  maxlength="20"              <!-- 최대 글자 수 -->
  minlength="2"               <!-- 최소 글자 수 -->
  pattern="[A-Za-z]+"         <!-- 정규식 검증 -->
  autocomplete="username"     <!-- 자동완성 힌트 -->
  autofocus                   <!-- 페이지 열림 시 자동 포커스 -->
>
```

---

## 4. `<label>` — input 과 짝

### 방법 1 — for 속성
```html
<label for="email">이메일</label>
<input type="email" id="email">
```

### 방법 2 — 감싸기
```html
<label>
  이메일
  <input type="email">
</label>
```

라벨 클릭 시 input 으로 포커스. **접근성에 매우 중요**. 항상 label 사용.

---

## 5. `<textarea>` — 여러 줄

```html
<textarea
  rows="5"          <!-- 표시할 줄 수 -->
  cols="40"         <!-- 표시할 폭 -->
  placeholder="긴 내용"
  maxlength="500"
></textarea>
```

CSS 로 크기 조절 권장 (`rows` 보다):
```css
textarea {
  width: 100%;
  min-height: 120px;
  resize: vertical;     /* 세로 크기 조절만 허용 */
}
```

---

## 6. `<select>` — 드롭다운

```html
<select name="country">
  <option value="">선택하세요</option>
  <option value="kr">한국</option>
  <option value="jp" selected>일본</option>
  <option value="us">미국</option>
</select>
```

### 그룹
```html
<select>
  <optgroup label="아시아">
    <option>한국</option>
    <option>일본</option>
  </optgroup>
  <optgroup label="유럽">
    <option>프랑스</option>
  </optgroup>
</select>
```

### Multiple (여러 개 선택)
```html
<select multiple>
  <option>A</option>
  <option>B</option>
</select>
```

---

## 7. Checkbox / Radio

### Checkbox (여러 개 선택 가능)
```html
<label>
  <input type="checkbox" name="hobby" value="reading"> 독서
</label>
<label>
  <input type="checkbox" name="hobby" value="coding"> 코딩
</label>
```

### Radio (하나만 선택)
```html
<label>
  <input type="radio" name="gender" value="m"> 남자
</label>
<label>
  <input type="radio" name="gender" value="f"> 여자
</label>
```

같은 `name` 이어야 그룹으로 묶임. radio 는 한 그룹에서 하나만 선택.

---

## 8. `<button>` 3 종

```html
<button type="submit">제출</button>   <!-- form 안에서 클릭 시 제출 -->
<button type="reset">초기화</button>  <!-- form 안 모든 값 초기화 -->
<button type="button">일반</button>   <!-- 아무 동작 안 함 (JS 로 처리) -->
```

`<form>` 안에서 type 안 적으면 기본이 `submit` 임 (의도치 않게 form 제출됨).
**반드시 type 명시**.

---

## 9. fieldset / legend (관련 input 묶음)

```html
<fieldset>
  <legend>주소</legend>
  <input type="text" placeholder="도로명">
  <input type="text" placeholder="상세 주소">
</fieldset>
```

→ 외곽선 + legend 라벨. 접근성 좋음.

---

## 10. CSS 스타일링

### 기본 reset
```css
input, textarea, select, button {
  font: inherit;       /* 부모 폰트 따라감 */
  color: inherit;
  background: none;
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 8px 12px;
}
```

### Input 한 줄
```css
.input {
  width: 100%;
  padding: 10px 14px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  font-size: 14px;
  transition: border-color 0.15s;
}
.input:focus {
  outline: none;
  border-color: #4ade80;
  box-shadow: 0 0 0 3px rgba(74, 222, 128, 0.2);
}
```

### Button primary
```css
.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  background: #4ade80;
  color: white;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s;
}
.btn:hover { background: #22c55e; }
.btn:disabled {
  background: #d1d5db;
  cursor: not-allowed;
}
```

### Button 변형
```css
.btn-secondary {
  background: transparent;
  color: #4ade80;
  border: 1px solid #4ade80;
}
.btn-secondary:hover {
  background: rgba(74, 222, 128, 0.1);
}
```

### Checkbox / Radio (커스텀)
브라우저 기본 디자인은 못생김. accent-color 한 줄로 색만:
```css
input[type="checkbox"],
input[type="radio"] {
  accent-color: #4ade80;
}
```

---

## 11. 검증 (validation)

### HTML5 기본 검증
```html
<input type="email" required>
<input type="text" minlength="3" maxlength="20">
<input type="number" min="0" max="100">
<input type="text" pattern="[A-Za-z]+" title="영문만">
```

제출 시 자동 검증. 실패 시 브라우저 에러 메시지.

### CSS 검증 상태
```css
input:valid { border-color: green; }
input:invalid { border-color: red; }
input:required { background: #fffbea; }
```

### JS 로 메시지 커스텀
```javascript
input.setCustomValidity("3 글자 이상 입력하세요");
```

---

## 12. 자주 만드는 폼 패턴

### 로그인 폼
```html
<form class="login-form" onsubmit="handleSubmit(event)">
  <label>
    이메일
    <input type="email" name="email" required>
  </label>

  <label>
    비밀번호
    <input type="password" name="password" required minlength="8">
  </label>

  <label class="checkbox-row">
    <input type="checkbox" name="remember">
    로그인 유지
  </label>

  <button type="submit" class="btn">로그인</button>
</form>
```

```css
.login-form {
  display: flex;
  flex-direction: column;
  gap: 16px;
  max-width: 320px;
}
.login-form label {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 14px;
}
.checkbox-row {
  flex-direction: row !important;
  align-items: center;
}
```

### 검색 바
```html
<form class="search">
  <input type="search" placeholder="검색..." name="q">
  <button type="submit">🔍</button>
</form>
```

```css
.search {
  display: flex;
  border: 1px solid #ccc;
  border-radius: 999px;
  overflow: hidden;
}
.search input {
  flex: 1;
  border: none;
  padding: 10px 16px;
  outline: none;
}
.search button {
  border: none;
  background: #4ade80;
  color: white;
  padding: 0 16px;
  cursor: pointer;
}
```

---

## 13. 폼 제출 (JS / fetch)

```html
<form id="my-form">
  <input name="email">
  <button>제출</button>
</form>

<script>
  document.getElementById("my-form").addEventListener("submit", async (e) => {
    e.preventDefault();    // 페이지 새로고침 방지

    const formData = new FormData(e.target);
    const data = Object.fromEntries(formData);

    const res = await fetch("/api/submit", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });
    const json = await res.json();
    console.log(json);
  });
</script>
```

자세히는 React 의 [[../../react/forms/forms]] 참고.

---

## 14. 함정

| 증상 | 원인 |
| --- | --- |
| `<button>` 클릭 시 페이지 새로 고침 | `type="button"` 안 적음 → 기본 submit |
| placeholder 가 label 처럼 보임 | label 도 같이 적어야 접근성 OK |
| 모바일에서 키패드가 일반 | `type` 을 `email` / `tel` / `number` 로 |
| input 글자 안 보임 | color 가 부모랑 다름. `color: inherit` |
| autofocus 가 모바일에서 키패드 안 뜸 | iOS Safari 정책. user 클릭 후 띄우기 |

---

## 15. 다음 단계

1. **[[../semantic-and-accessibility]]** — 의미 / 접근성
2. **[[../../javascript/javascript]]** — JS 로 동적 동작
3. **[[../../react/forms/forms]]** — React 의 controlled form

---

연결: [[../html-css|↑ hub]] · [[../html-tags-essentials]]
