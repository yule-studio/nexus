---
title: "troubleshooting · CSS Grid blow-out (min-width: auto)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, css, grid, flexbox]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# CSS Grid blow-out — grid item 이 grid 밖으로 나감

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- `grid-template-columns: 1fr 1fr` 인데 child 의 긴 텍스트 / 이미지가 column 넘어 grid 자체 widen
- 같은 패턴 flex 에서도 발생 (`flex: 1` item 이 contents 따라 widen)
- overflow content 가 다음 row 로 가지 않고 horizontal scroll 생김

## 원인

CSS Grid / Flex item 의 default `min-width: auto` = content 의 intrinsic minimum width. 긴 텍스트 한 줄 (`100자` URL 등) 이면 grid track 강제 확장.

```html
<div style="display: grid; grid-template-columns: 1fr 1fr; width: 400px;">
  <div>https://very-very-very-long-url-that-cannot-wrap.com/path</div>
  <div>second column</div>
</div>
<!-- → grid 가 600px+ widen, 첫 column 가 다 차지 -->
```

## 해결

### `min-width: 0` 명시
```css
.grid-item {
  min-width: 0;   /* 또는 0px */
}

/* 또는 모든 grid item */
.grid > * {
  min-width: 0;
  min-height: 0;   /* grid row 에도 같은 문제 */
}
```

### `overflow: hidden` (text 자름)
```css
.grid-item {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

### `word-break` / `overflow-wrap`
```css
.grid-item {
  word-break: break-word;       /* 영어 단어 breaking */
  overflow-wrap: anywhere;       /* 모든 곳 break */
}
```

### Grid track 강제 max
```css
.grid {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
  /* minmax(0, 1fr) 가 핵심 — min auto 무시 */
}
```

→ 가장 깨끗한 해결책.

## Flex 동일 패턴

```css
.flex-container {
  display: flex;
}

.flex-item {
  flex: 1;
  min-width: 0;   /* ← 필수 */
}
```

특히 `<input>` 이 flex item 일 때 — `min-width: 0` 없으면 input default 200px 가 강제됨.

## 진단

```
# DevTools
1. Elements → grid item 선택
2. Computed → "min-width" 가 "auto" 면 문제
3. Layout panel → grid track 시각화 (Chrome 가 grid overlay 제공)
```

## 재발 방지

- 전역 reset: `.grid > *, .flex > * { min-width: 0; min-height: 0; }`
- design system 컴포넌트는 항상 `minmax(0, 1fr)` 패턴
- e2e visual regression — 긴 텍스트 fixture 로 layout 검증

## 관련

- [[frontend]] / [[ios-safari-100vh]]
