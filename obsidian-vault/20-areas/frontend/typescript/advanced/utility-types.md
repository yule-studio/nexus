---
title: "TS Utility Types — Partial / Pick / Omit / Record 등"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, utility-types, advanced]
---

# Utility Types

**[[../typescript|↑ TS hub]]**

> TS 가 내장으로 제공하는 타입 변환 도구. 현업 매일 사용.

---

## 1. Partial<T> — 모든 필드 optional

```typescript
type User = { name: string; age: number };

type PartialUser = Partial<User>;
// { name?: string; age?: number }

function update(user: User, patch: Partial<User>): User {
  return { ...user, ...patch };
}

update(user, { age: 26 });        // OK
update(user, {});                  // OK
```

---

## 2. Required<T> — 모든 필드 필수

```typescript
type Config = {
  port?: number;
  host?: string;
};

type FullConfig = Required<Config>;
// { port: number; host: string }
```

---

## 3. Readonly<T> — 모든 필드 readonly

```typescript
type User = { name: string; age: number };
type ReadOnlyUser = Readonly<User>;

const u: ReadOnlyUser = { name: "유찬", age: 25 };
u.name = "철수";   // ❌
```

---

## 4. Pick<T, K> — 특정 필드만

```typescript
type User = {
  id: string;
  name: string;
  email: string;
  password: string;
};

type PublicUser = Pick<User, "id" | "name" | "email">;
// { id: string; name: string; email: string }
```

API 응답 / DB → public 변환 시 자주.

---

## 5. Omit<T, K> — 특정 필드 제외

```typescript
type CreateUser = Omit<User, "id">;
// id 빼고 다 (생성 시 id 가 서버 부여)

type PublicUser = Omit<User, "password">;
// password 빼고
```

`Pick` 의 반대.

---

## 6. Record<K, V> — key/value 매핑

```typescript
type Status = "pending" | "active" | "done";

type StatusLabels = Record<Status, string>;
// {
//   pending: string;
//   active: string;
//   done: string;
// }

const labels: StatusLabels = {
  pending: "대기",
  active: "활성",
  done: "완료",
};
```

### 동적 키
```typescript
type Cache = Record<string, User>;

const cache: Cache = {};
cache.foo = user1;
cache.bar = user2;
```

---

## 7. Extract<T, U> / Exclude<T, U>

union 에서 일부 가져오기 / 빼기.

```typescript
type Status = "pending" | "active" | "done" | "error";

type Active = Extract<Status, "active" | "done">;
// "active" | "done"

type NotError = Exclude<Status, "error">;
// "pending" | "active" | "done"
```

---

## 8. NonNullable<T> — null / undefined 제거

```typescript
type Name = string | null | undefined;
type SafeName = NonNullable<Name>;   // string
```

### 활용
```typescript
function safeFilter<T>(arr: (T | null | undefined)[]): NonNullable<T>[] {
  return arr.filter((x): x is NonNullable<T> => x != null);
}
```

---

## 9. ReturnType<F> — 함수의 반환 타입

```typescript
function getUser() {
  return { name: "유찬", age: 25 };
}

type User = ReturnType<typeof getUser>;
// { name: string; age: number }
```

API 함수의 응답 타입을 자동 추출 → 별도 type 정의 불필요.

---

## 10. Parameters<F> — 함수의 인자 타입 (tuple)

```typescript
function fn(a: string, b: number) {}

type Args = Parameters<typeof fn>;
// [string, number]
```

### 활용 — 같은 인자로 다른 함수
```typescript
function logCall<F extends (...args: any[]) => any>(
  fn: F,
  ...args: Parameters<F>
): ReturnType<F> {
  console.log("calling with", args);
  return fn(...args);
}
```

---

## 11. Awaited<T> — Promise 풀기

```typescript
type Data = Awaited<Promise<User>>;   // User
type Data = Awaited<Promise<Promise<User>>>;   // User (재귀)

// 함수의 async 반환 타입
type FetchResult = Awaited<ReturnType<typeof fetchUser>>;
```

---

## 12. ConstructorParameters / InstanceType

```typescript
class User {
  constructor(public name: string, public age: number) {}
}

type Args = ConstructorParameters<typeof User>;   // [string, number]
type Inst = InstanceType<typeof User>;             // User
```

---

## 13. string 조작 utility types

```typescript
type Upper = Uppercase<"hello">;            // "HELLO"
type Lower = Lowercase<"HELLO">;            // "hello"
type Cap = Capitalize<"hello">;             // "Hello"
type Uncap = Uncapitalize<"Hello">;         // "hello"
```

### 활용
```typescript
type EventName = `on${Capitalize<string>}`;
// "onClick", "onChange" 등

const ev: EventName = "onClick";   // OK
const ev: EventName = "click";     // ❌
```

---

## 14. 자주 만드는 조합

### A. DeepPartial
```typescript
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

type DPUser = DeepPartial<User>;
// 모든 중첩 객체도 optional
```

### B. NonEmpty array
```typescript
type NonEmpty<T> = [T, ...T[]];

const a: NonEmpty<string> = [];       // ❌ 빈 배열
const b: NonEmpty<string> = ["a"];    // OK
```

### C. Either (둘 중 하나)
```typescript
type Either<A, B> =
  | (A & { [K in Exclude<keyof B, keyof A>]?: never })
  | (B & { [K in Exclude<keyof A, keyof B>]?: never });

// 사용 예: { value: string } 또는 { defaultValue: string } 둘 중 하나
```

### D. Mutable (readonly 제거)
```typescript
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
```

### E. Strict Omit
TS 의 Omit 은 key 가 T 에 없어도 에러 안 남:
```typescript
type StrictOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
```

---

## 15. 자주 쓰는 React 패턴

### A. 기존 Props 확장
```typescript
type ButtonProps = ComponentProps<"button"> & {
  variant?: "primary" | "secondary";
};
```

### B. Children 만 제외
```typescript
type WithoutChildren<T> = Omit<T, "children">;
```

### C. ref 다루기
```typescript
type InputProps = ComponentPropsWithoutRef<"input">;
type InputRef = ElementRef<"input">;
```

자세히 [[../react/component-typing]].

---

## 16. 함정

| 증상 | 원인 |
| --- | --- |
| `Partial<T>` 가 nested 까지 안 됨 | DeepPartial 직접 구현 |
| `Omit` 의 K 검사 안 됨 | `StrictOmit` 또는 `keyof T` 명시 |
| utility 가 너무 복잡 | 보통 zod 같은 schema-first 가 더 깔끔 |
| ReturnType 가 Promise 그대로 | `Awaited<ReturnType<...>>` |

---

## 17. 다음 단계

1. **[[type-guards-narrowing]]** — 타입 좁힘
2. **[[mapped-conditional]]** — mapped / conditional types
3. **[[../react/component-typing]]** — React Props 타입

---

연결: [[../typescript|↑ hub]] · [[../types/generics]]
