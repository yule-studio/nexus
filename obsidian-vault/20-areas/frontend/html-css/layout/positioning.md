---
title: "Position / z-index — 위치 잡기"
kind: knowledge
project: frontend
status: current
tags: [frontend, css, beginner, position, layout]
---

# Position / z-index

**[[../html-css|↑ HTML/CSS hub]]**

> 요소를 화면 어딘가에 직접 띄우거나 고정시킬 때.

---

## 1. position 5 종

```css
position: static;     /* 기본 — 흐름대로 (top/left 안 먹음) */
position: relative;   /* 본래 자리에서 상대 이동 */
position: absolute;   /* 가장 가까운 relative 부모 기준 */
position: fixed;      /* 화면 기준 (스크롤 영향 X) */
position: sticky;     /* 스크롤하다 특정 위치에 고정 */
```

### top / right / bottom / left
`position` 이 static 이 아니면 작동:
```css
.box {
  position: absolute;
  top: 10px;
  left: 20px;
}
```

---

## 2. relative — 본래 자리 + 약간 이동

```css
.box {
  position: relative;
  top: 10px;       /* 본래 자리에서 10px 아래로 */
  left: 5px;       /* 5px 오른쪽으로 */
}
```

다른 요소엔 영향 X (자리 차지 유지, 시각적으로만 이동).

### 주 용도
**absolute 자식의 기준점 만들기**:
```css
.parent {
  position: relative;     /* 기준점 */
}
.child {
  position: absolute;     /* parent 기준 */
  top: 0;
  right: 0;
}
```

---

## 3. absolute — 부모 기준 절대 위치

```css
.parent {
  position: relative;
  width: 300px;
  height: 200px;
}
.badge {
  position: absolute;
  top: 8px;
  right: 8px;
}
```

→ badge 가 parent 박스의 우상단에.

### 주의
- 가장 가까운 `position: relative/absolute/fixed/sticky` 인 조상 기준
- 조상이 다 static 이면 → `<body>` (페이지 전체) 기준
- 본래 자리 차지 X (다른 요소들이 그 자리 흘러 들어감)

### 예 — 카드 우상단 NEW 배지
```html
<div class="card">
  <span class="badge">NEW</span>
  <h2>제목</h2>
</div>
```
```css
.card {
  position: relative;
  padding: 16px;
}
.badge {
  position: absolute;
  top: 8px;
  right: 8px;
  background: red;
  color: white;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 11px;
}
```

---

## 4. fixed — 화면 기준 고정

```css
.fab {
  position: fixed;
  bottom: 20px;
  right: 20px;
}
```

→ 스크롤 해도 화면 우하단에 고정 (Floating Action Button 같은).

### 헤더 상단 고정
```css
.header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 60px;
  background: white;
  z-index: 100;       /* 다른 요소 위에 */
}

body {
  padding-top: 60px;   /* header 가 가린 만큼 본문 위쪽 padding */
}
```

---

## 5. sticky — 스크롤 시 고정

```css
.nav {
  position: sticky;
  top: 0;       /* 화면 상단 닿으면 고정 시작 */
  background: white;
  z-index: 50;
}
```

→ 평소엔 흐름대로, 스크롤 해서 상단 닿으면 거기 멈춤.

### 차이
- `fixed` — 항상 화면 기준
- `sticky` — 평소엔 일반 흐름, 특정 위치 도달하면 고정

본인 hompage 의 Topbar / Nav 가 `position: sticky` 사용.

---

## 6. z-index — 위/아래 쌓임 순서

```css
.a { z-index: 1; }
.b { z-index: 10; }    /* b 가 a 위에 */
.c { z-index: -1; }    /* 아래로 */
```

### 동작 규칙
- `position` 이 static 이 아닐 때만 적용
- 숫자가 클 수록 위
- 같은 z-index 면 HTML 순서대로 (뒤에 있는 게 위)

