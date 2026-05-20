---
title: "TS Union / Intersection / Literal"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, union, intersection, literal]
---

# Union / Intersection / Literal

**[[../typescript|↑ TS hub]]**

> `|` (OR) / `&` (AND) / 특정 값. TS 의 표현력 핵심.

---

## 1. Union (`|`) — 여러 타입 중 하나

```typescript
type Id = string | number;

let id: Id = "abc";
id = 42;          // OK
id = true;        // ❌
```

### 객체 union
```typescript
type Loading = { state: "loading" };
type Success = { state: "success"; data: User };
type Failure = { state: "failure"; error: string };

type Status = Loading | Success | Failure;
```

→ [[#discriminated-union]] 핵심 패턴.

### array of union
```typescript
type Mixed = (string | number)[];
const arr: Mixed = ["a", 1, "b", 2];
```

---

## 2. Intersection (`&`) — 합치기 (AND)

```typescript
type WithName = { name: string };
type WithAge = { age: number };

type Person = WithName & WithAge;
// { name: string; age: number }

const p: Person = { name: "유찬", age: 25 };
```

### 함수 시그니처 합치기
```typescript
type Fn = ((a: string) => string) & ((b: number) => number);
// overload
```

---

## 3. Literal Type — 특정 값만

```typescript
type Direction = "up" | "down" | "left" | "right";

let d: Direction = "up";       // OK
d = "diagonal";                // ❌
```

### 숫자 literal
```typescript
type HttpStatus = 200 | 404 | 500;

function handle(s: HttpStatus) {}
handle(200);   // OK
handle(401);   // ❌
```

### boolean literal
```typescript
type True = true;
const x: True = true;   // OK
const y: True = false;  // ❌
```

---

## 4. discriminated union — 가장 강력한 패턴

각 union member 에 공통 discriminator field.

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number }
  | { kind: "rect"; width: number; height: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;   // s.radius 자동 추론
    case "square": return s.size ** 2;
    case "rect":   return s.width * s.height;
  }
}
```

### 핵심
- `kind` 같은 literal field 가 discriminator
- `switch` / `if` 안에서 TS 가 자동으로 타입 좁힘
- 빠진 case → exhaustive 검사 가능

### exhaustiveness check
```typescript
function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return ...;
    case "square": return ...;
    case "rect":   return ...;
    default:
      const _exhaustive: never = s;   // 모든 case 처리 시 never
      throw new Error(`Unknown kind`);
  }
}
```

새 Shape 추가하면 default 에서 컴파일 에러 → 빠진 case 발견.

---

## 5. union of types vs union of values

```typescript
// 값의 union
let id: "user" | "admin" = "user";

// 타입의 union
type Id = string | number;
```

용도 다름. 둘 다 자주.

---

## 6. 자주 만드는 union 패턴

### A. nullable
```typescript
type User = { name: string };
type MaybeUser = User | null;

const u: MaybeUser = await fetchUser();
if (u) {
  console.log(u.name);   // 좁힘 됨
}
```

### B. 결과 / 에러
```typescript
type Result<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };

async function divide(a: number, b: number): Promise<Result<number>> {
  if (b === 0) return { ok: false, error: "Div by zero" };
  return { ok: true, data: a / b };
}

const r = await divide(10, 2);
if (r.ok) {
  console.log(r.data);
} else {
  console.log(r.error);
}
```

Rust 의 Result 비슷한 패턴. exception 보다 안전.

### C. 이벤트
```typescript
type AppEvent =
  | { type: "user_login"; userId: string }
  | { type: "page_view"; path: string; duration: number }
  | { type: "purchase"; productId: string; price: number };

function track(event: AppEvent) {
  switch (event.type) {
    case "purchase": send(event.price);   // 자동 추론
    ...
  }
}
```

---

## 7. 자주 만드는 intersection 패턴

### A. mixin
```typescript
type Timestamps = { createdAt: Date; updatedAt: Date };
type SoftDelete = { deletedAt: Date | null };

type User = { name: string } & Timestamps & SoftDelete;
```

### B. branded type (id 안전)
```typescript
type Brand<T, B> = T & { __brand: B };

type UserId = Brand<string, "UserId">;
type PostId = Brand<string, "PostId">;

const userId = "abc" as UserId;
const postId = "xyz" as PostId;

function getUser(id: UserId) {}
getUser(userId);    // OK
getUser(postId);    // ❌ 다른 brand
```

---

## 8. literal 좁힘 (narrowing)

```typescript
type Mode = "edit" | "view";

function setMode(m: Mode) {
  if (m === "edit") {
    // m 은 "edit"
  } else {
    // m 은 "view"
  }
}
```

### never 로 강제
```typescript
function impossible(x: never): never {
  throw new Error(`Unexpected: ${x}`);
}

function setMode(m: Mode) {
  if (m === "edit") return ...;
  if (m === "view") return ...;
  return impossible(m);    // 새 Mode 추가 시 컴파일 에러
}
```

---

## 9. union 좁히기 — typeof / in / instanceof

```typescript
function fn(x: string | number) {
  if (typeof x === "string") {
    x.toUpperCase();   // string 추론
  } else {
    x.toFixed(2);      // number 추론
  }
}

type Cat = { meow(): void };
type Dog = { bark(): void };

function speak(a: Cat | Dog) {
  if ("meow" in a) a.meow();
  else a.bark();
}

function handle(e: Error | string) {
  if (e instanceof Error) console.log(e.stack);
  else console.log(e);
}
```

자세히 [[../advanced/type-guards-narrowing]].

---

## 10. union 의 array vs array of union

```typescript
type A = string[] | number[];     // 다 string 또는 다 number
type B = (string | number)[];      // 섞임 OK

const a: A = ["a", "b"];           // OK
const a: A = [1, 2];                // OK
const a: A = ["a", 1];              // ❌

const b: B = ["a", 1];              // OK
```

---

## 11. 자주 만나는 함정

| 증상 | 원인 |
| --- | --- |
| union 의 객체에 .x 접근 안 됨 | 공통 field 만 직접 접근 — 좁히기 필요 |
| intersection 의 conflict | `{a: string} & {a: number}` = `never` |
| literal 가 자동 추론 안 됨 | `as const` 또는 명시 |
| switch 다 처리했는데 컴파일러가 모른다 | exhaustive check (never 패턴) 추가 |

---

## 12. 다음 단계

1. **[[generics]]** — 제네릭
2. **[[../advanced/utility-types]]** — Partial / Pick / Omit

---

연결: [[../typescript|↑ hub]] · [[objects-interfaces]] · [[generics]]
