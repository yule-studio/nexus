---
title: "troubleshooting · 무한 re-render — Too many re-renders"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, react, hooks, performance, rerender]
home_hub: "60-troubleshooting"
related: ["[[frontend]]", "[[stale-closure]]"]
---

# 무한 re-render — `Too many re-renders` / `Maximum update depth exceeded`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

```
Error: Too many re-renders. React limits the number of renders to prevent an infinite loop.
Warning: Maximum update depth exceeded.
```

- 페이지가 freezing / CPU 100%
- React DevTools Profiler 에 같은 컴포넌트 수백 번 render

## 원인

### setState in render
```tsx
function Bad() {
  const [n, setN] = useState(0);
  setN(n + 1);   // ← render 마다 호출 → 다시 render
  return <div>{n}</div>;
}
```

### useEffect dep 가 매번 새 reference
```tsx
function Bad({ user }) {
  useEffect(() => {
    fetch('/api/data', { headers: { 'X-User': user.id } })
      .then(r => setData(r));
  }, [user]);   // user 가 매 render 새 object → effect 매번 실행 → setData → re-render
}
```

### handler 가 즉시 호출
```tsx
<button onClick={handleClick()}>  // ← ()  즉시 호출
<button onClick={handleClick}>    // OK
```

## 해결

### dep array 정확히
```tsx
useEffect(() => {
  fetch('/api/data', { headers: { 'X-User': user.id } })
    .then(r => setData(r));
}, [user.id]);   // primitive — stable
```

### useMemo / useCallback
```tsx
const userParams = useMemo(() => ({ id: user.id, role: user.role }), [user.id, user.role]);
const handleClick = useCallback(() => doStuff(id), [id]);
```

### setState callback — prev 사용
```tsx
// Bad — closure stale
setCount(count + 1);

// Good
setCount(prev => prev + 1);
```

### useRef — render 안 트리거하면서 값 유지
```tsx
const renderCount = useRef(0);
useEffect(() => {
  renderCount.current++;
});
```

## 진단

```tsx
// 어느 컴포넌트가 자주 render 하는지
// React DevTools Profiler → Highlight renders
// → 빨간 깜빡임 컴포넌트

// 어떤 prop / state 가 바뀌는지
function MyComp(props) {
  const prev = useRef(props);
  useEffect(() => {
    Object.entries(props).forEach(([k, v]) => {
      if (prev.current[k] !== v) console.log('changed:', k, prev.current[k], '→', v);
    });
    prev.current = props;
  });
}
```

## 재발 방지

- React DevTools Profiler 정기 점검
- ESLint `react-hooks/exhaustive-deps` 켜기
- `why-did-you-render` library (dev only)
- Storybook visual regression — re-render 폭증 시 paint 시간 spike

## 관련

- [[frontend]] / [[stale-closure]]
