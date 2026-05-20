---
title: "JS 함수 기초"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, beginner, functions]
---

# 함수 기초

**[[../javascript|↑ JavaScript hub]]**

---

## 1. 함수 정의 3 가지

### 함수 선언 (function declaration)
```javascript
function greet(name) {
  return `안녕, ${name}!`;
}

greet("유찬");   // "안녕, 유찬!"
```

### 함수 표현 (function expression)
```javascript
const greet = function(name) {
  return `안녕, ${name}!`;
};
```

### 화살표 함수 (arrow — 가장 현대적, 가장 자주)
```javascript
const greet = (name) => {
  return `안녕, ${name}!`;
};

// 한 줄이면 더 짧게
const greet = (name) => `안녕, ${name}!`;

// 매개변수 1개면 괄호 생략 가능
const greet = name => `안녕, ${name}!`;

// 매개변수 없으면 빈 괄호 필수
const now = () => Date.now();
```

### 셋 다 비슷하지만
- React 같은 모던 코드는 거의 다 화살표 함수
- 차이는 `this` 처리 + hoisting (아래 §10)

---

## 2. 매개변수와 인자

```javascript
function add(a, b) {       // a, b = 매개변수 (parameter)
  return a + b;
}

add(3, 5);                  // 3, 5 = 인자 (argument)
```

### 기본값 (default parameter)
```javascript
function greet(name = "친구") {
  return `안녕, ${name}!`;
}

greet();         // "안녕, 친구!"
greet("유찬");   // "안녕, 유찬!"
```

### 나머지 매개변수 (rest)
```javascript
function sum(...numbers) {
  return numbers.reduce((acc, n) => acc + n, 0);
}

sum(1, 2, 3, 4);   // 10
```

`...` 로 여러 인자를 배열로 묶음.

### 매개변수 개수 안 맞아도 OK
```javascript
function add(a, b) { return a + b; }

add(1);          // NaN (b 가 undefined)
add(1, 2, 3);    // 3 (3 무시)
```

JS 는 매개변수 검사 안 함. 그래서 TypeScript 가 인기.

---

## 3. return

함수가 결과를 돌려줌.

```javascript
function double(n) {
  return n * 2;
}

const result = double(5);   // 10
```

### return 없으면 undefined
```javascript
function noReturn() {
  console.log("hi");
}

const x = noReturn();   // undefined
```

### early return
```javascript
function divide(a, b) {
  if (b === 0) return null;   // 일찍 빠져나옴
  return a / b;
}
```

---

## 4. 함수 안 변수 (스코프)

```javascript
function outer() {
  const x = 1;
  console.log(x);   // OK
}

console.log(x);     // ❌ 에러 — outer 안에서만 보임
```

### 중첩
```javascript
function outer() {
  const x = 1;
  function inner() {
    console.log(x);   // ✅ OK (부모 스코프 접근)
  }
  inner();
}
```

---

## 5. 콜백 함수 (callback)

다른 함수에 함수를 인자로 전달.

```javascript
function greetUser(callback) {
  const name = "유찬";
  callback(name);
}

greetUser((name) => {
  console.log(`안녕, ${name}!`);
});
```

### 자주 보는 곳 — array method
```javascript
[1, 2, 3].map((n) => n * 2);
//          ↑ 콜백 함수
```

### 이벤트 리스너
```javascript
button.addEventListener("click", () => {
  console.log("눌렸음");
});
```

---

## 6. 화살표 함수 — 자세히

### 한 줄짜리 (implicit return)
```javascript
const double = n => n * 2;
//                  ↑ return 생략 가능 (한 줄)
```

### 객체 반환할 때 주의 — 괄호로 감싸기
```javascript
const makeUser = (name) => ({ name });   // 객체 리턴
//                          ↑ 괄호 필수

// 이렇게 하면 ❌
const makeUser = (name) => { name };     // 빈 블록으로 해석됨
```

### 여러 줄
```javascript
const compute = (a, b) => {
  const sum = a + b;
  const product = a * b;
  return { sum, product };
};
```

