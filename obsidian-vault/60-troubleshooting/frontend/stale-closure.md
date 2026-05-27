---
title: "troubleshooting · Stale closure — handler 가 옛 state 사용"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, react, hooks, closure]
home_hub: "60-troubleshooting"
related: ["[[frontend]]", "[[infinite-rerender]]"]
---

# Stale closure — `setInterval` / event handler 가 옛 값

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);   // 항상 0
      setCount(count + 1);  // 항상 0 → 1
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>{count}</div>;
}
```

- count 가 0 → 1 → 1 → 1 ... (증가 안 함)
- WebSocket message handler 가 옛 state 보고 잘못된 분기

## 원인

`useEffect(..., [])` 가 mount 시 1회 실행. 그 시점의 `count = 0` 이 closure 에 캡처. interval callback 도 같은 closure 사용 → 영원히 0.

## 해결

### setState callback — 가장 단순
```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1);   // ← prev 가 latest
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### useRef — latest 참조
```tsx
const countRef = useRef(count);
useEffect(() => { countRef.current = count; }, [count]);

useEffect(() => {
  const id = setInterval(() => {
    console.log(countRef.current);   // latest
  }, 1000);
  return () => clearInterval(id);
}, []);
```

### dep array 에 state 포함
```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);   // 매번 effect 재실행 — interval 재생성
```

→ 작동하지만 매 tick 마다 interval 재생성 — 부담.

### Custom hook — useInterval
```tsx
function useInterval(callback: () => void, delay: number) {
  const saved = useRef(callback);
  useEffect(() => { saved.current = callback; }, [callback]);
  useEffect(() => {
    const id = setInterval(() => saved.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}

// 사용
useInterval(() => setCount(c => c + 1), 1000);
```

## WebSocket / EventSource 패턴

```tsx
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (e) => {
    // count 가 stale → useRef 또는 setState callback
    setMessages(prev => [...prev, JSON.parse(e.data)]);
  };
  return () => ws.close();
}, [url]);
```

## 진단

```tsx
// ESLint react-hooks/exhaustive-deps 가 dep 누락 경고
// → 끄지 말 것
```

## 재발 방지

- `react-hooks/exhaustive-deps` ESLint rule 강제
- setState 는 함수형 (`setX(prev => ...)`) 우선
- useEffect 안 외부 ref 사용 시 useRef 패턴 표준
- 회귀 테스트: Vitest + `@testing-library/react` — counter 테스트

## 관련

- [[frontend]] / [[infinite-rerender]] / [[event-listener-leak]]