### 자주 쓰는 값
- `z-index: 0` — 기본 평면
- `z-index: 10` — 카드 hover 효과
- `z-index: 50` — sticky nav
- `z-index: 100` — fixed header
- `z-index: 1000` — modal / dialog
- `z-index: 9999` — tooltip / notification

지나치게 큰 값 (9999, 99999) 은 디버깅 어려워짐.

---

## 7. 가운데 정렬 (position 활용)

### Flexbox 가 가장 쉽지만, position 으로도:
```css
.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
```

설명: 상단 50% / 좌측 50% 로 이동 (좌상단이) → translate 로 자기 크기의 절반만큼 다시 위/왼쪽으로 → 가운데.

---

## 8. 자주 만드는 패턴

### A. Floating button (FAB)
```css
.fab {
  position: fixed;
  bottom: 24px;
  right: 24px;
  width: 56px;
  height: 56px;
  border-radius: 50%;
  background: #4ade80;
  color: white;
  border: none;
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
  cursor: pointer;
  z-index: 100;
}
```

### B. Modal (다이얼로그)
```css
.modal-backdrop {
  position: fixed;
  inset: 0;                    /* top/right/bottom/left = 0 */
  background: rgba(0,0,0,0.5);
  z-index: 1000;
}
.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: white;
  padding: 24px;
  border-radius: 8px;
  z-index: 1001;
}
```

### C. Sticky 헤더 + sticky 사이드 nav
```css
.header {
  position: sticky;
  top: 0;
  z-index: 50;
}
.sidebar {
  position: sticky;
  top: 60px;     /* header 높이 만큼 아래 */
  height: calc(100vh - 60px);
  overflow-y: auto;
}
```

### D. Tooltip (요소 위 떠 있는 작은 박스)
```html
<button class="tip-target">
  마우스 올려봐
  <span class="tip">설명</span>
</button>
```
```css
.tip-target {
  position: relative;
}
.tip {
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  background: black;
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  white-space: nowrap;
  visibility: hidden;
  opacity: 0;
  transition: opacity 0.15s;
}
.tip-target:hover .tip {
  visibility: visible;
  opacity: 1;
}
```

### E. Cover overlay (사진 위 텍스트)
```html
<div class="cover">
  <img src="hero.jpg">
  <div class="cover-text">제목</div>
</div>
```
```css
.cover {
  position: relative;
}
.cover img {
  width: 100%;
}
.cover-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-size: 32px;
}
```

---

## 9. inset 단축

```css
inset: 0;          /* top: 0; right: 0; bottom: 0; left: 0 */
inset: 10px;       /* 4 방향 10px */
inset: 10px 20px;  /* 위아래 10, 좌우 20 */
```

---

## 10. 함정

| 증상 | 원인 |
| --- | --- |
| `top/left` 안 먹음 | `position: static` (기본) 이라서. `relative` 같은 거 추가 |
| absolute 가 엉뚱한 데로 | 부모에 `position: relative` 안 함 |
| sticky 안 됨 | 부모에 `overflow: hidden` 있음 |
| z-index 안 먹음 | `position` 이 static 임. 또는 stacking context 분리 |
| modal 이 다른 요소 아래로 깔림 | z-index 부족 |

---

## 11. Stacking context

새 stacking 영역이 생기는 조건:
- `position` + `z-index` 설정
- `opacity < 1`
- `transform`, `filter`, `backdrop-filter`
- `isolation: isolate`

부모가 stacking context 면 자식의 z-index 는 그 안에서만 비교. 모달이 안 떠 보이면 부모 중에 `opacity`, `transform` 있는지 의심.

---

## 12. 다음 단계

1. **[[../responsive/responsive-design]]** — 반응형
2. **[[../forms/forms-basics]]** — 폼 / 입력
3. **[[../semantic-and-accessibility]]** — 의미 / 접근성

---

연결: [[../html-css|↑ hub]] · [[flexbox]] · [[grid]]
