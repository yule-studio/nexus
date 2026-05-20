---
title: "TS Type Guards / Narrowing"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, type-guards, narrowing, advanced]
---

# Type Guards / Narrowing

**[[../typescript|↑ TS hub]]**

> union 타입을 좁혀서 안전하게 사용. 현업 매일 사용 패턴.

---

## 1. 기본 narrowing — typeof

```typescript
function fn(x: string | number) {
  if (typeof x === "string") {
    x.toUpperCase();   // x 는 string 으로 좁혀짐
  } else {
    x.toFixed(2);      // x 는 number
  }
}
```

### typeof 가 알아내는 것
- `"string"`, `"number"`, `"boolean"`, `"bigint"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`

객체 / 배열 구분 X (둘 다 "object"). 다른 방법 필요.

---

## 2. instanceof — class

```typescript
function handle(e: Error | string) {
  if (e instanceof Error) {
    console.log(e.stack);     // Error 메서드 사용 가능
  } else {
    console.log(e);            // string
  }
}
```

class 만 가능 (function 으로 만든 거 포함).

---

## 3. in — 객체 property 존재 확인

```typescript
type Cat = { meow(): void };
type Dog = { bark(): void };

function speak(a: Cat | Dog) {
  if ("meow" in a) {
    a.meow();    // Cat
  } else {
    a.bark();    // Dog
  }
}
```

객체 union 좁힘에 유용.

---

## 4. discriminated union (가장 강력)

각 union member 에 공통 literal field.

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number }
  | { kind: "rect"; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;   // 자동 좁힘
    case "square": return s.size ** 2;
    case "rect":   return s.w * s.h;
  }
}
```

### exhaustive check (모든 case 확인)
```typescript
function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return ...;
    case "square": return ...;
    case "rect":   return ...;
    default:
      const _: never = s;     // 새 Shape 추가 시 컴파일 에러
      throw new Error("unreachable");
  }
}
```

`Shape` 에 `triangle` 추가하면 `_: never` 에서 에러 → 처리 누락 발견.

---

## 5. user-defined type guard

```typescript
function isString(x: unknown): x is string {
  return typeof x === "string";
}

const x: unknown = await fetchData();
if (isString(x)) {
  x.toUpperCase();   // string
}
```

`x is string` = "true 반환하면 x 는 string" 알려줌.

### 객체 type guard
```typescript
type User = { name: string; email: string };

function isUser(x: unknown): x is User {
  return (
    typeof x === "object" &&
    x !== null &&
    "name" in x &&
    "email" in x &&
    typeof (x as any).name === "string" &&
    typeof (x as any).email === "string"
  );
}

const data: unknown = await fetch(...).then(r => r.json());
if (isUser(data)) {
  console.log(data.name);   // 안전
}
```

복잡 — 실무는 zod 권장 (§ 9).

---

## 6. assertion function

```typescript
function assertString(x: unknown): asserts x is string {
  if (typeof x !== "string") throw new Error("Not string");
}

const x: unknown = ...;
assertString(x);
x.toUpperCase();   // 이후 string 으로 사용 가능
```

`asserts x is string` = "이 함수가 throw 안 했으면 x 는 string".

---

## 7. truthy / nullish 좁힘

```typescript
function fn(x: string | null | undefined) {
  if (x) {
    x.length;   // string (null/undefined 제거)
  }
}

// 더 명시적
if (x != null) { ... }
if (x !== undefined) { ... }
```

### filter 로 null 제거
```typescript
const arr: (string | null)[] = ["a", null, "b"];
const cleaned = arr.filter((x): x is string => x !== null);
// string[]
```

---

## 8. Array.isArray

```typescript
function fn(x: string | string[]) {
  if (Array.isArray(x)) {
    x.length;     // array
  } else {
    x.length;     // string
  }
}
```

---

## 9. zod — 런타임 검증 + 타입 자동 생성

수동 type guard 의 대안. 현업 표준.

```bash
npm install zod
```

```typescript
import { z } from "zod";

const UserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().int().positive().optional(),
});

type User = z.infer<typeof UserSchema>;
// { name: string; email: string; age?: number }

const data: unknown = await fetch(...).then(r => r.json());

const result = UserSchema.safeParse(data);
if (result.success) {
  console.log(result.data.name);   // User
} else {
  console.log(result.error);
}
```

### 장점
- 런타임 검증 + 컴파일 타임 타입 한 번에
- 에러 메시지 자세함
- 변환 가능 (transform)

---

## 10. discriminated union + 사용자 정의

```typescript
type ApiResponse<T> =
  | { ok: true; data: T }
  | { ok: false; error: string };

function isSuccess<T>(r: ApiResponse<T>): r is { ok: true; data: T } {
  return r.ok;
}

const res = await fetchUser();
if (isSuccess(res)) {
  console.log(res.data);
}
```

또는 더 간단히 `if (res.ok)` 만 해도 자동 좁힘.

---

## 11. 자주 만드는 type guard

### A. 객체 check
```typescript
function isObject(x: unknown): x is Record<string, unknown> {
  return typeof x === "object" && x !== null && !Array.isArray(x);
}
```

### B. 에러 객체
```typescript
function isError(x: unknown): x is Error {
  return x instanceof Error;
}

function getErrorMessage(x: unknown): string {
  if (isError(x)) return x.message;
  if (typeof x === "string") return x;
  return "Unknown error";
}
```

### C. truthy 필터 (배열)
```typescript
function isTruthy<T>(x: T | null | undefined | false | "" | 0): x is T {
  return Boolean(x);
}

const arr = [1, null, "a", undefined, 0, "b"];
const truthy = arr.filter(isTruthy);
// string | number[] (truthy 값만)
```

### D. enum 값 검증
```typescript
const STATUSES = ["pending", "active", "done"] as const;
type Status = typeof STATUSES[number];

function isStatus(x: unknown): x is Status {
  return typeof x === "string" && (STATUSES as readonly string[]).includes(x);
}

const input: unknown = "active";
if (isStatus(input)) {
  // input 은 Status
}
```

---

## 12. 함정

| 증상 | 원인 |
| --- | --- |
| 좁힘이 풀림 | callback 안에서 다시 검사 / let → const |
| typeof null === "object" | null 별도 검사: `x !== null` |
| Array.isArray 후에 T[] 추론 못 함 | TS 가 readonly tuple 가능. `as T[]` |
| `instanceof` 가 다른 realm 의 객체 안 잡힘 | iframe / worker — duck-typing 으로 |

---

## 13. 다음 단계

1. **[[mapped-conditional]]** — mapped / conditional types
2. **[[../react/component-typing]]** — React Props

---

연결: [[../typescript|↑ hub]] · [[utility-types]] · [[../types/unions-intersections]]
