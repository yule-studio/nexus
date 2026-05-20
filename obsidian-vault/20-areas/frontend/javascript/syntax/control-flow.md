---
title: "JS 흐름 제어 — if / for / while"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, beginner, control-flow]
---

# 흐름 제어

**[[../javascript|↑ JavaScript hub]]**

---

## 1. if / else

```javascript
const age = 20;

if (age >= 20) {
  console.log("성인");
} else if (age >= 14) {
  console.log("청소년");
} else {
  console.log("어린이");
}
```

### 단축 형태 (1 줄)
```javascript
if (age >= 20) console.log("성인");
```

중괄호 `{}` 권장 — 줄 추가 시 안전.

---

## 2. 삼항 연산자 (ternary)

```javascript
const message = age >= 20 ? "성인" : "미성년";

// = 위 if/else 와 같음:
// let message;
// if (age >= 20) message = "성인";
// else message = "미성년";
```

### 중첩 (피하기)
```javascript
const msg = age >= 20 ? "성인" : age >= 14 ? "청소년" : "어린이";
// 가능하지만 가독성 ↓. if/else 가 나음.
```

---

## 3. switch

```javascript
const day = "월";

switch (day) {
  case "월":
  case "화":
  case "수":
  case "목":
  case "금":
    console.log("평일");
    break;
  case "토":
  case "일":
    console.log("주말");
    break;
  default:
    console.log("알 수 없음");
}
```

### 주의
- `break` 없으면 다음 case 까지 실행 (fall-through)
- `default` 는 어느 case 도 안 맞으면

### 객체 매핑 (요즘 더 권장)
```javascript
const dayType = {
  월: "평일", 화: "평일", 수: "평일", 목: "평일", 금: "평일",
  토: "주말", 일: "주말",
};

console.log(dayType[day] ?? "알 수 없음");
```

---

## 4. for — 반복

### 기본 for
```javascript
for (let i = 0; i < 5; i++) {
  console.log(i);   // 0, 1, 2, 3, 4
}
```

- `let i = 0` — 시작
- `i < 5` — 조건 (참인 동안 반복)
- `i++` — 매 회 실행

### for...of — 배열 / 문자열
```javascript
const list = ["a", "b", "c"];

for (const item of list) {
  console.log(item);   // a, b, c
}

for (const ch of "hi") {
  console.log(ch);     // h, i
}
```

### for...in — 객체 (또는 배열 인덱스)
```javascript
const user = { name: "유찬", age: 25 };

for (const key in user) {
  console.log(key, user[key]);
  // name 유찬
  // age 25
}
```

---

## 5. while / do-while

```javascript
let i = 0;
while (i < 5) {
  console.log(i);
  i++;
}
```

### do-while (최소 1번 실행)
```javascript
let i = 0;
do {
  console.log(i);
  i++;
} while (i < 5);
```

### 무한 루프
```javascript
while (true) {
  if (조건) break;
  // 작업
}
```

---

## 6. break / continue

### break — 반복 즉시 종료
```javascript
for (let i = 0; i < 10; i++) {
  if (i === 5) break;
  console.log(i);     // 0~4
}
```

### continue — 다음 회로 건너뛰기
```javascript
for (let i = 0; i < 5; i++) {
  if (i === 2) continue;
  console.log(i);     // 0, 1, 3, 4
}
```

---

## 7. 배열 method 로 반복 (모던 권장)

`for` 보다 array method 가 깔끔.

### forEach — 단순 반복
```javascript
[1, 2, 3].forEach((n) => {
  console.log(n);
});
```

### map — 변환
```javascript
const doubled = [1, 2, 3].map((n) => n * 2);  // [2, 4, 6]
```

### filter — 거름
```javascript
const even = [1, 2, 3, 4].filter((n) => n % 2 === 0);  // [2, 4]
```

### reduce — 누적
```javascript
const sum = [1, 2, 3].reduce((acc, n) => acc + n, 0);  // 6
```

### find — 찾기
```javascript
const found = [1, 2, 3].find((n) => n > 1);   // 2 (첫 번째)
```

### some / every
```javascript
[1, 2, 3].some((n) => n > 2);   // true (하나라도)
[1, 2, 3].every((n) => n > 0);  // true (전부)
```

자세히 [[../objects-arrays/arrays]].

---

## 8. try / catch — 에러 처리

```javascript
try {
  const result = riskyOperation();
} catch (error) {
  console.error("에러:", error.message);
}
```

### finally — 성공 / 실패 무관 항상 실행
```javascript
try {
  // ...
} catch (e) {
  // 에러 처리
} finally {
  // 정리 (DB 연결 닫기 등)
}
```

### 에러 던지기
```javascript
function divide(a, b) {
  if (b === 0) throw new Error("0 으로 나눌 수 없음");
  return a / b;
}

try {
  divide(10, 0);
} catch (e) {
  console.log(e.message);   // "0 으로 나눌 수 없음"
}
```

---

## 9. 자주 만나는 패턴

### A. 배열에서 조건 맞는 첫 항목
```javascript
const user = users.find((u) => u.id === 42);
```

### B. 조건에 따라 다른 메시지
```javascript
function greet(time) {
  if (time < 12) return "좋은 아침";
  if (time < 18) return "좋은 오후";
  return "좋은 저녁";
}
```

`if + return` 패턴 — early return 으로 중첩 줄임.

### C. 1~10 출력
```javascript
for (let i = 1; i <= 10; i++) {
  console.log(i);
}
```

### D. 배열 합계
```javascript
const total = numbers.reduce((sum, n) => sum + n, 0);
```

### E. 객체 키 / 값 순회
```javascript
const user = { name: "유찬", age: 25 };

for (const [key, value] of Object.entries(user)) {
  console.log(`${key}: ${value}`);
}
```

---

## 10. 함정

| 증상 | 원인 |
| --- | --- |
| `for...in` 으로 배열 → 이상한 순서 / index 가 문자열 | `for...of` 또는 `forEach` 사용 |
| `==` 가 의도와 다른 결과 | 항상 `===` 사용 |
| 무한 루프 | 종료 조건 확인. `i++` 빠뜨림 등 |
| `break` 가 안 먹음 (중첩 루프) | 안쪽 break 는 안쪽만. 라벨 또는 함수로 분리 |
| switch 의 fall-through | 매 case 끝에 `break` 또는 객체 매핑 |

---

## 11. 다음 단계

1. **[[../functions/functions-basics]]** — 함수
2. **[[../objects-arrays/arrays]]** — 배열 method 깊이

---

연결: [[../javascript|↑ hub]] · [[variables-types]] · [[../functions/functions-basics]]
