---
title: "TS Generics — 제네릭"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, generics]
---

# Generics — 제네릭

**[[../typescript|↑ TS hub]]**

> "타입을 변수처럼" — 함수 / 클래스 / 타입이 여러 타입에서 작동.

---

## 1. 가장 단순한 예

### 제네릭 없으면
```typescript
function identity(value: any): any {
  return value;
}

const x = identity("hi");   // any (자동 완성 X)
```

### 제네릭으로
```typescript
function identity<T>(value: T): T {
  return value;
}

const x = identity("hi");   // string (자동 추론)
const y = identity(42);     // number
```

`<T>` = 타입 변수. 호출 시 자동 채워짐.

---

## 2. 명시 vs 추론

```typescript
identity("hi");          // 추론: T = string
identity<string>("hi");  // 명시
```

대부분 추론으로 충분. 명시는 추론 불가 시.

---

## 3. 여러 타입 변수

```typescript
function pair<A, B>(a: A, b: B): [A, B] {
  return [a, b];
}

const p = pair("name", 25);   // [string, number]
```

---

## 4. 제약 (`extends`)

```typescript
function len<T extends { length: number }>(x: T): number {
  return x.length;
}

len("hi");        // 3 (string 은 length 있음)
len([1, 2, 3]);   // 3
len(42);          // ❌ number 는 length 없음
```

---

## 5. 제네릭 type

```typescript
type Box<T> = { value: T };

const a: Box<string> = { value: "hi" };
const b: Box<number> = { value: 42 };
```

### 여러 변수
```typescript
type Pair<A, B> = { first: A; second: B };

const p: Pair<string, number> = { first: "a", second: 1 };
```

---

## 6. 제네릭 interface

```typescript
interface Repository<T> {
  find(id: string): T | null;
  save(item: T): void;
  delete(id: string): void;
}

class UserRepo implements Repository<User> {
  find(id: string): User | null { ... }
  save(item: User): void { ... }
  delete(id: string): void { ... }
}
```

---

## 7. 제네릭 class

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
}

const numStack = new Stack<number>();
numStack.push(1);
const s = numStack.pop();   // number | undefined
```

---

## 8. 기본값 (default)

```typescript
type Box<T = string> = { value: T };

const a: Box = { value: "hi" };       // T = string
const b: Box<number> = { value: 42 };
```

---

## 9. 제네릭 + Promise

```typescript
async function fetchJson<T>(url: string): Promise<T> {
  const res = await fetch(url);
  return res.json();
}

const user = await fetchJson<User>("/api/user");
// user 는 User 타입
```

### 더 안전 (validation 포함)
```typescript
import { z } from "zod";

const UserSchema = z.object({ name: z.string(), age: z.number() });
type User = z.infer<typeof UserSchema>;

async function fetchSafe<T>(url: string, schema: z.ZodSchema<T>): Promise<T> {
  const res = await fetch(url);
  return schema.parse(await res.json());
}

const user = await fetchSafe("/api/user", UserSchema);
```

---

## 10. 자주 만드는 패턴

### A. API response wrapper
```typescript
type ApiResponse<T> = {
  data: T;
  meta: { page: number; total: number };
};

async function getUsers(): Promise<ApiResponse<User[]>> { ... }
```

### B. 결과 (Rust Result 비슷)
```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

async function divide(a: number, b: number): Promise<Result<number>> {
  if (b === 0) return { ok: false, error: new Error("Div") };
  return { ok: true, value: a / b };
}
```

### C. 컬렉션 / repository
```typescript
class Repo<T extends { id: string }> {
  private store = new Map<string, T>();

  add(item: T) { this.store.set(item.id, item); }
  get(id: string): T | undefined { return this.store.get(id); }
  all(): T[] { return [...this.store.values()]; }
}

const userRepo = new Repo<User>();
```

### D. Custom hook (React)
```typescript
function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : initial;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue] as const;
}

const [name, setName] = useLocalStorage("name", "");
```

### E. 함수 합성
```typescript
function pipe<A, B>(fn1: (a: A) => B): (a: A) => B;
function pipe<A, B, C>(fn1: (a: A) => B, fn2: (b: B) => C): (a: A) => C;
function pipe<A, B, C, D>(fn1: (a: A) => B, fn2: (b: B) => C, fn3: (c: C) => D): (a: A) => D;
function pipe(...fns: Array<(x: any) => any>) {
  return (x: any) => fns.reduce((acc, fn) => fn(acc), x);
}

const f = pipe(
  (x: string) => x.length,    // string → number
  (n) => n * 2,                // number → number
  (n) => `${n}px`,             // number → string
);

f("hello");   // "10px"
```

---

## 11. keyof + generic

```typescript
function get<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "유찬", age: 25 };
const name = get(user, "name");   // string
const age = get(user, "age");     // number
const bad = get(user, "foo");     // ❌
```

---

## 12. infer (조건부 타입 안)

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type X = ReturnType<() => string>;   // string
type Y = ReturnType<() => number>;   // number
```

`infer R` = "R 을 자동으로 추출".

이미 TS 가 `ReturnType` 등 utility 로 제공. 자세히 [[../advanced/utility-types]].

---

## 13. 자주 만나는 함정

| 증상 | 원인 |
| --- | --- |
| 추론이 너무 넓음 | 제약 (`extends`) 추가 |
| `T` 가 any 같은 동작 | 제약 없으면 사실상 unknown — 제약 부여 |
| 명시 안 하면 추론 실패 | 함수 호출에 `<T>` 명시 |
| 제네릭 chain 이 너무 복잡 | utility type 사용 — 직접 구현 X |

---

## 14. 다음 단계

1. **[[../advanced/utility-types]]** — Partial / Pick / Omit / Record
2. **[[../advanced/type-guards-narrowing]]** — 타입 좁힘
3. **[[../react/component-typing]]** — React + Generic

---

연결: [[../typescript|↑ hub]] · [[unions-intersections]] · [[../advanced/utility-types]]
