---
title: "React + TS — Props / Event / Ref / Children 타입"
kind: knowledge
project: frontend
status: current
tags: [frontend, typescript, react, props, types]
---

# React Component 타입

**[[../typescript|↑ TS hub]]**

> React 컴포넌트의 Props / Event / Ref 를 어떻게 타입화하는가.

---

## 1. Props 기본

```typescript
type ButtonProps = {
  text: string;
  onClick?: () => void;
  disabled?: boolean;
};

function Button({ text, onClick, disabled = false }: ButtonProps) {
  return <button onClick={onClick} disabled={disabled}>{text}</button>;
}
```

### React.FC 는 안 씀 (구식)
```typescript
// ❌ 옛 스타일
const Button: React.FC<ButtonProps> = ({ text }) => <button>{text}</button>;

// ✅ 현재 권장
function Button({ text }: ButtonProps) { ... }
```

`React.FC` 는 children 을 암묵적으로 받는 등 부작용. 명시 props 사용.

---

## 2. children

### 기본 — ReactNode
```typescript
type Props = {
  children: React.ReactNode;
};

function Card({ children }: Props) {
  return <div className="card">{children}</div>;
}
```

`ReactNode` = string, number, element, fragment, null, undefined, array of them 등.

### 옵셔널
```typescript
type Props = {
  children?: React.ReactNode;
};
```

### children 강제 React Element 만
```typescript
type Props = { children: React.ReactElement };
```

거의 안 씀 — ReactNode 가 보통.

### render prop (함수)
```typescript
type Props = {
  children: (data: User) => React.ReactNode;
};

function UserProvider({ children }: Props) {
  const user = useUser();
  return <>{children(user)}</>;
}

// 사용
<UserProvider>
  {(user) => <h1>{user.name}</h1>}
</UserProvider>
```

---

## 3. Event 핸들러

### click
```typescript
type Props = {
  onClick?: React.MouseEventHandler<HTMLButtonElement>;
};

// 또는
type Props = {
  onClick?: (e: React.MouseEvent<HTMLButtonElement>) => void;
};

function Button({ onClick }: Props) {
  return <button onClick={onClick}>눌러</button>;
}
```

### change (input)
```typescript
type Props = {
  onChange?: React.ChangeEventHandler<HTMLInputElement>;
};
```

또는:
```typescript
function handle(e: React.ChangeEvent<HTMLInputElement>) {
  console.log(e.target.value);
}
```

### form submit
```typescript
function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
  e.preventDefault();
  const data = new FormData(e.currentTarget);
}
```

### keyboard
```typescript
function handleKey(e: React.KeyboardEvent<HTMLInputElement>) {
  if (e.key === "Enter") { ... }
}
```

### 가장 짧은 형태 (값만)
```typescript
type Props = {
  onChange?: (value: string) => void;
};

function Input({ onChange }: Props) {
  return <input onChange={(e) => onChange?.(e.target.value)} />;
}
```

콜백을 추상화 → 사용자 입장에서 더 간단.

---

## 4. native HTML 속성 확장

### 기본 button 의 모든 속성 + 내 props
```typescript
type ButtonProps = React.ComponentProps<"button"> & {
  variant?: "primary" | "secondary";
};

function Button({ variant = "primary", ...rest }: ButtonProps) {
  return <button className={variant} {...rest} />;
}

// 사용
<Button onClick={...} disabled type="submit" variant="secondary">눌러</Button>
```

### ComponentPropsWithoutRef (ref 제외)
```typescript
type Props = React.ComponentPropsWithoutRef<"input">;
```

### 자주 쓰는 element
```typescript
ComponentProps<"button">    // <button> 의 모든 속성
ComponentProps<"input">     // <input>
ComponentProps<"a">         // <a>
ComponentProps<"div">       // <div>
```

---

## 5. ref

### forwardRef
```typescript
type InputProps = React.ComponentPropsWithoutRef<"input">;

const Input = React.forwardRef<HTMLInputElement, InputProps>(
  function Input(props, ref) {
    return <input ref={ref} {...props} />;
  }
);

// 사용
const inputRef = useRef<HTMLInputElement>(null);
<Input ref={inputRef} />
```

### React 19 — forwardRef 불필요
React 19+ 에서 ref 가 일반 prop:
```typescript
type InputProps = React.ComponentPropsWithoutRef<"input"> & {
  ref?: React.Ref<HTMLInputElement>;
};

function Input({ ref, ...rest }: InputProps) {
  return <input ref={ref} {...rest} />;
}
```

