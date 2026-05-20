---
title: "hompage styling — 반응형 (모바일 / 태블릿 / 데스크탑)"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:30:00+09:00
tags: [project, hompage, styling, responsive, media-query]
home_hub: hompage
---

# 반응형

**[[../hompage|↑ hompage hub]]**

> 화면 폭별로 무엇이 어떻게 달라지는지 + 자기 페이지 추가 시 반응형 룰 패턴.

---

## 1. Breakpoint

`src/styles/layout.css` 에 정의된 media query:

| 폭 | 별칭 | 효과 |
| --- | --- | --- |
| ≤ 640px | Mobile | grid 4-col, 카드 전부 full width, topbar status 숨김 |
| 641px ~ 960px | Tablet | grid 8-col, sm/md card 자동 축소 |
| ≥ 961px | Desktop | grid 12-col, 모든 span 활성 |

---

## 2. Mobile (≤ 640px)

```css
@media (max-width: 640px) {
  .topbar-status { display: none; }
  .theme-toggle span { display: none; }
  .theme-toggle { padding: 6px; }

  .grid {
    grid-template-columns: repeat(4, minmax(0, 1fr));
    grid-auto-rows: 88px;
  }
  .card[data-span] {
    grid-column: span 4;
    grid-row: span 3;
  }

  .project-detail-head { grid-template-columns: 1fr; }
  .project-actions { justify-content: flex-start; }
}
```

### 효과
- Topbar 의 status badge / theme toggle 글자 숨김 (아이콘만)
- grid 가 4-col 로 축소
- 모든 카드가 full-width (4col = 100%)
- Project detail 의 head 가 한 칼럼으로

---

## 3. Tablet (≤ 960px)

```css
@media (max-width: 960px) {
  .grid {
    grid-template-columns: repeat(8, minmax(0, 1fr));
  }
  .card[data-span="sm"]   { grid-column: span 4; }
  .card[data-span="md"]   { grid-column: span 4; }
  .card[data-span="lg"]   { grid-column: span 8; }
  .card[data-span="xl"]   { grid-column: span 8; }
  .card[data-span="tall"] { grid-column: span 4; }
  .card[data-span="wide"] { grid-column: span 8; }
  .card[data-span="hero"] { grid-column: span 4; }
}
```

### 효과
- grid 가 8-col
- sm / md / tall / hero → 4col (한 줄에 2개)
- lg / xl / wide → 8col (한 줄에 1개 full)

---

## 4. Desktop (≥ 961px)

별도 media query 없음 — 기본 룰 적용.
- grid 12-col
- sm 4col / md 8col / xl 12col / hero 4col×6row

---

## 5. 새 페이지 / 컴포넌트 반응형 만들기

### 패턴 1 — 칼럼 수 조정
```css
.my-grid {
  display: grid;
  gap: var(--space-3);
  grid-template-columns: repeat(3, 1fr);   /* 데스크탑 */
}
@media (max-width: 960px) {
  .my-grid { grid-template-columns: repeat(2, 1fr); }   /* 태블릿 */
}
@media (max-width: 640px) {
  .my-grid { grid-template-columns: 1fr; }              /* 모바일 */
}
```

### 패턴 2 — 폰트 크기 줄이기
```css
.my-title { font-size: 24px; }
@media (max-width: 640px) {
  .my-title { font-size: 18px; }
}
```

또는 `clamp` 한 줄로:
```css
.my-title { font-size: clamp(18px, 4vw, 24px); }
```

### 패턴 3 — 모바일에서 요소 숨김
```css
@media (max-width: 640px) {
  .my-side-panel { display: none; }
}
```

### 패턴 4 — 모바일에서 직선 → 세로
```css
.my-row {
  display: flex;
  gap: var(--space-3);
}
@media (max-width: 640px) {
  .my-row { flex-direction: column; }
}
```

---

## 6. Container padding 반응형

```css
--container-pad: clamp(16px, 4vw, 32px);
```
이미 반응형. 작은 화면 16px, 큰 화면 32px, 중간은 4vw.

### 모바일에서 padding 더 작게
```css
--container-pad: clamp(8px, 3vw, 32px);
```

---

## 7. Page-shell / Home-shell 반응형

이미 `width: min(100%, 720px)` 형태라 자동 반응형 (작은 화면이면 100% 사용).

### 모바일에서만 더 좁게 (양쪽 여백)
```css
@media (max-width: 640px) {
  .page-shell, .home-shell {
    width: min(100% - 16px, 720px);
  }
}
```

---

## 8. Topbar 의 반응형

```css
.topbar-inner {
  display: flex; align-items: center;
  gap: var(--space-3);
  height: var(--topbar-h);
}
```

모바일에선 status / theme toggle 글자가 숨겨져서 공간 확보. brand handle 만 보임.

### 모바일에서 topbar 더 압축
```css
@media (max-width: 640px) {
  --topbar-h: 48px;
  .brand-handle { font-size: 11px; }
}
```

---

## 9. Nav (목차) 반응형

```css
.nav-pill {
  max-width: 100%;
  overflow-x: auto;
  scrollbar-width: none;
}
```

이미 반응형 — 작은 화면에선 nav 가 가로 스크롤. 항목이 많으면 스크롤로 접근.

### 모바일에서 nav 숨기고 햄버거 메뉴 (고급)
별도 컴포넌트 추가 필요. agent 에게 부탁 권장.

---

## 10. Hero h1 (Home 이름) 반응형

```css
.hero h1 {
  font-size: clamp(34px, 6vw, 56px);
}
.home-shell .hero h1 {
  font-size: clamp(36px, 7vw, 48px);
}
```

`clamp(min, ideal, max)` — 화면 따라 자동.

### Hero 모바일에서 한 줄로 안 깨지게
```css
@media (max-width: 640px) {
  .hero h1 { font-size: clamp(28px, 8vw, 36px); }
}
```

---

## 11. Card-list (모바일에서 더 작게)

```css
@media (max-width: 640px) {
  .card-list li {
    padding-block: 6px;
    font-size: var(--text-xs);
  }
}
```

---

## 12. 테스트 — 반응형 확인 방법

### 브라우저
1. `npm run dev`
2. Chrome / Safari 검사도구 (Cmd+Option+I)
3. 우상단 디바이스 토글 (Cmd+Shift+M) → iPhone / iPad 시뮬레이션

### 실 디바이스
1. dev 서버 시작 시 `Network` URL 도 표시됨 (예: `http://192.168.x.x:5173`)
2. 같은 wifi 의 폰/태블릿에서 그 URL 열기
3. 실 디바이스에서 직접 확인

### 자주 깨지는 곳
- 카드 안 긴 영문 단어 → `overflow-wrap: anywhere` 필요
- 한국어 단어 잘림 → `word-break: keep-all`
- 가로 스크롤 발생 → `min-width: 0` + `overflow: hidden` 빠진 곳 찾기

---

## 13. 가로 스크롤 (overflow) 디버깅

가로 스크롤 생기면:
```js
// 콘솔에서 실행
[...document.querySelectorAll('*')].forEach(el => {
  if (el.offsetWidth > document.documentElement.offsetWidth) {
    console.log('overflowing:', el);
  }
});
```
→ 폭 넘는 element 찾아냄.

보통:
- table / pre (코드 블록) — 이미 `overflow-x: auto` 적용
- 긴 URL / 외부 콘텐츠 → `word-break` / `overflow-wrap`

---

연결: [[../hompage]] · [[tokens]] · [[layout-grid]] · [[cards]]