### 매개변수 없음
```javascript
const now = () => Date.now();
```

---

## 7. 함수도 값 — 변수 / 배열 / 객체 에 담기

### 변수에
```javascript
const greet = (name) => `안녕, ${name}!`;
```

### 배열에
```javascript
const fns = [
  (x) => x + 1,
  (x) => x * 2,
];
fns[0](5);   // 6
fns[1](5);   // 10
```

### 객체에 — method
```javascript
const calculator = {
  add: (a, b) => a + b,
  sub: (a, b) => a - b,
};
calculator.add(3, 5);   // 8
```

### 함수에서 함수 반환 (higher-order)
```javascript
function multiplier(n) {
  return (x) => x * n;
}

const double = multiplier(2);
const triple = multiplier(3);

double(5);   // 10
triple(5);   // 15
```

---

## 8. 즉시 실행 함수 (IIFE)

```javascript
(function() {
  console.log("정의하자마자 실행");
})();

// 화살표 버전
(() => {
  console.log("실행");
})();
```

옛 방식. 요즘은 module 쓰니까 거의 안 씀.

---

## 9. 함수 종류별 사용 시점

| 종류 | 언제 |
| --- | --- |
| `const x = () => ...` | 거의 모든 곳 (요즘 표준) |
| `function x() { ... }` | 파일 최상위, hoisting 활용 시 |
| `obj.method() { ... }` | 객체 내부 method (this 활용) |

---

## 10. hoisting

```javascript
greet();   // ✅ 작동 — 함수 선언은 위로 hoisting
function greet() { console.log("hi"); }
```

```javascript
greet();   // ❌ 에러 — 변수 (화살표 / 표현식) 은 hoisting 안 됨
const greet = () => console.log("hi");
```

### 권장
hoisting 에 의존 X. **선언 → 사용** 순서로.

---

## 11. this (간단)

객체의 method 에서 `this` = 그 객체.

```javascript
const user = {
  name: "유찬",
  greet() {
    console.log(`안녕, ${this.name}!`);
  },
};

user.greet();   // "안녕, 유찬!"
```

### 화살표 함수의 this 는 다름
```javascript
const user = {
  name: "유찬",
  greet: () => {
    console.log(this.name);   // undefined (화살표는 자체 this 없음)
  },
};
```

→ method 정의 시 화살표 함수 쓰지 마세요. 단순 콜백은 OK.

자세한 `this` 는 OOP 부분에서.

---

## 12. 자주 만드는 함수 패턴

### A. 간단 계산
```javascript
const add = (a, b) => a + b;
```

### B. 객체 가공
```javascript
const formatUser = (user) => ({
  ...user,
  fullName: `${user.firstName} ${user.lastName}`,
});
```

### C. 조건 분기 함수
```javascript
function getStatus(score) {
  if (score >= 90) return "A";
  if (score >= 80) return "B";
  if (score >= 70) return "C";
  return "F";
}
```

### D. 비동기 (Promise)
```javascript
async function fetchUser(id) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}
```

자세히 [[../async/promises-async-await]].

### E. 이벤트 핸들러
```javascript
button.addEventListener("click", (event) => {
  event.preventDefault();
  console.log("눌림");
});
```

---

## 13. 함정

| 증상 | 원인 |
| --- | --- |
| `return` 빠뜨려 undefined | 모든 분기에 return 확인 |
| 화살표 함수에서 `this` 의도와 다름 | method 정의는 일반 함수로 |
| `()=>` 가 객체 반환 안 됨 | `({...})` 괄호 감싸기 |
| 매개변수 너무 많아서 헷갈림 | 객체 1개로: `function fn({ a, b, c }) {}` |
| callback 안 callback 안 callback | "콜백 지옥" — async/await 으로 해결 |

---

## 14. 다음 단계

1. **[[../objects-arrays/objects]]** — 객체
2. **[[../objects-arrays/arrays]]** — 배열 method
3. **[[../async/promises-async-await]]** — Promise / async

---

연결: [[../javascript|↑ hub]] · [[../syntax/control-flow]]
