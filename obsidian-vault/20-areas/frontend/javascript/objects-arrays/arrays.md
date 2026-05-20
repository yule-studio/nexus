---
title: "JS 배열과 method"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, beginner, arrays]
---

# 배열 (Array)

**[[../javascript|↑ JavaScript hub]]**

> 순서 있는 값 모음. React / Next.js 에서 가장 많이 다루는 자료구조.

---

## 1. 배열 만들기

```javascript
const list = [1, 2, 3];
const names = ["a", "b", "c"];
const mixed = [1, "two", true, null, { x: 1 }];   // 타입 섞임 OK
const empty = [];

// 길이로
const tens = new Array(10);    // 길이 10, 다 undefined
```

---

## 2. 인덱스 접근

```javascript
const list = ["a", "b", "c"];

list[0]              // "a"
list[1]              // "b"
list[2]              // "c"
list[100]            // undefined
list[-1]             // undefined (음수 인덱스 X)

list.at(0)           // "a"
list.at(-1)          // "c" (뒤에서)
```

---

## 3. 기본 method

```javascript
const list = [1, 2, 3];

list.length          // 3
list.push(4)         // [1,2,3,4]  - 뒤에 추가, 새 length 리턴
list.pop()           // [1,2,3]    - 뒤 제거, 제거된 값 리턴
list.unshift(0)      // [0,1,2,3]  - 앞에 추가
list.shift()         // [1,2,3]    - 앞 제거
list.includes(2)     // true
list.indexOf(2)      // 1 (위치)
list.indexOf(99)     // -1 (없음)
list.reverse()       // 원본 뒤집기
list.sort()          // 원본 정렬 (문자열 비교 — 주의!)
list.join(", ")      // "1, 2, 3"
list.slice(1, 3)     // [2, 3] (1번 부터 3번 전까지)
list.concat([4, 5])  // [1, 2, 3, 4, 5]
```

### 원본 변경 vs 새 배열
**원본 변경 (mutating)**: `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`
**새 배열 반환**: `slice`, `concat`, `map`, `filter`, `reduce`, 스프레드 `[...]`

React 는 보통 **새 배열 반환** 방식 사용 (불변성).

---

## 4. 정렬 — sort 주의

```javascript
[10, 2, 1].sort()           // [1, 10, 2]   ❌ 문자열 비교 ("10" < "2")
[10, 2, 1].sort((a, b) => a - b)   // [1, 2, 10]  ✅ 숫자 정렬
[10, 2, 1].sort((a, b) => b - a)   // [10, 2, 1]  내림차순

// 객체 정렬
users.sort((a, b) => a.age - b.age);                  // 나이 오름차순
users.sort((a, b) => a.name.localeCompare(b.name));   // 한국어 정렬
```

---

## 5. 자주 쓰는 6 method (필수)

### 5.1 forEach — 단순 반복
```javascript
[1, 2, 3].forEach((n, i) => {
  console.log(i, n);
  // 0 1
  // 1 2
  // 2 3
});
```

값 반환 X. 부수 효과 (로그 / DOM 조작 등) 용.

### 5.2 map — 변환
```javascript
const doubled = [1, 2, 3].map((n) => n * 2);
// [2, 4, 6]

const names = users.map((u) => u.name);
// 객체 배열 → 이름 배열
```

각 항목 → 새 값. **새 배열 반환** (같은 길이).

### 5.3 filter — 조건 맞는 것만
```javascript
const adults = users.filter((u) => u.age >= 20);
const evens  = [1,2,3,4,5].filter((n) => n % 2 === 0);   // [2, 4]
```

**새 배열 반환** (보통 더 짧음).

### 5.4 reduce — 누적
```javascript
const sum = [1, 2, 3, 4].reduce((acc, n) => acc + n, 0);
// 10
// (acc=0, n=1) → 1 → (acc=1, n=2) → 3 → ... → 10
```

`reduce(callback, 초기값)`. 단일 값으로 압축.

```javascript
// 객체 만들기
const counts = ["a", "b", "a", "c"].reduce((acc, x) => {
  acc[x] = (acc[x] ?? 0) + 1;
  return acc;
}, {});
// { a: 2, b: 1, c: 1 }
```

### 5.5 find / findIndex — 찾기
```javascript
const user = users.find((u) => u.id === 42);     // 첫 일치 객체
const idx  = users.findIndex((u) => u.id === 42); // 첫 일치 인덱스
```

없으면 `undefined` / `-1`.

### 5.6 some / every — boolean
```javascript
[1, 2, 3].some((n) => n > 2);    // true (하나라도)
[1, 2, 3].every((n) => n > 0);   // true (전부)
```

---

## 6. 체이닝 (여러 method 이어서)

```javascript
const result = users
  .filter((u) => u.age >= 20)
  .map((u) => u.name)
  .sort();
```