---

## 6. useState 타입

### 단순 (자동 추론)
```typescript
const [count, setCount] = useState(0);             // number
const [name, setName] = useState("");              // string
```

### 명시
```typescript
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<string[]>([]);
```

### updater 함수
```typescript
setCount((prev) => prev + 1);    // prev: number
```

---

## 7. useReducer 타입

```typescript
type State = { count: number };
type Action =
  | { type: "INC" }
  | { type: "DEC" }
  | { type: "SET"; value: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "INC": return { count: state.count + 1 };
    case "DEC": return { count: state.count - 1 };
    case "SET": return { count: action.value };
  }
}

const [state, dispatch] = useReducer(reducer, { count: 0 });
dispatch({ type: "SET", value: 42 });
```

discriminated union 의 자연스러운 활용.

---

## 8. useRef 타입

### DOM
```typescript
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  inputRef.current?.focus();
}, []);

return <input ref={inputRef} />;
```

### 일반 값
```typescript
const timerRef = useRef<number | null>(null);

timerRef.current = setTimeout(...);
```

---

## 9. Context 타입

```typescript
type ThemeContextValue = {
  theme: "light" | "dark";
  setTheme: (t: "light" | "dark") => void;
};

const ThemeContext = React.createContext<ThemeContextValue | null>(null);

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme outside provider");
  return ctx;
}
```

---

## 10. Custom hook 타입

```typescript
function useCounter(initial: number = 0) {
  const [count, setCount] = useState(initial);

  return {
    count,
    increment: () => setCount((c) => c + 1),
    decrement: () => setCount((c) => c - 1),
    reset: () => setCount(initial),
  };
}

const { count, increment } = useCounter(10);
```

### Generic hook
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
// readonly [string, Dispatch<...>]
```

---

## 11. 자주 만드는 패턴

### A. 합성 컴포넌트 (Compound)
```typescript
type CardProps = { children: React.ReactNode };
type CardTitleProps = { children: React.ReactNode };
type CardBodyProps = { children: React.ReactNode };

function Card({ children }: CardProps) {
  return <div className="card">{children}</div>;
}
Card.Title = function CardTitle({ children }: CardTitleProps) {
  return <h3>{children}</h3>;
};
Card.Body = function CardBody({ children }: CardBodyProps) {
  return <div className="body">{children}</div>;
};

// 사용
<Card>
  <Card.Title>제목</Card.Title>
  <Card.Body>내용</Card.Body>
</Card>
```

### B. polymorphic (as prop)
```typescript
type AsProp<T extends React.ElementType> = {
  as?: T;
} & React.ComponentPropsWithoutRef<T>;

function Box<T extends React.ElementType = "div">({
  as,
  children,
  ...rest
}: AsProp<T> & { children?: React.ReactNode }) {
  const Component = as || "div";
  return <Component {...rest}>{children}</Component>;
}

<Box as="button" onClick={...}>버튼</Box>
<Box as="a" href="/">링크</Box>
<Box>div</Box>
```

### C. discriminated props (variant 별 다른 props)
```typescript
type ButtonProps =
  | { variant: "link"; href: string }
  | { variant: "button"; onClick: () => void };

function Button(props: ButtonProps) {
  if (props.variant === "link") {
    return <a href={props.href}>...</a>;
  } else {
    return <button onClick={props.onClick}>...</button>;
  }
}
```

### D. 모든 children 가 특정 컴포넌트
```typescript
type ListProps = {
  children: React.ReactElement<typeof ListItem> | React.ReactElement<typeof ListItem>[];
};

function List({ children }: ListProps) { ... }
```

---

## 12. 함정

| 증상 | 원인 |
| --- | --- |
| `React.FC` 의 children 보임 | 옛 스타일. 직접 타입 명시 |
| ref 가 readonly | 일반 props 처럼 못 받음 — forwardRef 또는 React 19+ |
| event handler 타입 안 맞음 | element 타입 (`HTMLInputElement`) 맞춤 |
| useState undefined | 초기값 명시 또는 union 에 undefined |
| as prop 의 자동 완성 안 됨 | polymorphic 어려움 — Headless UI 처럼 prop 추론 |

---

## 13. 다음 단계

1. **[[../advanced/utility-types]]** — utility types
2. **[[../../react/typescript/component-types|React TS 깊이]]**

---

연결: [[../typescript|↑ hub]] · [[../types/generics]]
