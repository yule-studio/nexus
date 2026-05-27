---
title: "troubleshooting · z-index 가 안 먹음 — stacking context"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, css, z-index, stacking-context]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# z-index 가 안 먹음 — modal 이 element 뒤로 / dropdown 가 잘림

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- `z-index: 9999` 한 modal 이 다른 element 뒤로 가려짐
- dropdown 메뉴가 parent overflow 안에 갇힘
- z-index 를 더 크게 해도 효과 없음

## 원인 (root cause)

z-index 는 **같은 stacking context 안에서만** 비교. parent 가 새 stacking context 만들면 child 의 z-index 가 아무리 커도 parent context 안에서만 작동.

새 stacking context 생성 조건:
- `position: relative / absolute / fixed / sticky` + `z-index !== auto`
- `opacity < 1`
- `transform`, `filter`, `perspective`, `clip-path`
- `will-change` 가 위 속성
- `isolation: isolate`
- flex / grid item + `z-index !== auto`

```html
<div style="opacity: 0.99">              <!-- 새 stacking context -->
  <div class="modal" style="z-index: 9999"></div>   <!-- 이 9999 는 부모 안에서만 -->
</div>

<div style="z-index: 1"></div>   <!-- modal 보다 위로 -->
```

## 해결

### Portal — DOM 위치 변경 (React)
```tsx
import { createPortal } from 'react-dom';

function Modal({ children }) {
  return createPortal(
    <div className="modal">{children}</div>,
    document.body   // ← body 직속 → root stacking context
  );
}
```

dropdown / toast / modal 등은 portal 표준.

### Stacking context 의도적 제어
```css
.app {
  isolation: isolate;   /* 명시적 새 context — 안쪽은 격리 */
}

.modal-root {
  position: fixed;
  z-index: 1000;
  isolation: isolate;
}
```

### z-index 스케일 정의
```css
:root {
  --z-base: 0;
  --z-dropdown: 1000;
  --z-modal: 2000;
  --z-toast: 3000;
  --z-tooltip: 4000;
}

.modal { z-index: var(--z-modal); }
```

→ 9999 / 999999 같은 magic number 회피.

### Parent overflow 제거 (dropdown 잘림)
```css
.dropdown-parent {
  overflow: visible;   /* hidden 이면 child dropdown 가 잘림 */
}
```

또는 portal 로 분리.

## 진단

```
# Chrome DevTools
1. Elements tab → 컴포넌트 선택
2. Computed tab → "stacking context" 확인
3. 또는 3D View (Layers panel) — 시각화

# 다른 도구
- "CSS Stacking Context Inspector" Chrome extension
```

## 재발 방지

- z-index 는 design token (`--z-modal` 등) 으로 통일
- modal / dropdown / toast 는 portal 표준
- ESLint stylelint rule: z-index 9999 같은 magic number 경고

## 관련

- [[frontend]]
