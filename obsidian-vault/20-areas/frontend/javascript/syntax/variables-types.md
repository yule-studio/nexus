---
title: "JS 변수와 타입"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, beginner, variables, types]
---

# 변수와 타입

**[[../javascript|↑ JavaScript hub]]**

---

## 1. 변수 선언 3 가지

### `let` — 값 바뀔 수 있음
```javascript
let name = "유찬";
name = "철수";   // OK
```

### `const` — 값 고정 (한 번만 할당)
```javascript
const PI = 3.14;
PI = 3.14159;    // ❌ 에러
```

### `var` — 옛 방식 (쓰지 마세요)
```javascript
var x = 1;       // 비추천. let / const 사용.
```

### 기본 규칙
- 우선 `const` — 변경 안 되면 다.
- 변경되어야 하면 `let`.
- `var` 안 씀.

---

## 2. 데이터 타입 7 종 (primitive)

### string — 문자열
```javascript
const a = "큰 따옴표";
const b = '작은 따옴표';
const c = `백틱 (템플릿)`;

const name = "유찬";
const greeting = `안녕, ${name}!`;     // "안녕, 유찬!"
```

백틱은 `${...}` 로 변수 끼워넣기 + 여러 줄 OK.

### number — 숫자
```javascript
const age = 25;
const pi = 3.14;
const big = 1_000_000;     // 1000000 (가독성)
const negative = -10;
const inf = Infinity;
const notNumber = NaN;     // 0/0 같은 결과
```

### boolean — true / false
```javascript
const isReady = true;
const isDone = false;
```

### null — 의도적 "비어 있음"
```javascript
const user = null;
```

### undefined — 값 없음 (자동)
```javascript
let x;            // 자동 undefined
console.log(x);   // undefined
```

### bigint — 큰 정수
```javascript
const big = 9007199254740993n;   // n 붙임. 거의 안 씀.
```

### symbol — 유일한 값
```javascript
const id = Symbol("id");   // 거의 안 씀.
```

---

## 3. 객체 (object) — 비-primitive

```javascript
const user = { name: "유찬", age: 25 };
```

자세히 [[../objects-arrays/objects]].

### 배열도 객체의 일종
```javascript
const list = [1, 2, 3];
```

자세히 [[../objects-arrays/arrays]].

---

## 4. 타입 확인 — typeof

```javascript
typeof "hi"        // "string"
typeof 42          // "number"
typeof true        // "boolean"
typeof undefined   // "undefined"
typeof null        // "object" (역사적 버그)
typeof {}          // "object"
typeof []          // "object"
typeof function(){}// "function"
```

배열 확인:
```javascript
Array.isArray([1,2,3])   // true
```

---

## 5. 연산자

### 산술
```javascript
5 + 3        // 8
5 - 3        // 2
5 * 3        // 15
5 / 3        // 1.666...
5 % 3        // 2 (나머지)
5 ** 3       // 125 (거듭제곱)
```

### 문자열 합치기
```javascript
"a" + "b"            // "ab"
"안녕" + " " + "유찬"  // "안녕 유찬"
```

### 비교 (`==` vs `===`)
```javascript
5 == "5"     // true   (타입 변환 후 비교 — 비추천)
5 === "5"    // false  (엄격 비교 — 추천)
5 === 5      // true
```

**항상 `===` 사용**.

### 부등호
```javascript
5 > 3        // true
5 >= 5       // true
5 < 3        // false
5 !== 3      // true
```

### 논리 (boolean)
```javascript
true && false   // false (AND)
true || false   // true  (OR)
!true           // false (NOT)
```

### 단축 (short-circuit)
```javascript
user && user.name           // user 가 truthy 면 user.name
user || "기본값"            // user 가 falsy 면 "기본값"
user?.name                  // user 가 null/undefined 면 undefined
user?.name ?? "기본"        // user.name 이 null/undefined 면 "기본"
```

`??` (nullish) 는 `null/undefined` 만, `||` 는 falsy 전체 (`0`, `""` 포함).

### 할당
```javascript
let x = 10;
x += 5;     // x = x + 5 = 15
x -= 2;     // 13
x *= 2;     // 26
x /= 2;     // 13
```

---

## 6. truthy / falsy

JavaScript 의 모든 값은 `if(...)` 에서 true / false 로 판정.

### Falsy (false 로 취급)
```javascript
false
0
""           // 빈 문자열
null
undefined
NaN
```

