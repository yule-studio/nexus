---
title: "TS 객체 — type vs interface"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, object, interface, type]
---

# 객체 / interface

**[[../typescript|↑ TS hub]]**

---

## 1. 객체 타입 정의 두 방법

### type
```typescript
type User = {
  name: string;
  age: number;
};
```

### interface
```typescript
interface User {
  name: string;
  age: number;
}
```

거의 동일. 차이 §4.

---

## 2. 옵셔널 / readonly

```typescript
type User = {
  readonly id: string;     // 변경 불가
  name: string;
  age?: number;            // 있을 수도, 없을 수도
};
```

---

## 3. 중첩

```typescript
type User = {
  name: string;
  address: {
    city: string;
    zip: string;
  };
  friends: User[];
};
```

또는 분리:
```typescript
type Address = { city: string; zip: string };

type User = {
  name: string;
  address: Address;
  friends: User[];
};
```

분리가 재사용성 ↑.

---

## 4. type vs interface — 차이

### 둘 다 가능
- 객체 모양 정의
- 확장
- 자동 완성

### type 만 가능
- union (`type X = A | B`)
- intersection (`type X = A & B`)
- mapped (`type X = { [K in keyof T]: ... }`)
- conditional (`type X = T extends U ? A : B`)
- 함수 / 튜플 / 리터럴

### interface 만 가능
- declaration merging (같은 이름 여러 번 선언 → 합쳐짐)
- `extends` 키워드 (보기 좋음)
- class 가 `implements`

### 권장
- **객체 모양** + 확장 위주 → `interface`
- **모든 표현 통일** → `type` (현업 자주)
- 한 프로젝트에선 일관성. 본인 hompage 는 `type`.

---

## 5. 확장 (extend)

### interface — extends
```typescript
interface Animal {
  name: string;
}

interface Dog extends Animal {
  breed: string;
}
```

### type — intersection (&)
```typescript
type Animal = { name: string };
type Dog = Animal & { breed: string };
```

### 여러 확장
```typescript
interface A { a: string }
interface B { b: string }
interface C extends A, B { c: string }
// = { a: string, b: string, c: string }

type C = A & B & { c: string };
```

---

## 6. index signature — 동적 키

```typescript
type Dict = {
  [key: string]: number;
};

const counts: Dict = {
  apple: 5,
  banana: 2,
};
counts.cherry = 7;     // OK
```

### Record (단축)
```typescript
type Dict = Record<string, number>;
```

### 알려진 + 동적
```typescript
type Mixed = {
  name: string;       // 정의됨
  [key: string]: any; // 그 외 자유
};
```

---

## 7. 함수 시그니처 in 객체

### method 단축
```typescript
type Calc = {
  add(a: number, b: number): number;
  subtract(a: number, b: number): number;
};
```

### property 형태
```typescript
type Calc = {
  add: (a: number, b: number) => number;
};
```

차이는 미묘. 보통 method 단축 사용.

---

## 8. 객체 typeof

```typescript
const config = {
  port: 80,
  host: "localhost",
};

type Config = typeof config;
// { port: number; host: string }
```

### as const + typeof
```typescript
const STATUS = {
  PENDING: "pending",
  ACTIVE: "active",
} as const;

type Status = typeof STATUS[keyof typeof STATUS];
// "pending" | "active"
```

---

## 9. keyof

```typescript
type User = { name: string; age: number };

type Keys = keyof User;
// "name" | "age"

function get<K extends keyof User>(user: User, key: K): User[K] {
  return user[key];
}

const name = get(user, "name");   // string
const age = get(user, "age");     // number
```

---

## 10. 자주 만드는 패턴

### A. API 응답 타입
```typescript
type ApiResponse<T> = {
  data: T;
  meta: {
    page: number;
    total: number;
  };
  error?: string;
};

const users: ApiResponse<User[]> = await fetch(...);
```

### B. Form data
```typescript
type LoginForm = {
  email: string;
  password: string;
  remember?: boolean;
};

function login(data: LoginForm) {}
```

### C. Discriminated Union (state)
```typescript
type State =
  | { kind: "idle" }
  | { kind: "loading" }
  | { kind: "success"; data: User[] }
  | { kind: "error"; error: Error };

function render(state: State) {
  switch (state.kind) {
    case "success": return state.data;   // 타입 좁힘 자동
    case "error":   return state.error;
  }
}
```

자세히 [[../advanced/type-guards-narrowing]].

### D. Props 타입
```typescript
type ButtonProps = {
  text: string;
  onClick?: () => void;
  variant?: "primary" | "secondary";
  disabled?: boolean;
};

function Button({ text, onClick, variant = "primary", disabled = false }: ButtonProps) {
  return <button onClick={onClick} disabled={disabled}>{text}</button>;
}
```

자세히 [[../react/component-typing]].

### E. 환경변수 타입
```typescript
// env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_TOKEN: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}

// 사용
import.meta.env.VITE_API_URL    // string (자동 완성)
```

---

## 11. 함정

| 증상 | 원인 |
| --- | --- |
| 객체 추가 property 가 에러 | exact type 검사. extra property 안 됨 |
| `{ a: 1 }` 가 `User` 와 호환 안 됨 | structural typing — 모양만 보지만 missing field 가 있으면 에러 |
| index signature 가 너무 넓음 | `Record<keyof X, V>` 로 좁히기 |
| type 과 interface 둘 다 같은 이름 | interface 만 merging. 충돌 |

---

## 12. 다음 단계

1. **[[unions-intersections]]** — union / intersection / literal
2. **[[generics]]** — 제네릭

---

연결: [[../typescript|↑ hub]] · [[primitives]] · [[unions-intersections]]
