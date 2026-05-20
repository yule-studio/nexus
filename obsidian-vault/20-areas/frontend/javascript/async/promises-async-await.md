---
title: "Promise / async / await / fetch"
kind: knowledge
project: frontend
status: current
tags: [frontend, javascript, beginner, async, promise, fetch]
---

# 비동기 — Promise / async / await

**[[../javascript|↑ JavaScript hub]]**

> 시간 걸리는 작업 (네트워크 요청, 파일 읽기 등) 을 처리.

---

## 1. 동기 vs 비동기

### 동기 (synchronous) — 줄 서기
```javascript
console.log("1");
console.log("2");
console.log("3");
// 1, 2, 3 순서대로
```

### 비동기 (asynchronous) — 미래에 끝남
```javascript
console.log("1");
setTimeout(() => console.log("2"), 1000);   // 1초 뒤
console.log("3");
// 1, 3, 2  ← setTimeout 은 비동기
```

---

## 2. Promise — "미래의 값" 표현

```javascript
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("성공!");
    // 실패면: reject("실패");
  }, 1000);
});
```

Promise = "1초 뒤에 결과가 올 것" 같은 약속.

---

## 3. then / catch / finally — 옛 방식

```javascript
fetch("/api/users")
  .then((response) => response.json())     // 성공 → 다음
  .then((data) => console.log(data))
  .catch((error) => console.error(error))  // 실패
  .finally(() => console.log("done"));     // 성공/실패 무관
```

읽기 어려움. 요즘은 `async/await` 권장.

---

## 4. async / await — 모던 방식

```javascript
async function fetchUsers() {
  const response = await fetch("/api/users");
  const data = await response.json();
  console.log(data);
}

fetchUsers();
```

**`async` 키워드 붙은 함수 안에서만 `await` 사용 가능.**

### 작동 흐름
1. `await fetch(...)` — fetch 끝날 때까지 기다림
2. `response` 받음
3. `await response.json()` — 파싱 기다림
4. `data` 받음

코드가 동기처럼 위→아래로 읽힘.

---

## 5. 에러 처리 — try / catch

```javascript
async function fetchUsers() {
  try {
    const response = await fetch("/api/users");
    if (!response.ok) throw new Error("Server error");
    const data = await response.json();
    return data;
  } catch (error) {
    console.error("실패:", error.message);
    return [];
  }
}
```

---

## 6. fetch — HTTP 요청

### GET
```javascript
const res = await fetch("/api/users");
const data = await res.json();
```

### POST (JSON 보내기)
```javascript
const res = await fetch("/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "유찬", age: 25 }),
});
const data = await res.json();
```

### PUT / DELETE
```javascript
await fetch(`/api/users/${id}`, { method: "PUT", body: JSON.stringify(...) });
await fetch(`/api/users/${id}`, { method: "DELETE" });
```

### 응답 확인
```javascript
const res = await fetch(url);

res.status        // 200, 404, 500 ...
res.ok            // 200~299 면 true
res.statusText    // "OK", "Not Found"
res.headers       // Headers
```

### 응답 본문 처리
```javascript
const data = await res.json();    // JSON
const text = await res.text();    // 텍스트
const blob = await res.blob();    // 파일
```

---

## 7. 헤더 추가

### 인증 토큰
```javascript
const res = await fetch("/api/me", {
  headers: { Authorization: `Bearer ${token}` },
});
```

### 다른 헤더
```javascript
const res = await fetch(url, {
  headers: {
    "Content-Type": "application/json",
    "X-Custom-Header": "value",
  },
});
```

---

## 8. 여러 비동기 동시에 — Promise.all

### 순차 (느림)
```javascript
const user = await fetchUser();        // 1초
const posts = await fetchPosts();       // 1초
const comments = await fetchComments(); // 1초
// 총 3초
```

### 병렬 — Promise.all
```javascript
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments(),
]);
// 총 1초 (가장 느린 것 기준)
```

### 일부 실패해도 OK — Promise.allSettled
```javascript
const results = await Promise.allSettled([
  fetchA(),
  fetchB(),
]);

results.forEach((r) => {
  if (r.status === "fulfilled") console.log(r.value);
  else console.error(r.reason);
});
```

