---
title: "Design Systems — Hub"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [area, frontend, design-systems, hub]
---

# Design Systems — Hub

**[[../frontend|↑ frontend]]**

> 일관된 UI / 재사용 가능한 컴포넌트 / 디자인 토큰. 큰 팀 / 큰 사이트 필수.

---

## 1. 무엇인가

| 요소 | 의미 |
| --- | --- |
| Design Tokens | 색·간격·폰트의 **변수** (브랜드 / 모드 / 플랫폼 별) |
| Components | Button / Input 같은 재사용 UI (HTML+CSS+JS 묶음) |
| Patterns | 자주 등장하는 조합 (Form / Modal / Toast 등) |
| Documentation | 사용법 + Do/Don't + 예시 (Storybook) |

---

## 2. 학습 로드맵

- [[tokens/design-tokens]] — Design Tokens 의 모든 것
- [[storybook/storybook-basics]] — Storybook 셋업 + 컴포넌트 문서화

---

## 3. 유명 디자인 시스템

| 시스템 | 회사 | 특징 |
| --- | --- | --- |
| Material Design | Google | Android / Web 표준 |
| Apple HIG | Apple | iOS 디자인 |
| Fluent | Microsoft | Windows / Office |
| Atlassian Design | Atlassian | 좋은 문서 |
| Polaris | Shopify | e-commerce 강함 |
| Carbon | IBM | enterprise |
| Spectrum | Adobe | creative |
| Lightning | Salesforce | enterprise |

### React UI 라이브러리 (디자인 시스템 구현)
- **MUI / Material UI** — Material Design
- **Mantine** — 모던, 깔끔
- **Chakra UI** — 접근성 친화
- **Ant Design** — enterprise / dashboard
- **Radix UI / shadcn/ui** — 헤드리스 (스타일 X)
- **Headless UI** (Tailwind) — 헤드리스

### Headless 라이브러리 (요즘 추세)
스타일 없음, 동작만 제공. Tailwind / 본인 CSS 와 조합.
- Radix UI
- React Aria
- Headless UI
- shadcn/ui (Radix + Tailwind 조합)

---

## 4. Atomic Design

```
Atoms       — Button, Input, Label (가장 작음)
Molecules   — SearchBar (Input + Button)
Organisms   — Header (Logo + Nav + SearchBar)
Templates   — 레이아웃 (data 없음)
Pages       — 실제 데이터 들어간 화면
```

5 계층 분리. 모든 팀이 따르진 않지만 영향력 큼.

---

## 5. Design Token 체계 (다층)

```
1. Primitive — 원시 값 ("blue-500" = #3b82f6)
2. Semantic — 의미 ("primary" = blue-500)
3. Component — 컴포넌트 별 ("button-bg" = primary)
```

```css
:root {
  /* primitive */
  --blue-500: #3b82f6;
  --gray-100: #f3f4f6;

  /* semantic */
  --color-primary: var(--blue-500);
  --color-bg: var(--gray-100);

  /* component */
  --button-bg: var(--color-primary);
  --card-bg: white;
}
```

자세히 [[tokens/design-tokens]].

---

## 6. 본인 hompage 예시

`src/styles/tokens.css` 가 작은 디자인 시스템:
- 색 (--bg / --text / --accent ...)
- 간격 (--space-1 ~ 10)
- 폰트 (--text-xs ~ 3xl)
- 반지름 (--radius-sm/md/lg/pill)
- 트랜지션 (--t-fast / base)

자세히 [[../../10-projects/hompage/styling/tokens|↑ hompage tokens]].

---

## 7. Storybook — 컴포넌트 문서화

```bash
npx storybook@latest init
```

각 컴포넌트의 사용 예시를 컴포넌트 별 ".stories.tsx" 로 작성:
```tsx
// Button.stories.tsx
export default { component: Button };
export const Primary = { args: { variant: "primary", text: "Click" } };
export const Secondary = { args: { variant: "secondary", text: "Click" } };
export const Disabled = { args: { variant: "primary", disabled: true } };
```

→ `npm run storybook` → 브라우저에서 각 variant 확인 / 시각 회귀 테스트.

자세히 [[storybook/storybook-basics]].

---

## 8. 관련

- [[../react/ui-libraries/ui-libraries|↑ React UI 라이브러리]]
- [[../testing-frameworks/testing-frameworks|↑ Testing]] (Storybook 부분)
