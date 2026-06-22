---
title: "UI 라이브러리"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - frontend
  - ui-libraries
---

# UI 라이브러리

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-06-22 | engineering-agent/tech-lead | 라이브러리 선택 가이드로 재작성 |

**[[../frontend|↑ frontend]]**

> 컴포넌트 UI 라이브러리 선택 가이드 — headless vs styled, 디자인 시스템 연동 관점에서 정리.

## 1. 무엇인가

UI 라이브러리는 버튼·모달·드롭다운·테이블 같은 공통 컴포넌트를 미리 만들어 둔 패키지다. 프레임워크가 아니라 프레임워크(주로 React) 위에서 쓰는 부품 모음이다. 크게 두 축으로 나뉜다.

- **Styled(완성형)** — 디자인까지 입혀진 컴포넌트를 바로 가져다 쓴다. 예: **MUI(Material UI)**, **Ant Design**, **Chakra UI**, **Mantine**.
- **Headless(무스타일/동작만)** — 접근성·키보드·포커스·상태 관리 등 "동작과 접근성"만 제공하고, 스타일은 사용자가 입힌다. 예: **Radix UI**, **Headless UI**, **React Aria**, **TanStack(Table 등)**.
- **복사형(코드 소유)** — **shadcn/ui** 는 라이브러리 설치가 아니라 Radix + Tailwind 기반 컴포넌트 코드를 내 레포에 복사해 넣는 방식. 의존성이 아니라 내 소스가 된다.

## 2. 왜 / 언제 선택하나

(1) **강점/기준**
- **빠른 완성**이 목표면 styled(MUI/Ant) — 어드민·사내 도구는 디자인을 처음부터 만들 필요가 없다.
- **고유 디자인 시스템**이 목표면 headless(Radix) 또는 복사형(shadcn) — 접근성/동작은 검증된 것을 받고 외형은 자유롭게.

(2) **안 맞는데 쓰면**
- 강한 디자인 정체성이 필요한 제품에 MUI/Ant 같은 styled 를 쓰면, 기본 테마와 싸우며 커스터마이즈하느라 오히려 더 느려지고 번들도 무거워진다("탈출 비용").
- 반대로 디자이너·시간이 없는 사내 어드민에 headless 를 쓰면 모든 스타일을 직접 짜야 해 비효율적이다.

(3) **대안 비교(요지)**
- **MUI** — 가장 큰 생태계, Material Design 색채가 강함, 컴포넌트가 방대. 커스터마이즈는 가능하나 기본 룩에서 벗어날수록 노력 급증.
- **Ant Design** — 어드민/엔터프라이즈 폼·테이블에 강함. 한국 사내 도구에서 흔함. 디자인 색이 진해 브랜드 제품엔 덜 적합.
- **Chakra UI** — 접근성·DX 좋고 테마 토큰이 깔끔. 적당히 styled + 적당히 커스터마이즈.
- **Mantine** — 컴포넌트+훅이 풍부하고 현대적. Chakra 대안으로 부상.
- **Radix UI** — headless 의 사실상 표준. 접근성/동작만, 스타일 0. shadcn 의 토대.
- **shadcn/ui** — Radix+Tailwind 코드를 복사 소유. 버전 종속 없이 자유 수정, 단 업스트림 업데이트는 수동.

(4) **트레이드오프** — styled 는 초기 속도↑·디자인 자유도↓·번들↑, headless/shadcn 은 디자인 자유도↑·초기 비용↑·유지보수 책임↑(특히 shadcn 은 코드를 직접 소유). 디자인 시스템 보유 여부와 디자이너 리소스가 결정 변수다.

## 3. 핵심 개념 / 어떻게

```tsx
// styled: 바로 쓰는 완성형 (MUI)
import { Button } from "@mui/material";
<Button variant="contained" onClick={save}>저장</Button>

// headless: 동작/접근성만, 스타일은 내가 (Radix Dialog)
import * as Dialog from "@radix-ui/react-dialog";
<Dialog.Root>
  <Dialog.Trigger className="my-btn">열기</Dialog.Trigger>
  <Dialog.Content className="my-modal">...</Dialog.Content>
</Dialog.Root>

// shadcn: 설치가 아니라 코드 복사 → 내 레포의 소스가 됨
// npx shadcn@latest add button  →  components/ui/button.tsx 생성
import { Button } from "@/components/ui/button";
```

- styled 는 테마 객체/토큰으로 전역 디자인을 제어한다(MUI `ThemeProvider`, Chakra theme).
- headless 는 보통 Tailwind 또는 CSS-in-JS 와 짝지어 외형을 입힌다.

## 4. 적용 맥락

- **어드민/사내 도구·빠른 MVP** → MUI 또는 Ant Design(완성형).
- **브랜드 제품·자체 디자인 시스템** → Radix(+Tailwind) 또는 shadcn/ui(코드 소유).
- **적당한 절충·좋은 DX** → Chakra / Mantine.
- 디자인 시스템을 Figma 등에서 운영한다면, 토큰을 코드로 옮기기 쉬운 headless/shadcn 쪽이 동기화에 유리하다.

## 5. 관련

- [[../react/react|↗ React hub]]
- [[../_common/_common|↗ 컴포넌트 모델·스타일링 공통]]
- [[../frontend|↑ frontend]]
- 공식 — MUI https://mui.com / Ant Design https://ant.design / Chakra https://chakra-ui.com / Mantine https://mantine.dev / Radix https://www.radix-ui.com / shadcn https://ui.shadcn.com
