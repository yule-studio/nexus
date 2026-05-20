---
title: "State Management — Hub (모든 옵션 비교)"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, state-management, hub]
---

# State Management — Hub

**[[../frontend|↑ frontend]]**

> 데이터 흐름 / 어디에 상태를 저장할지. 가장 자주 잘못된 선택.

---

## 1. 상태의 4 종

### A. UI 상태 (Local)
폼 / modal / accordion 등 → `useState` / `useReducer`.

### B. 서버 상태 (Server)
API 데이터 → **TanStack Query** / SWR.

### C. 글로벌 클라이언트
인증 / 테마 / 언어 → Zustand / Jotai / Context.

### D. URL 상태
페이지 / 필터 / 정렬 → `useSearchParams`.

---

## 2. 옵션 비교

| 도구 | 종류 | 추천 시점 |
| --- | --- | --- |
| `useState` | local | 작은 상태, 90% |
| `useReducer` | local | state machine |
| `useContext` | global | 작은 글로벌 |
| **Zustand** | global | 글로벌 표준 |
| **Jotai** | global | atomic |
| **Redux Toolkit** | global | 큰 enterprise |
| **TanStack Query** | server | 서버 데이터 표준 |
| **SWR** | server | 가벼운 대안 |
| **XState** | machine | 복잡한 흐름 (결제) |

자세히 [[comparison/state-strategy]] (또는 [[../react/state-management/state-strategy|React state strategy]]).

---

## 3. 흔한 안티패턴

### ❌ Redux 로 server data
손으로 loading/error/cache 다 관리.

### ✅ TanStack Query
```typescript
const { data, isLoading } = useQuery({
  queryKey: ["users"],
  queryFn: fetchUsers,
});
```

### ❌ Context provider 지옥
변경 시 모든 자식 re-render.

### ✅ Zustand selector
```typescript
const user = useStore((s) => s.user);    // 그 부분만 re-render
```

---

## 4. 결정 트리

```
서버 API?       → TanStack Query
URL param?      → useSearchParams
한 컴포넌트?    → useState
자식 1~2 단계?  → props / Context
글로벌?         → Zustand
```

---

## 5. 자주 묻는 질문

**Q. Context vs Zustand?**
→ Context — 작은 글로벌 (테마). Zustand — 큰 (사용자, 카트). Context 는 변경 시 모든 자식 re-render.

**Q. Redux 는 죽었어?**
→ Redux Toolkit 쓰면 OK. 새 프로젝트는 Zustand 가 더 가볍.

---

## 6. 관련

- [[../react/state-management/state-management|↑ React state]] — 더 깊이
- [[../react/server-state/server-state|↑ Server state]]
- [[../react/state-management/zustand|↑ Zustand]]
- [[../react/server-state/react-query|↑ TanStack Query]]
