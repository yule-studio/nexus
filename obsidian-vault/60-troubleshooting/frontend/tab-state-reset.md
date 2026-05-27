---
title: "troubleshooting · Tab 전환 시 React state reset"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [frontend, react, state, key, lifecycle]
home_hub: "60-troubleshooting"
related: ["[[frontend]]"]
---

# Tab 전환 시 React state reset

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | frontend-engineer | 최초 |

## 증상

- Tab 1 → Tab 2 → Tab 1 돌아왔는데 입력했던 form 데이터 사라짐
- 또는 modal 닫고 다시 열면 scroll 위치 초기화
- `key` prop 가 바뀌어 컴포넌트 unmount

## 원인

React 가 컴포넌트 instance 동일성 판단:
- 같은 위치 + 같은 type → 같은 instance (state 유지)
- 다른 type 또는 `key` 다름 → unmount + remount (state lost)

```tsx
// Bad
{showTab === 'a' ? <FormA /> : <FormB />}
// 위치 1: <FormA /> or <FormB />
// 전환 시 type 다름 → 양쪽 모두 reset
```

또는 parent 의 `key` 가 변하면 child 도 unmount:
```tsx
<Layout key={pathname}>   // pathname 변경 시 전체 child unmount
```

## 해결

### 모든 tab 항상 render — `hidden` 으로 숨김
```tsx
<div hidden={tab !== 'a'}><FormA /></div>
<div hidden={tab !== 'b'}><FormB /></div>
```

→ DOM 에 항상 존재, display:none 만 → state 유지. 단 모든 tab 가 동시에 mount → 무거우면 다른 방법.

### State 를 parent 로 lift
```tsx
function Tabs() {
  const [formData, setFormData] = useState({ a: {}, b: {} });   // ← parent 유지

  return tab === 'a'
    ? <FormA value={formData.a} onChange={v => setFormData({...formData, a: v})} />
    : <FormB value={formData.b} onChange={v => setFormData({...formData, b: v})} />;
}
```

### Context / Zustand / Jotai — 전역 state
```tsx
const useFormStore = create(set => ({
  formData: {},
  setForm: (data) => set({ formData: data }),
}));

function FormA() {
  const { formData, setForm } = useFormStore();
  // ...
}
```

→ 컴포넌트 unmount 와 무관하게 state 유지.

### URL state (search params) — 가장 안정적
```tsx
import { useSearchParams } from 'next/navigation';

const [searchParams, setSearchParams] = useSearchParams();
const value = searchParams.get('q') || '';

setSearchParams({ q: newValue });
```

→ tab 전환 / 페이지 reload 후에도 유지. SEO / share 도 OK.

### React Router `<Outlet />` — layout shared

Router 의 layout 컴포넌트는 child route 변경에도 unmount 안 됨:
```tsx
<Route element={<Layout />}>   {/* 유지 */}
  <Route path="a" element={<TabA />} />
  <Route path="b" element={<TabB />} />
</Route>
```

## form 의 special case — uncontrolled
```tsx
<form>
  <input name="email" defaultValue={savedEmail} />
</form>
```

→ React state 와 무관. DOM 자체에 value 보존. 단 React 가 input 을 unmount 하면 사라짐.

## 진단

```tsx
// React DevTools — Components tab
// → tab 전환 후 컴포넌트 tree 검사
// → unmount 됐으면 reset 원인

// useEffect 로 mount/unmount 추적
useEffect(() => {
  console.log('FormA mounted');
  return () => console.log('FormA unmounted');
}, []);
```

## 재발 방지

- form / scroll 등 중요 state 는 처음부터 lift up 또는 store
- URL state 우선 (shareable + history)
- 컴포넌트 tree 의 `key` 변경 신중

## 관련

- [[frontend]] / [[infinite-rerender]]
