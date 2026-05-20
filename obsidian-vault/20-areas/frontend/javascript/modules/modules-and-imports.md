---
title: "JS 모듈 — import / export"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, beginner, modules, import]
---

# 모듈 — import / export

**[[../javascript|↑ JavaScript hub]]**

> 코드를 여러 파일로 분리 + 서로 가져다 쓰기.

---

## 1. 왜 모듈?

### 한 파일에 다 — 깨짐
- 파일 수천 줄 → 읽기 / 디버깅 불가
- 같은 변수 이름 충돌
- 의존 관계 안 보임

### 모듈 — 파일별로
```
src/
├─ utils/
│  └─ date.js          ← 날짜 함수
├─ components/
│  └─ Button.js        ← 버튼 컴포넌트
└─ App.js              ← date, Button 사용
```

---

## 2. ES Module 기본 — export / import

### 내보내기 (export)
```javascript
// utils/date.js

export const today = () => new Date();

export function formatDate(d) {
  return d.toISOString().slice(0, 10);
}

export const MONTHS = ["Jan", "Feb", "Mar", ...];
```

### 가져오기 (import)
```javascript
// App.js
import { today, formatDate, MONTHS } from "./utils/date.js";

console.log(formatDate(today()));
```

- 같은 폴더면 `./`
- 부모 폴더면 `../`
- node_modules 면 패키지 이름만 — `import x from "lodash";`

---

## 3. default export (한 파일에 1개)

```javascript
// Button.js
function Button({ text }) {
  return <button>{text}</button>;
}

export default Button;
```

```javascript
// App.js
import Button from "./Button.js";   // 중괄호 X
// 이름은 자유 — import MyBtn from "./Button.js" 도 가능
```

### default + named 같이
```javascript
// Button.js
export default function Button() {}

export const BUTTON_SIZES = ["sm", "md", "lg"];
```

```javascript
import Button, { BUTTON_SIZES } from "./Button.js";
```

---

## 4. 별칭 (as)

```javascript
import { formatDate as fmt } from "./date.js";
fmt(new Date());

// 또는 다 가져오기
import * as DateUtils from "./date.js";
DateUtils.today();
DateUtils.formatDate(...);
```

---

## 5. 재내보내기 (re-export)

```javascript
// utils/index.js — barrel file
export { today, formatDate } from "./date.js";
export { capitalize } from "./string.js";
export { default as Button } from "./Button.js";
```

```javascript
// 다른 파일
import { today, capitalize, Button } from "./utils";
```

`utils/index.js` 한 곳에 모아둠 → import 짧아짐.

---

## 6. 브라우저에서 모듈 사용

### `<script type="module">`
```html
<script type="module" src="./app.js"></script>
```

`type="module"` 필수. 안 하면 import 안 됨.

### CORS 주의
모듈은 `file://` 으로 안 열림 (`http://` 필요). Live Server / Vite dev 서버 사용.

---

## 7. Vite / Next / 번들러 환경

위 표준 ES Module 문법 그대로 사용. 번들러 (Vite, webpack 등) 가:
- 모든 import 추적
- 하나의 큰 파일로 묶음 (production)
- dev 에서는 즉시 import / hot reload

---

## 8. CommonJS (구 방식 — Node.js 옛 코드)

```javascript
// utils.js
function add(a, b) { return a + b; }
module.exports = { add };

// 가져오기
const { add } = require("./utils.js");
```

요즘은 ES Module 권장. Node 도 ES Module 기본 지원 (`"type": "module"` in package.json).

---

## 9. import 종류별 차이 한눈에

```javascript
// named
export const x = 1;
import { x } from "...";

// default
export default function() {}
import any_name from "...";

// 둘 다
export default function Foo() {}
export const BAR = 1;
import Foo, { BAR } from "...";

// 별칭
import { x as y } from "...";

// 전체
import * as ns from "...";
ns.x;
```

---

## 10. dynamic import (코드 분할)

```javascript
async function loadChart() {
  const { default: Chart } = await import("./Chart.js");
  return new Chart();
}
```

페이지 진입 시 안 가져오고, 필요할 때 가져오기 → 초기 로딩 빠름.

Next 의 `next/dynamic`, React 의 `lazy()` 도 같은 원리.

---

## 11. 외부 패키지 (npm)

### 설치
```bash
npm install lodash
```

### 사용
```javascript
import _ from "lodash";

_.chunk([1,2,3,4], 2);    // [[1,2], [3,4]]
```

### 특정 부분만
```javascript
import { chunk } from "lodash";
chunk([1,2,3,4], 2);

// 트리 쉐이킹 (번들 크기 줄임)
import chunk from "lodash/chunk";
```

---

## 12. import 순서 컨벤션

가독성 위해 보통 이 순서:

```javascript
// 1. 외부 패키지
import React from "react";
import { Link } from "react-router-dom";

// 2. 내 절대 경로 (alias)
import Layout from "@/components/Layout";

// 3. 상대 경로
import { user } from "./data";
import "./style.css";
```

각 그룹 사이 빈 줄.

ESLint 의 `import/order` rule 로 자동 강제 가능.

---

## 13. 파일 확장자

```javascript
// 명시 (안전)
import x from "./utils.js";
import x from "./utils.ts";

// 생략 (번들러가 자동 추측)
import x from "./utils";
```

Vite / Next 는 자동으로 `.js`, `.ts`, `.tsx`, `.jsx` 등 찾음.

순수 Node 는 명시 권장.

---

## 14. 자주 만드는 패턴

### A. 유틸 함수 모음
```javascript
// utils/format.js
export const formatPrice = (n) => `${n.toLocaleString()}원`;
export const formatDate  = (d) => d.toISOString().slice(0, 10);
export const formatTime  = (d) => d.toTimeString().slice(0, 5);
```

```javascript
import { formatPrice } from "./utils/format";
```

### B. 상수 모음
```javascript
// constants.js
export const API_URL = "https://api.example.com";
export const PAGE_SIZE = 20;
export const COLORS = {
  primary: "#4ade80",
  danger: "#f87171",
};
```

### C. 컴포넌트 + 타입
```typescript
// Button.tsx
export type ButtonProps = { text: string; onClick?: () => void };

export default function Button({ text, onClick }: ButtonProps) {
  return <button onClick={onClick}>{text}</button>;
}
```

```typescript
import Button, { type ButtonProps } from "./Button";
```

---

## 15. 함정

| 증상 | 원인 |
| --- | --- |
| `import 가 syntax error` | `<script type="module">` 안 함 / 번들러 없는 환경 |
| `export default` 가 두 개 | 한 파일에 1번만 |
| `import { x }` 에러 — undefined | 다른 파일에 `export const x` 안 함 |
| 순환 참조 (A → B → A) | 구조 재설계 — 보통 공통 base 로 빼냄 |
| 같은 이름 두 번 import | 별칭 (`as`) 사용 |
| `index.js` 인데 import 가 `./folder/` 만 적었는데도 됨 | index 가 자동 entry. 명시 권장 |

---

## 16. 다음 단계

1. **[[../../typescript/|TypeScript]]** — 타입 + import
2. **[[../../react/getting-started|React getting started]]** — JS 모듈로 컴포넌트 조립

---

연결: [[../javascript|↑ hub]] · [[../objects-arrays/objects]]
