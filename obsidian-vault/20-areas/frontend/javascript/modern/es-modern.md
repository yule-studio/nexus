---
title: "Modern JS — ES2020+ 핵심 문법"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, modern, es2020, es2024]
---

# Modern JS (ES2020+)

**[[../javascript|↑ JavaScript hub]]**

> 현업에서 자주 쓰는 모던 JS 문법.

---

## 1. Optional Chaining (`?.`)

ES2020.

```javascript
user?.address?.city                 // 중간이 null/undefined 면 undefined

users?.[0]?.name                    // 배열 인덱스
fn?.(arg)                            // 함수 호출 (있으면)

// 옛 방식
user && user.address && user.address.city
```

---

## 2. Nullish Coalescing (`??`)

ES2020.

```javascript
value ?? "default"                  // null/undefined 면 default
```

### `||` 와 차이
```javascript
0 || "default"        // "default" (0 은 falsy)
0 ?? "default"        // 0 (null/undefined 만 false 취급)

"" || "default"       // "default"
"" ?? "default"       // ""
```

`??` 가 더 안전 (0 / "" / false 같은 의도된 값 유지).

---

## 3. Logical Assignment (`??=` `||=` `&&=`)

ES2021.

```javascript
obj.value ??= "default";    // value 가 null/undefined 면 할당
obj.value ||= "default";    // falsy 면 할당
obj.value &&= "default";    // truthy 면 할당
```

```javascript
// 위 = 아래
obj.value = obj.value ?? "default";
```

---

## 4. Numeric Separators

ES2021.

```javascript
const big = 1_000_000;          // 가독성
const hex = 0xFF_FF_FF;
const bin = 0b1010_0101;
```

작동 동일 — `_` 무시. 사람용.

---

## 5. String.replaceAll

ES2021.

```javascript
"a-b-c".replaceAll("-", "_")   // "a_b_c"

// 옛
"a-b-c".replace(/-/g, "_")
```

---

## 6. Array.at

ES2022.

```javascript
[1, 2, 3].at(-1)         // 3 (뒤에서)
[1, 2, 3].at(0)          // 1

// 옛
arr[arr.length - 1]
```

---

## 7. Object.hasOwn

ES2022.

```javascript
Object.hasOwn(obj, "name")     // 안전
obj.hasOwnProperty("name")      // 옛 (prototype pollution 위험)
```

---

## 8. structuredClone (deep copy)

ES2022.

```javascript
const a = { x: { y: 1 } };
const b = structuredClone(a);

b.x.y = 999;
console.log(a.x.y);   // 1 (영향 X)
```

JSON.parse(JSON.stringify) 의 안전 대체 (Date, Map, Set, function 등 처리 더 정확).

---

## 9. Array group / groupToMap

ES2024.

```javascript
const items = [
  { type: "fruit", name: "apple" },
  { type: "fruit", name: "banana" },
  { type: "veg", name: "carrot" },
];

const grouped = Object.groupBy(items, (i) => i.type);
// { fruit: [...], veg: [...] }

const mapGrouped = Map.groupBy(items, (i) => i.type);
// Map { "fruit" => [...], "veg" => [...] }
```

---

## 10. Promise.withResolvers

ES2024.

```javascript
const { promise, resolve, reject } = Promise.withResolvers();

// 다른 곳에서
setTimeout(() => resolve("done"), 1000);

await promise;   // "done"
```

옛 방식 (deferred):
```javascript
let resolve, reject;
const promise = new Promise((r, rj) => { resolve = r; reject = rj; });
```

---

## 11. AbortSignal.timeout

```javascript
// 5초 timeout
const res = await fetch(url, { signal: AbortSignal.timeout(5000) });
```

옛:
```javascript
const c = new AbortController();
setTimeout(() => c.abort(), 5000);
fetch(url, { signal: c.signal });
```

---

## 12. private class fields (`#`)

ES2022.

