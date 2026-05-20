---
title: "Vitest + React Testing Library"
kind: knowledge
project: frontend
status: current
tags: [frontend, testing, vitest, rtl]
---

# Vitest + RTL

**[[../testing-frameworks|↑ Testing hub]]**

> Vite 기반 프로젝트의 표준 unit / integration 테스트.

---

## 1. 셋업

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

`vite.config.ts`:
```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: "jsdom",
    setupFiles: "./src/test-setup.ts",
  },
});
```

`src/test-setup.ts`:
```typescript
import "@testing-library/jest-dom";
import { afterEach } from "vitest";
import { cleanup } from "@testing-library/react";

afterEach(() => {
  cleanup();
});
```

`package.json`:
```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:cov": "vitest run --coverage"
  }
}
```

---

## 2. 첫 함수 테스트

`src/utils/add.ts`:
```typescript
export function add(a: number, b: number): number {
  return a + b;
}
```

`src/utils/add.test.ts`:
```typescript
import { describe, it, expect } from "vitest";
import { add } from "./add";

describe("add", () => {
  it("두 수를 더한다", () => {
    expect(add(2, 3)).toBe(5);
  });

  it("음수 처리", () => {
    expect(add(-1, 1)).toBe(0);
  });
});
```

```bash
npm test
```

---

## 3. 컴포넌트 테스트

`Counter.tsx`:
```tsx
import { useState } from "react";

export default function Counter({ initial = 0 }) {
  const [count, setCount] = useState(initial);
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>증가</button>
    </div>
  );
}
```

`Counter.test.tsx`:
```tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import Counter from "./Counter";

describe("Counter", () => {
  it("초기값을 보여준다", () => {
    render(<Counter initial={5} />);
    expect(screen.getByText("Count: 5")).toBeInTheDocument();
  });

  it("버튼 클릭 시 +1", async () => {
    const user = userEvent.setup();
    render(<Counter />);

    const btn = screen.getByRole("button", { name: "증가" });
    await user.click(btn);

    expect(screen.getByText("Count: 1")).toBeInTheDocument();
  });
});
```

---

## 4. RTL 의 쿼리

### 우선순위 (높을수록 권장)

#### 1) Accessibility (가장 권장)
```typescript
screen.getByRole("button", { name: "제출" });
screen.getByRole("textbox", { name: "이메일" });
screen.getByRole("heading", { level: 1 });
screen.getByLabelText("이메일");
screen.getByPlaceholderText("이름 입력");
```

#### 2) 시각 가능 텍스트
```typescript
screen.getByText("제출");
screen.getByText(/제출하기/i);   // 정규식
```

#### 3) DOM-specific (최후)
```typescript
screen.getByTestId("submit-button");
// component 에 data-testid="submit-button" 추가 필요
```

### getBy / queryBy / findBy

| | 못 찾으면 | 사용 시점 |
| --- | --- | --- |
| `getBy*` | throw | 있을 거라고 확신 |
| `queryBy*` | null | 없을 수도 (negation 검사) |
| `findBy*` | Promise (대기) | 비동기 (loading 후) |

```typescript
expect(screen.queryByText("에러")).not.toBeInTheDocument();   // 없어야 함
const elem = await screen.findByText("로드됨");                 // 기다림
```

### All variants
```typescript
screen.getAllByRole("listitem");   // 여러 개
```

---

## 5. user-event vs fireEvent

```typescript
// fireEvent (저수준, 비추천)
fireEvent.click(button);

// userEvent (실제 사용자 시뮬레이션, 추천)
const user = userEvent.setup();
await user.click(button);
await user.type(input, "hello");
await user.keyboard("{Enter}");
await user.tab();
await user.selectOptions(select, "option1");
```

userEvent 는 hover / focus / pointer events 다 시뮬레이션 → 실제 동작과 가까움.

---

## 6. assertions (matchers)

### jest-dom 의 matcher
```typescript
expect(elem).toBeInTheDocument();
expect(elem).not.toBeInTheDocument();
expect(elem).toHaveTextContent("hello");
expect(elem).toBeDisabled();
expect(elem).toBeEnabled();
expect(elem).toHaveAttribute("href", "/");
expect(elem).toHaveClass("active");
expect(elem).toHaveValue("input value");
expect(elem).toBeChecked();
expect(elem).toBeVisible();
```

### Vitest 기본
```typescript
expect(x).toBe(5);
expect(x).toEqual({ a: 1 });        // deep
expect(arr).toContain(item);
expect(fn).toHaveBeenCalled();
expect(fn).toHaveBeenCalledWith(arg1, arg2);
expect(promise).rejects.toThrow();
```

---

## 7. mock

### 함수 mock
```typescript
import { vi } from "vitest";

const fn = vi.fn();
fn(1, 2);
expect(fn).toHaveBeenCalledWith(1, 2);

// 반환값 지정
const fn = vi.fn().mockReturnValue(42);
fn();   // 42

// 다른 호출마다 다른 값
const fn = vi.fn()
  .mockReturnValueOnce(1)
  .mockReturnValueOnce(2)
  .mockReturnValue(3);
```