### 가장 빠른 하나 — Promise.race
```javascript
const fastest = await Promise.race([
  fetchFromServer1(),
  fetchFromServer2(),
]);
```

---

## 9. setTimeout / setInterval

### setTimeout — N 초 뒤 1번
```javascript
setTimeout(() => {
  console.log("3초 뒤");
}, 3000);
```

### setInterval — N 초마다 반복
```javascript
const id = setInterval(() => {
  console.log("매 초");
}, 1000);

// 취소
clearInterval(id);
```

### sleep (await 가능)
```javascript
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function fn() {
  console.log("1");
  await sleep(1000);
  console.log("2 (1초 뒤)");
}
```

---

## 10. 자주 만드는 패턴

### A. 기본 API 호출
```javascript
async function getUser(id) {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

// 사용
const user = await getUser(42);
```

### B. 로딩 / 에러 상태 (React 안 자주)
```javascript
let isLoading = true;
let error = null;
let data = null;

try {
  data = await fetchData();
} catch (e) {
  error = e;
} finally {
  isLoading = false;
}
```

### C. 디바운스 (자주 호출 막기)
```javascript
function debounce(fn, delay) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}

const search = debounce((query) => fetchResults(query), 300);

// 사용자가 빠르게 타이핑해도 300ms 후 마지막 입력만 fetch
```

### D. 재시도 (retry)
```javascript
async function retry(fn, times = 3) {
  for (let i = 0; i < times; i++) {
    try {
      return await fn();
    } catch (e) {
      if (i === times - 1) throw e;
      await sleep(1000);
    }
  }
}

const data = await retry(() => fetchData(), 5);
```

### E. 캐싱 (간단)
```javascript
const cache = new Map();

async function cachedFetch(url) {
  if (cache.has(url)) return cache.get(url);
  const data = await fetch(url).then((r) => r.json());
  cache.set(url, data);
  return data;
}
```

### F. AbortController — 요청 취소
```javascript
const controller = new AbortController();

fetch(url, { signal: controller.signal })
  .then((res) => res.json())
  .catch((e) => {
    if (e.name === "AbortError") console.log("취소됨");
  });

// 5초 뒤 취소
setTimeout(() => controller.abort(), 5000);
```

---

## 11. async 함수의 return

```javascript
async function fn() {
  return 42;
}

fn();         // Promise (resolved with 42)
await fn();   // 42
fn().then(v => console.log(v));   // 42
```

→ `async` 함수는 항상 Promise 반환.

---

## 12. await 잘못 사용

### ❌ 일반 함수 안 await
```javascript
function fn() {
  await fetch(url);   // 에러 — fn 가 async 아님
}
```

### ✅
```javascript
async function fn() {
  await fetch(url);
}
```

### ❌ await 안 함 (Promise 채로 사용)
```javascript
const data = fetch(url);    // Promise 객체
console.log(data);          // [object Promise]
```

### ✅
```javascript
const res = await fetch(url);
const data = await res.json();
```

---

## 13. 함정

| 증상 | 원인 |
| --- | --- |
| `await` 가 에러 | 함수에 `async` 안 붙임 |
| Promise 채로 콘솔에 출력 | `await` 안 함 또는 then 안 받음 |
| `for` 안 await 가 병렬 안 됨 | `for` 의 each 가 순차. 병렬은 `Promise.all` 로 |
| 네트워크 에러 (CORS) | 서버에 CORS 헤더 없음 (서버 설정) |
| `res.json()` 가 fail | 본문이 JSON 아님 — `res.text()` 로 확인 |

---

## 14. CORS 간단 설명

서버가 "이 출처는 접근 가능" 헤더를 응답에 줘야 브라우저가 허용.

```
Access-Control-Allow-Origin: *
```

내가 만든 서버면 헤더 설정. 외부 API 면 서버 측에 요청해야.

dev 환경에서 우회는 Vite/Next 의 proxy 사용.

---

## 15. 다음 단계

1. **[[../modules/modules-and-imports]]** — 파일 분리 / import
2. **[[../../react/http/]]** — React 의 fetch 패턴
3. **[[../../react/server-state/]]** — TanStack Query (캐싱 자동)

---

연결: [[../javascript|↑ hub]] · [[../functions/functions-basics]]