```javascript
class Counter {
  #count = 0;        // private (외부 접근 X)

  increment() { this.#count++; }
  get value() { return this.#count; }
}

const c = new Counter();
c.#count;        // ❌ 에러
c.value;          // OK
```

---

## 13. static class members

```javascript
class MathUtils {
  static PI = 3.14;
  static add(a, b) { return a + b; }
}

MathUtils.PI;
MathUtils.add(1, 2);
```

---

## 14. Top-level await

ES2022. ES Module 최상위에서:
```javascript
// main.mjs
const data = await fetch(url).then(r => r.json());
console.log(data);
```

async 함수 wrap 불필요.

---

## 15. iterator helpers (ES2025)

```javascript
const result = arr
  .values()
  .filter((n) => n > 0)
  .map((n) => n * 2)
  .take(10)
  .toArray();
```

Iterator 자체에 method (메모리 효율). Chrome 122+.

---

## 16. Set methods (ES2024)

```javascript
const a = new Set([1, 2, 3]);
const b = new Set([2, 3, 4]);

a.intersection(b);    // Set { 2, 3 }
a.union(b);           // Set { 1, 2, 3, 4 }
a.difference(b);      // Set { 1 }
a.symmetricDifference(b);   // Set { 1, 4 }

a.isSubsetOf(b);      // false
a.isSupersetOf(b);    // false
a.isDisjointFrom(b);  // false
```

---

## 17. Temporal (예정, 차세대 Date)

Date 의 부족함을 보완. Stage 3, 곧 표준.

```javascript
const date = Temporal.Now.plainDateTimeISO();
const tomorrow = date.add({ days: 1 });
const diff = date.until(other);
```

지금은 polyfill 필요 — `@js-temporal/polyfill`.

대신 `date-fns` / `dayjs` / `luxon` 권장 (현 시점).

---

## 18. import.meta

ES Module 의 메타 정보:
```javascript
import.meta.url           // "file:///..."
import.meta.env          // Vite 환경변수 (Vite 만)
```

---

## 19. dynamic import

```javascript
const module = await import("./heavy");
```

자세히 [[../modules/modules-and-imports]].

---

## 20. WeakRef / WeakMap / WeakSet

GC 친화적 자료구조. cache / memoization 에 활용.

```javascript
const cache = new WeakMap();
cache.set(user, { computed: ... });
```

key 객체가 GC 되면 자동 cleanup.

---

## 21. Regex `d` flag (indices)

```javascript
const match = "hello world".match(/(\w+)/d);
console.log(match.indices);   // [[0, 5], [0, 5]]
```

각 capture group 의 정확한 위치.

---

## 22. 자주 만드는 패턴

### A. 안전한 환경변수 접근
```javascript
const PORT = Number(process.env.PORT ?? 3000);
```

### B. 객체 합치기 (안전)
```javascript
const result = structuredClone({ ...defaults, ...userOpts });
```

### C. 조건부 객체 property
```javascript
const obj = {
  name: "유찬",
  ...(includeAge && { age: 25 }),    // 조건 부합 시만
};
```

### D. 빈 배열 / 객체
```javascript
const arr = data?.items ?? [];
const obj = data?.user ?? {};
```

### E. 첫 / 마지막
```javascript
arr.at(0)         // first
arr.at(-1)        // last
```

---

## 23. 브라우저 지원

| 기능 | 지원 |
| --- | --- |
| `?.` `??` | Chrome 80+ / Safari 13.4+ / Firefox 74+ (모두 모던) |
| `Array.at` | Chrome 92+ |
| `structuredClone` | Chrome 98+ |
| `Object.groupBy` | Chrome 117+ |
| `Set.intersection` | Chrome 122+ |
| Top-level await (module) | Chrome 89+ |

타겟 브라우저 / Vite 의 target 설정 따라 polyfill 필요할 수 있음.

---

## 24. 다음 단계

1. **[[../async/promises-async-await]]** — 비동기
2. **[[../modules/modules-and-imports]]** — 모듈

---

연결: [[../javascript|↑ hub]]
