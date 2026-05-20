---
title: "TS 기본 타입 — string / number / array / tuple / enum"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, types, basics]
---

# 기본 타입

**[[../typescript|↑ TS hub]]**

---

## 1. primitive

```typescript
const name: string = "유찬";
const age: number = 25;
const isReady: boolean = true;
const big: bigint = 9007199254740993n;
const id: symbol = Symbol("id");
```

대부분 `:` 없이 자동 추론:
```typescript
const name = "유찬";     // 자동으로 string
const age = 25;           // number
```

명시는 함수 인자 / 외부 입력에 주로.

---

## 2. null / undefined

```typescript
let x: string | null = null;
let y: number | undefined = undefined;

// strict 모드면 명시 안 하면 null/undefined 안 받음
let z: string = null;   // ❌ 에러
```

### 옵셔널 (?)
```typescript
type User = {
  name: string;
  age?: number;          // = number | undefined
};
```

---

## 3. array

### 두 가지 표기
```typescript
const arr1: string[] = ["a", "b"];        // 권장
const arr2: Array<string> = ["a", "b"];   // 동일
```

### 여러 타입 (union)
```typescript
const mixed: (string | number)[] = ["a", 1, "b", 2];
```

### readonly
```typescript
const fixed: readonly string[] = ["a", "b"];
fixed.push("c");   // ❌ 에러
```

또는:
```typescript
const fixed: ReadonlyArray<string> = ["a", "b"];
```

---

## 4. tuple — 길이 / 타입 고정

```typescript
let pair: [string, number] = ["x", 1];

pair[0]   // string
pair[1]   // number
pair[2]   // ❌ 에러 (인덱스 초과)
```

### 가변 (rest)
```typescript
let row: [string, ...number[]] = ["header", 1, 2, 3];
```

### 라벨링 (가독성)
```typescript
type Point = [x: number, y: number];
const p: Point = [10, 20];
```

---

## 5. enum (자주 피함)

```typescript
enum Status {
  Pending,    // 0
  Active,     // 1
  Done,       // 2
}

const s: Status = Status.Active;
```

### 문자열 enum
```typescript
enum Color {
  Red = "RED",
  Green = "GREEN",
}
```

### enum 의 단점
- 런타임 코드 생성 (번들 사이즈 ↑)
- tree-shaking 안 됨
- 옛 스타일

### 대안 — `as const` + union
```typescript
const STATUS = ["pending", "active", "done"] as const;
type Status = typeof STATUS[number];
// "pending" | "active" | "done"
```

또는:
```typescript
const STATUS = {
  PENDING: "pending",
  ACTIVE: "active",
  DONE: "done",
} as const;
type Status = typeof STATUS[keyof typeof STATUS];
```

**현업 권장**: `as const` 패턴. enum 안 씀.

---

## 6. any / unknown / never / void

### any — 검사 끔 (피하기)
```typescript
let x: any = "anything";
x.foo.bar();    // 에러 없음 (런타임 폭발)
```

`any` 는 TS 의 패배. 임시로만.

### unknown — 모름, 안전한 any
```typescript
let x: unknown = fetchData();

// 바로 사용 X
x.foo();        // ❌ 에러

// 검사 후
if (typeof x === "string") {
  x.toUpperCase();   // ✅ OK
}
```

외부 데이터 (API 응답) 에 권장.

### never — 절대 안 일어남
```typescript
function fail(): never {
  throw new Error("...");
}

function infinite(): never {
  while (true) {}
}
```

타입 추론에서 "불가능" 표현.

### void — return 없음
```typescript
function log(msg: string): void {
  console.log(msg);
}
```

return 값 사용 X.

---

## 7. literal type — 특정 값만

```typescript
let direction: "north" | "south" | "east" | "west";
direction = "north";    // ✅
direction = "up";       // ❌

let port: 80 | 443 | 8080 = 8080;
```

### 활용 — 옵션 제한
```typescript
function setStatus(s: "pending" | "active" | "done") {}
setStatus("active");   // ✅
setStatus("foo");      // ❌
```

---

## 8. as const — 리터럴 유지

```typescript
const colors = ["red", "blue"];
// 추론: string[]

const colors = ["red", "blue"] as const;
// 추론: readonly ["red", "blue"]

const config = {
  port: 80,
  host: "localhost",
} as const;
// 추론: { readonly port: 80; readonly host: "localhost" }
```

리터럴 그대로 → union 만들 때 유용.

---

## 9. 함수 타입

```typescript
function add(a: number, b: number): number {
  return a + b;
}

// 화살표
const add = (a: number, b: number): number => a + b;

// 추론 활용 (return type 생략)
const add = (a: number, b: number) => a + b;
```

### 함수 타입 별칭
```typescript
type AddFn = (a: number, b: number) => number;

const add: AddFn = (a, b) => a + b;
```

### 옵셔널 / default 인자
```typescript
function greet(name: string, greeting?: string) {
  return `${greeting ?? "Hi"}, ${name}`;
}

function greet(name: string, greeting = "Hi") {
  return `${greeting}, ${name}`;
}
```

### rest
```typescript
function sum(...nums: number[]) {
  return nums.reduce((a, b) => a + b, 0);
}
```

---

## 10. 타입 캐스팅 — as

타입 변경 강제. 조심.

```typescript
const el = document.getElementById("btn") as HTMLButtonElement;
el.disabled = true;

// 또는
const el = <HTMLButtonElement>document.getElementById("btn");
```

런타임 검증 X. 잘못된 캐스팅 → 폭발.

### 안전한 방법 — type guard
```typescript
const el = document.getElementById("btn");
if (el instanceof HTMLButtonElement) {
  el.disabled = true;
}
```

---

## 11. 자주 만드는 패턴

### A. 옵션 객체
```typescript
type Options = {
  retries?: number;
  timeout?: number;
};

function fetch(url: string, opts: Options = {}) {
  const retries = opts.retries ?? 3;
}
```

### B. 상태 (state machine)
```typescript
type State =
  | { kind: "idle" }
  | { kind: "loading"; startedAt: number }
  | { kind: "success"; data: User }
  | { kind: "error"; error: string };
```

### C. ID 타입 안전
```typescript
type UserId = string & { __brand: "UserId" };
type PostId = string & { __brand: "PostId" };

function fn(id: UserId) {}
fn("any-string");        // ❌ 에러 (brand 없음)
fn(userId);              // ✅
```

---

## 12. 함정

| 증상 | 원인 |
| --- | --- |
| `any` 가 퍼짐 | 한 곳에서 any → 전체 검사 안 됨 |
| 추론이 너무 넓음 | `as const` 또는 명시 |
| `null` 체크 매번 귀찮음 | `?.` / `??` 활용 |
| union 의 array 가 이상 | `(string | number)[]` 와 `string[] | number[]` 다름 |

---

## 13. 다음 단계

1. **[[objects-interfaces]]** — 객체 / interface
2. **[[unions-intersections]]** — union / intersection

---

연결: [[../typescript|↑ hub]] · [[objects-interfaces]]
