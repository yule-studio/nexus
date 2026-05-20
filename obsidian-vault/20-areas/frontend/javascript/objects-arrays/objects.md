---
title: "JS 객체 (object)"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, beginner, objects]
---

# 객체 (object)

**[[../javascript|↑ JavaScript hub]]**

> key-value 쌍 모음. JS 의 모든 데이터 구조의 기본.

---

## 1. 객체 만들기

```javascript
const user = {
  name: "유찬",
  age: 25,
  email: "yc@example.com",
};
```

- 중괄호 `{ }`
- `key: value` 쌍을 `,` 로 구분
- key 는 보통 영문, value 는 어떤 타입이든 OK

---

## 2. 값 읽기 / 쓰기

### dot
```javascript
user.name             // "유찬"
user.name = "철수";   // 변경
user.newField = "x";  // 추가
delete user.email;    // 삭제
```

### bracket
```javascript
user["name"]          // "유찬"
const key = "age";
user[key]             // 25
```

dot 이 일반적. bracket 은 key 가 변수일 때 / 특수 문자 포함 시.

---

## 3. 중첩 객체

```javascript
const user = {
  name: "유찬",
  address: {
    city: "Seoul",
    zip: "12345",
  },
};

user.address.city          // "Seoul"
user.address.zip = "99999"; // 변경
```

### 깊이 접근 시 안전 — optional chaining
```javascript
user.address?.city           // 있으면 city, 없으면 undefined
user.profile?.bio?.short     // 단계마다 안전
```

`?.` 가 있으면 중간이 undefined 라도 에러 X.

---

## 4. 함수도 값 — method

```javascript
const calc = {
  add(a, b) {            // method 단축 형태
    return a + b;
  },
  subtract: function(a, b) {  // 일반 함수
    return a - b;
  },
};

calc.add(3, 5);          // 8
```

### method 안 this
```javascript
const user = {
  name: "유찬",
  greet() {
    console.log(`안녕, ${this.name}`);   // this = user
  },
};
user.greet();   // "안녕, 유찬"
```

---

## 5. spread (`...`) — 펴기 / 합치기

### 객체 복사
```javascript
const a = { x: 1, y: 2 };
const b = { ...a };       // { x: 1, y: 2 } 새 객체
```

### 합치기
```javascript
const a = { x: 1 };
const b = { y: 2 };
const c = { ...a, ...b }; // { x: 1, y: 2 }
```

### 덮어쓰기 (뒤가 우선)
```javascript
const defaults = { color: "red", size: "M" };
const user = { color: "blue" };
const final = { ...defaults, ...user };  // { color: "blue", size: "M" }
```

React 의 props 패턴에서 매우 자주.

---

## 6. 구조 분해 할당 (destructuring)

```javascript
const user = { name: "유찬", age: 25 };

const { name, age } = user;
console.log(name);   // "유찬"
console.log(age);    // 25
```

### 다른 이름으로
```javascript
const { name: userName } = user;
console.log(userName);   // "유찬"
```

### 기본값
```javascript
const { city = "Unknown" } = user;
console.log(city);   // "Unknown" (user 에 city 없음)
```

### 중첩
```javascript
const user = { address: { city: "Seoul" } };
const { address: { city } } = user;
console.log(city);   // "Seoul"
```

### 함수 매개변수에서
```javascript
function greet({ name, age }) {
  console.log(`${name}, ${age}`);
}

greet({ name: "유찬", age: 25 });
```

React 컴포넌트가 항상 이렇게 props 받음.

---

## 7. 객체 순회

### keys / values / entries
```javascript
const user = { name: "유찬", age: 25 };

Object.keys(user);      // ["name", "age"]
Object.values(user);    // ["유찬", 25]
Object.entries(user);   // [["name", "유찬"], ["age", 25]]
```

### for-of + entries
```javascript
for (const [key, value] of Object.entries(user)) {
  console.log(`${key}: ${value}`);
}
```

### for-in
```javascript
for (const key in user) {
  console.log(key, user[key]);
}
```