읽기 쉬움. 매 단계가 새 배열 반환이라 가능.

---

## 7. spread (`...`) — 펴기 / 합치기

### 복사
```javascript
const a = [1, 2, 3];
const b = [...a];      // 새 배열
```

### 합치기
```javascript
const a = [1, 2];
const b = [3, 4];
const c = [...a, ...b];   // [1, 2, 3, 4]
const d = [...a, 999, ...b];  // [1, 2, 999, 3, 4]
```

### 함수 인자로
```javascript
const nums = [1, 2, 3];
Math.max(...nums);    // 3 (= Math.max(1, 2, 3))
```

---

## 8. 구조 분해

```javascript
const [first, second, ...rest] = [1, 2, 3, 4, 5];
// first = 1, second = 2, rest = [3, 4, 5]

// 건너뛰기
const [, , third] = [1, 2, 3];
// third = 3
```

---

## 9. 새/추가/제거 (불변성 패턴)

React 에서 자주.

### 추가
```javascript
const list = [1, 2, 3];

// 뒤에
const a = [...list, 4];          // [1, 2, 3, 4]

// 앞에
const b = [0, ...list];          // [0, 1, 2, 3]

// 중간 (인덱스 2 에 99 끼우기)
const c = [...list.slice(0, 2), 99, ...list.slice(2)];
// [1, 2, 99, 3]
```

### 제거 (특정 인덱스)
```javascript
const list = [1, 2, 3, 4];

// 인덱스 1 제거
const a = list.filter((_, i) => i !== 1);
// [1, 3, 4]
```

### 제거 (특정 값)
```javascript
const a = list.filter((n) => n !== 3);   // 3 빼고 다
```

### 업데이트 (특정 인덱스)
```javascript
const list = [1, 2, 3];
const a = list.map((n, i) => (i === 1 ? 99 : n));
// [1, 99, 3]
```

### 업데이트 (조건)
```javascript
const list = [{ id: 1, done: false }, { id: 2, done: false }];

const a = list.map((item) =>
  item.id === 1 ? { ...item, done: true } : item
);
```

---

## 10. flat / flatMap

```javascript
[1, [2, 3], [4, [5]]].flat();        // [1, 2, 3, 4, [5]]   (1 단계)
[1, [2, 3], [4, [5]]].flat(2);       // [1, 2, 3, 4, 5]

// map 후 flat
[[1,2],[3,4]].flatMap((x) => x);     // [1, 2, 3, 4]
```

---

## 11. Array.from / Array.of

### Array.from — iterable → array
```javascript
Array.from("hi");           // ["h", "i"]
Array.from({ length: 3 }, (_, i) => i);  // [0, 1, 2]

// NodeList → 진짜 배열
Array.from(document.querySelectorAll("p"));
```

### Array.of
```javascript
Array.of(1, 2, 3);   // [1, 2, 3]
```

---

## 12. 자주 만드는 패턴

### A. 객체 배열에서 특정 필드 모음
```javascript
const names = users.map((u) => u.name);
const ids   = users.map((u) => u.id);
```

### B. 합계 / 평균
```javascript
const sum = numbers.reduce((a, b) => a + b, 0);
const avg = sum / numbers.length;
```

### C. 그룹화
```javascript
const groups = items.reduce((acc, item) => {
  const key = item.category;
  if (!acc[key]) acc[key] = [];
  acc[key].push(item);
  return acc;
}, {});
// { books: [...], music: [...] }
```

### D. 중복 제거
```javascript
const unique = [...new Set([1, 2, 2, 3])];
// [1, 2, 3]
```

### E. 두 배열 합치고 정렬
```javascript
const merged = [...a, ...b].sort((x, y) => x - y);
```

### F. 페이지네이션
```javascript
const PAGE_SIZE = 10;
const page = items.slice((pageNum - 1) * PAGE_SIZE, pageNum * PAGE_SIZE);
```

---

## 13. 함정

| 증상 | 원인 |
| --- | --- |
| `list.sort()` 가 숫자 정렬 안 됨 | 비교 함수 필요: `sort((a,b) => a - b)` |
| `for...in` 으로 배열 → 이상한 결과 | `for...of` 또는 `forEach` |
| `push` 결과를 받아서 사용했더니 length | `push` 는 새 length 리턴. 원본 변경. |
| `map` 안 return 빠뜨림 | undefined 배열 됨 |
| 객체 동일 비교가 안 됨 | 참조 비교라 같은 내용이라도 false |
| `length` 변경으로 자름 가능 | `list.length = 3` → 4번째부터 잘림 (잘 안 씀) |

---

## 14. 다음 단계

1. **[[../async/promises-async-await]]** — 비동기 / fetch
2. **[[../modules/modules-and-imports]]** — 파일 분리

---

연결: [[../javascript|↑ hub]] · [[objects]] · [[../syntax/control-flow]]