### 모듈 mock
```typescript
import { vi } from "vitest";

vi.mock("./api", () => ({
  fetchUser: vi.fn(() => Promise.resolve({ name: "테스트" })),
}));
```

### Spy (기존 함수 감시)
```typescript
const spy = vi.spyOn(console, "log");
fn();
expect(spy).toHaveBeenCalled();
spy.mockRestore();
```

---

## 8. mock fetch

```typescript
// global fetch mock
beforeEach(() => {
  global.fetch = vi.fn(() =>
    Promise.resolve({
      ok: true,
      json: () => Promise.resolve({ data: [...] }),
    } as Response)
  );
});

// 또는 MSW (Mock Service Worker) 추천 — 자세히 [[mocking]]
```

---

## 9. async / waitFor

```typescript
it("데이터 로드 후 표시", async () => {
  render(<UserList />);

  // 로딩 보임
  expect(screen.getByText("로딩...")).toBeInTheDocument();

  // 데이터 오면
  await screen.findByText("유찬");
  // 또는
  await waitFor(() => {
    expect(screen.getByText("유찬")).toBeInTheDocument();
  });
});
```

### waitFor — 조건 만족까지 대기
```typescript
await waitFor(() => expect(fn).toHaveBeenCalled());
await waitFor(() => expect(elem).toBeVisible(), { timeout: 3000 });
```

---

## 10. Hook 테스트

```typescript
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

it("counter 증가", () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});
```

### Provider 와 함께
```typescript
const wrapper = ({ children }) => (
  <QueryClientProvider client={new QueryClient()}>
    {children}
  </QueryClientProvider>
);

const { result } = renderHook(() => useUsers(), { wrapper });
```

---

## 11. setup / teardown

```typescript
import { beforeEach, afterEach, beforeAll, afterAll } from "vitest";

describe("Counter", () => {
  beforeAll(() => { /* 전체 시작 전 */ });
  afterAll(() => { /* 전체 끝난 후 */ });

  beforeEach(() => { /* 각 it 전 */ });
  afterEach(() => { /* 각 it 후 */ });

  it("...", () => {});
});
```

---

## 12. snapshot (조심)

```typescript
it("snapshot", () => {
  const { container } = render(<Button>Hi</Button>);
  expect(container).toMatchSnapshot();
});
```

처음 실행 → snapshot 파일 생성. 다음 실행 → 비교.

⚠️ 남용하면 의미 없어짐. 작은 컴포넌트 / 안 바뀔 것에만.

---

## 13. coverage

```bash
npm install -D @vitest/coverage-v8
npm test -- --coverage
```

설정:
```typescript
test: {
  coverage: {
    reporter: ["text", "html"],
    exclude: ["**/*.test.ts", "**/test-setup.ts"],
    thresholds: {
      lines: 80,
      branches: 70,
      functions: 80,
      statements: 80,
    },
  },
},
```

---

## 14. UI mode (Vitest 의 GUI)

```bash
npm test -- --ui
```

브라우저에서 테스트 실행 / 디버그 가능.

---

## 15. 자주 만드는 패턴

### A. Form 제출
```tsx
it("폼 제출", async () => {
  const onSubmit = vi.fn();
  const user = userEvent.setup();
  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText("이메일"), "test@example.com");
  await user.type(screen.getByLabelText("비밀번호"), "12345678");
  await user.click(screen.getByRole("button", { name: "로그인" }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: "test@example.com",
    password: "12345678",
  });
});
```

### B. API 호출
```tsx
it("로드 후 사용자 표시", async () => {
  vi.mocked(fetchUsers).mockResolvedValue([{ name: "유찬" }]);
  render(<UserList />);

  expect(await screen.findByText("유찬")).toBeInTheDocument();
});
```

### C. Error 상태
```tsx
it("에러 시 메시지", async () => {
  vi.mocked(fetchUsers).mockRejectedValue(new Error("Network"));
  render(<UserList />);

  expect(await screen.findByText(/에러/)).toBeInTheDocument();
});
```

---

## 16. 함정

| 증상 | 원인 |
| --- | --- |
| `act` warning | async 작업을 await 안 함 |
| `getBy` 가 없다 throw | `query` 또는 `find` 사용 |
| `userEvent` 가 async X | v14+ 는 모두 async. await 필수 |
| Mock 이 안 됨 | `vi.mock` 호출 위치 / hoisting |
| cleanup 안 됨 | setupFiles 의 afterEach 확인 |

---

## 17. 다음 단계

1. **[[rtl-patterns]]** — 더 깊은 컴포넌트 테스트 패턴
2. **[[../e2e/playwright]]** — E2E 테스트

---

연결: [[../testing-frameworks|↑ hub]] · [[rtl-patterns]]
