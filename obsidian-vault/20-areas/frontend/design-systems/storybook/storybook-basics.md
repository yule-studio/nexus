---
title: "Storybook — 컴포넌트 문서화 / 시각 테스트"
kind: knowledge
project: frontend
status: current
tags: [frontend, storybook, design-systems, documentation]
---

# Storybook

**[[../design-systems|↑ Design Systems hub]]**

> 컴포넌트의 카탈로그. 디자인 시스템의 표준.

---

## 1. 왜 Storybook?

### 문제
- 컴포넌트 만들었는데 사용자 (개발자 / 디자이너) 가 모름
- 다른 페이지에서 호출해야만 미리 보기
- variant / props 마다 case 가 흩어짐

### 해결
- 컴포넌트 카탈로그 (UI)
- 각 컴포넌트의 모든 variant 시각
- 동작 가능한 playground
- 자동 문서

---

## 2. 셋업

```bash
npx storybook@latest init
```

자동 감지 (React / Vue / Svelte 등) + 의존성 + 예시 stories.

폴더:
```
.storybook/
  ├─ main.ts          ← 설정
  └─ preview.ts       ← 전역 스타일 / 데코레이터
src/
  └─ Button.stories.tsx
```

```bash
npm run storybook
# http://localhost:6006
```

---

## 3. 첫 story

`Button.tsx`:
```tsx
type Props = {
  text: string;
  variant?: "primary" | "secondary" | "danger";
  disabled?: boolean;
  onClick?: () => void;
};

export default function Button({ text, variant = "primary", disabled, onClick }: Props) {
  return <button className={`btn btn-${variant}`} disabled={disabled} onClick={onClick}>{text}</button>;
}
```

`Button.stories.tsx`:
```tsx
import type { Meta, StoryObj } from "@storybook/react";
import Button from "./Button";

const meta: Meta<typeof Button> = {
  component: Button,
  title: "UI/Button",
  argTypes: {
    variant: {
      control: "select",
      options: ["primary", "secondary", "danger"],
    },
    onClick: { action: "clicked" },
  },
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: { text: "눌러", variant: "primary" },
};

export const Secondary: Story = {
  args: { text: "눌러", variant: "secondary" },
};

export const Danger: Story = {
  args: { text: "삭제", variant: "danger" },
};

export const Disabled: Story = {
  args: { text: "비활성", disabled: true },
};
```

→ Storybook 에 4 variant 표시.

---

## 4. argTypes / controls

각 prop 의 control UI:

```typescript
argTypes: {
  text: { control: "text" },              // 문자열 입력
  variant: { control: "select", options: ["a", "b"] },   // dropdown
  disabled: { control: "boolean" },        // checkbox
  count: { control: { type: "number", min: 0, max: 100 } },
  color: { control: "color" },             // color picker
  size: { control: "radio", options: ["sm", "md", "lg"] },
  onClick: { action: "clicked" },          // 이벤트 로깅
}
```

---

## 5. autodocs

```typescript
const meta: Meta<typeof Button> = {
  title: "UI/Button",
  component: Button,
  tags: ["autodocs"],   // ← 추가
  // ...
};
```

→ Storybook 이 자동으로 문서 페이지 생성 (props 표 + 모든 stories).

---

## 6. play function (interaction test)

```tsx
import { userEvent, within, expect } from "@storybook/test";

export const ClickTest: Story = {
  args: { text: "Click" },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const button = canvas.getByRole("button", { name: "Click" });
    await userEvent.click(button);
    await expect(button).toBeInTheDocument();
  },
};
```

→ Storybook 안에서 자동 동작. RTL 패턴 그대로.

---

## 7. decorator — 모든 story 에 wrapper

```typescript
// .storybook/preview.ts
import type { Preview } from "@storybook/react";
import { ThemeProvider } from "../src/theme";
import "../src/index.css";

const preview: Preview = {
  decorators: [
    (Story) => (
      <ThemeProvider>
        <Story />
      </ThemeProvider>
    ),
  ],
  parameters: {
    backgrounds: {
      default: "dark",
      values: [
        { name: "dark", value: "#0b0d0c" },
        { name: "light", value: "#fff" },
      ],
    },
  },
};
export default preview;
```

---

## 8. 다른 viewport (반응형)

```typescript
// preview.ts
parameters: {
  viewport: {
    viewports: INITIAL_VIEWPORTS,
  },
}
```

또는 individual story:
```typescript
export const Mobile: Story = {
  parameters: { viewport: { defaultViewport: "mobile1" } },
};
```

---

## 9. addons (확장)

설치 시 자동:
- `@storybook/addon-a11y` — 접근성 검사
- `@storybook/addon-themes` — 다크/라이트 toggle
- `@chromatic-com/storybook` — 시각 회귀 (Chromatic)

```bash
npx storybook@latest add @storybook/addon-a11y
```

---

## 10. 시각 회귀 테스트 — Chromatic

```bash
npm install -D chromatic
npx chromatic --project-token=YOUR_TOKEN
```

각 story 의 스크린샷 → 변경 시 자동 diff → PR 코멘트.

대안:
- **Playwright snapshot** (자체 호스팅)
- **Percy** (BrowserStack)

---

## 11. MDX (마크다운 + JSX)

```mdx
import { Meta, Story, Canvas } from "@storybook/blocks";
import * as ButtonStories from "./Button.stories";

<Meta of={ButtonStories} />

# Button

설명 본문 마크다운.

## Variants

<Canvas of={ButtonStories.Primary} />

## 사용 시점
- Primary: 기본 액션
- Secondary: 보조
- Danger: 삭제 / 파괴
```

---

## 12. 배포

### Storybook 자체 빌드
```bash
npm run build-storybook
# storybook-static/ 생성
```

→ 정적 사이트. Vercel / Netlify / GitHub Pages 등에 배포.

### Chromatic 호스팅
무료 (open source) — 자동 배포 + 시각 회귀.

---

## 13. 실 프로젝트 예시

```
src/
├─ components/
│  ├─ Button/
│  │  ├─ Button.tsx
│  │  ├─ Button.stories.tsx
│  │  └─ Button.test.tsx
│  ├─ Card/
│  ├─ Input/
│  └─ ...
└─ design-tokens/
   └─ tokens.css
```

### 디자인 시스템 라이브러리로 publish
1. Storybook 으로 카탈로그 작성
2. npm publish (또는 monorepo workspace)
3. 다른 프로젝트에서 import

---

## 14. 자주 만나는 함정

| 증상 | 원인 |
| --- | --- |
| story 가 안 보임 | 파일명 `.stories.tsx` 확인 |
| 스타일 안 적용 | preview.ts 에서 globals.css import |
| Provider 에러 | decorator 에 wrapper |
| Vite alias 안 됨 | .storybook/main.ts 의 viteFinal |

---

## 15. 다음 단계

1. **[[../tokens/design-tokens]]** — Design Tokens
2. **[[../../testing-frameworks/testing-frameworks]]** — 테스트와 결합

---

연결: [[../design-systems|↑ hub]]