---

## 8. 객체 합치기 / 변환

### Object.assign — 합치기
```javascript
const a = { x: 1 };
const b = { y: 2 };
const merged = Object.assign({}, a, b);   // { x: 1, y: 2 }
```

spread 가 더 자주 — `{ ...a, ...b }` 와 같음.

### Object.fromEntries — 배열 → 객체
```javascript
const arr = [["a", 1], ["b", 2]];
Object.fromEntries(arr);   // { a: 1, b: 2 }
```

### entries 변환 패턴
```javascript
const user = { name: "유찬", age: 25 };

// 모든 값 대문자 (가능한 경우)
const upper = Object.fromEntries(
  Object.entries(user).map(([k, v]) =>
    [k, typeof v === "string" ? v.toUpperCase() : v]
  )
);
// { name: "유찬", age: 25 }  ← 한글은 그대로
```

---

## 9. 객체 복사 — 얕은 / 깊은

### 얕은 복사
```javascript
const a = { x: { y: 1 } };
const b = { ...a };

b.x.y = 999;
console.log(a.x.y);   // 999 (원본도 바뀜!)
```

`...` 는 1단계만 복사. 중첩 객체는 참조 공유.

### 깊은 복사
```javascript
const b = JSON.parse(JSON.stringify(a));    // 함수/Date 손실
// 또는
const b = structuredClone(a);                // 모던, 권장 (ES2022+)
```

---

## 10. 자주 만드는 패턴

### A. 객체 일부만 업데이트 (불변성)
```javascript
const user = { name: "유찬", age: 25 };

// ❌ 직접 수정
user.age = 26;

// ✅ 새 객체 만들기 (React 권장)
const updated = { ...user, age: 26 };
```

### B. 객체 일부 제거
```javascript
const { email, ...rest } = user;
// rest = email 제외한 나머지
```

### C. key 존재 확인
```javascript
"name" in user            // true
user.name !== undefined   // true
user.hasOwnProperty("name") // true
```

### D. 빈 객체 확인
```javascript
Object.keys(user).length === 0
```

### E. dynamic key
```javascript
const key = "name";
const user = {
  [key]: "유찬",     // 변수 값을 key 로
};
// = { name: "유찬" }
```

---

## 11. JSON 변환

서버랑 통신 / 저장할 때.

```javascript
const user = { name: "유찬", age: 25 };

// 객체 → 문자열
const json = JSON.stringify(user);
// '{"name":"유찬","age":25}'

// 예쁘게 (들여쓰기 2)
JSON.stringify(user, null, 2);

// 문자열 → 객체
const obj = JSON.parse(json);
console.log(obj.name);   // "유찬"
```

### JSON 의 한계
- 함수 / Date / undefined / Symbol 손실
- 순환 참조 에러

---

## 12. 객체 동결 — 변경 금지

```javascript
const user = Object.freeze({ name: "유찬" });
user.name = "철수";       // 조용히 실패 (strict 모드면 에러)
user.age = 25;            // 추가도 실패
```

설정 객체 같이 변경되면 안 되는 곳에 사용.

---

## 13. 함정

| 증상 | 원인 |
| --- | --- |
| `{...obj}.name` 으로 변경했는데 원본도 바뀜 | spread 는 1단계만 — 중첩 객체는 깊은 복사 필요 |
| `user.profile.name` 에서 에러 | `user.profile` 가 undefined. `?.` 사용 |
| `if (user.age)` 가 0 인데 false | 0 / "" / false 도 falsy. `!== undefined` 사용 |
| `JSON.stringify` 결과에 함수 안 나옴 | JSON 은 함수 미지원 |
| 객체 비교 `{a:1} === {a:1}` 가 false | 참조 비교. 깊은 비교는 별도 library |

---

## 14. 다음 단계

1. **[[arrays]]** — 배열과 method
2. **[[../async/promises-async-await]]** — 비동기

---

연결: [[../javascript|↑ hub]] · [[arrays]] · [[../functions/functions-basics]]