### Truthy (true 로 취급)
나머지 전부 — `1`, `"hi"`, `{}`, `[]`, `"0"`, `"false"` 도 truthy.

### 활용
```javascript
if (user) {
  // user 가 truthy 면 (= 객체가 있음)
}

if (count) {
  // count 가 0 이면 실행 안 됨
}
```

---

## 7. 문자열 다루기

```javascript
const s = "Hello World";

s.length              // 11
s[0]                  // "H"
s.toUpperCase()       // "HELLO WORLD"
s.toLowerCase()       // "hello world"
s.includes("World")   // true
s.indexOf("World")    // 6
s.startsWith("He")    // true
s.endsWith("ld")      // true
s.slice(0, 5)         // "Hello"
s.split(" ")          // ["Hello", "World"]
s.replace("World", "JS") // "Hello JS"
s.trim()              // 앞뒤 공백 제거
s.repeat(3)           // "Hello WorldHello WorldHello World"
```

---

## 8. 숫자 다루기

```javascript
const n = 3.7;

Math.floor(n)         // 3 (내림)
Math.ceil(n)          // 4 (올림)
Math.round(n)         // 4 (반올림)
Math.abs(-5)          // 5
Math.max(1, 2, 3)     // 3
Math.min(1, 2, 3)     // 1
Math.random()         // 0~1 사이 랜덤
Math.pow(2, 10)       // 1024

(3.14159).toFixed(2)  // "3.14" (문자열)
parseInt("42")        // 42
parseFloat("3.14")    // 3.14
Number("42")          // 42
```

---

## 9. 형 변환

### 자동 변환 (피하기)
```javascript
1 + "1"      // "11" (숫자가 문자열로)
"5" - 1      // 4 (문자열이 숫자로)
```

### 명시적 변환
```javascript
Number("42")          // 42
String(42)            // "42"
Boolean(0)            // false
Boolean("a")          // true

// 단축
+("42")               // 42 (Number 단축)
""+42                 // "42" (String 단축)
!!"a"                 // true (Boolean 단축)
```

---

## 10. const vs let — 헷갈리는 케이스

### const 의 객체 / 배열
```javascript
const user = { name: "유찬" };
user.name = "철수";        // ✅ OK — 객체 내부 변경
user = { name: "다른" };   // ❌ 에러 — 재할당

const list = [1, 2];
list.push(3);              // ✅ OK — 배열 내부 추가
list = [4, 5];             // ❌ 에러
```

`const` 는 "재할당 금지", 내부는 변경 가능.

---

## 11. 변수 이름 규칙

- 영문 / 숫자 / `_` / `$` 만
- 숫자로 시작 X
- 예약어 X (`if`, `for` 등)
- camelCase 권장 (`userName`, `myFunction`)
- `UPPER_SNAKE` — 상수 (`MAX_SIZE`)
- `PascalCase` — 클래스 / 컴포넌트 (`UserCard`)

```javascript
const userName = "유찬";       // ✅
const user_name = "유찬";      // 가능하지만 비추천 (snake_case)
const MAX_SIZE = 100;          // ✅ 상수
class UserCard {}              // ✅
```

---

## 12. 주석

```javascript
// 한 줄 주석

/*
  여러 줄
  주석
*/
```

---

## 13. console — 디버깅

```javascript
console.log("일반");
console.error("에러");
console.warn("경고");
console.table([{a:1,b:2},{a:3,b:4}]);

// 여러 값
console.log("값:", x, "타입:", typeof x);
```

브라우저 콘솔 (F12 → Console) 에서 확인.

---

## 14. 함정

| 증상 | 원인 |
| --- | --- |
| `5 + "5"` = `"55"` | + 가 문자열이면 합치기 |
| `0.1 + 0.2 !== 0.3` | 부동소수점 오차 — `Math.round` 또는 `toFixed` |
| `null == undefined` = `true` | `===` 쓰면 false |
| `typeof null === "object"` | 역사적 버그. null 확인은 `=== null` |
| `if(0)` 실행 안 됨 | 0 은 falsy |
| `const x = 5; x = 10;` | const 재할당 금지. let 으로 |

---

## 15. 다음 단계

1. **[[control-flow]]** — if / for / while
2. **[[../functions/functions-basics]]** — 함수

---

연결: [[../javascript|↑ hub]] · [[control-flow]]
