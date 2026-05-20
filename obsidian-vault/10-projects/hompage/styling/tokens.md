---
title: "hompage styling — Design Tokens 사전"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:00:00+09:00
tags: [project, hompage, styling, css, tokens]
home_hub: hompage
related:
  - "[[../hompage]]"
  - "[[colors]]"
  - "[[typography]]"
---

# Design Tokens 사전

**[[../hompage|↑ hompage hub]]**

> `src/styles/tokens.css` 한 파일에 모든 색·여백·폰트·반지름 정의. 여기 한 줄만 바꿔도 전체 화면 영향.

---

## 0. 빠른 안내

### 위치
```
src/styles/tokens.css
```

### 작동 방식
- `:root` (전체) 에 CSS custom property 로 선언
- 다른 CSS 파일과 컴포넌트는 `var(--bg)` 처럼 참조만 함
- → tokens.css 값만 바꿔도 사이트 전역에 즉시 반영
- 다크/라이트 테마는 `<html data-theme="dark">` / `"light"` 로 스위치

### 변경 방법 (공통)
1. `src/styles/tokens.css` 열기
2. 원하는 토큰 줄에서 값만 수정 (예: `--accent: #4ade80;` → `#a78bfa;`)
3. 저장 → `npm run dev` 가 hot-reload, 즉시 반영
4. `git commit && git push` 후 5분 → 배포 사이트 반영

---

## 1. Surface (배경)

```css
--bg:              #0b0d0c;   ← 페이지 전체 배경 (가장 어두움)
--bg-elevated:     #101413;   ← 띄워진 영역 (현재 거의 안 씀)
--bg-card:         #0f1311;   ← 카드 배경 (Home / Projects / Skills 다)
--bg-card-hover:   #131816;   ← 카드 hover 시 배경
--bg-pill:         #1a201d;   ← pill / chip 배경 (목차·status·chip 등)
--bg-pill-active:  #1d2421;   ← pill hover 시 배경
```

### 라이트 모드 (`data-theme="light"`)
```css
--bg:              #f7f8f6;
--bg-elevated:     #ffffff;
--bg-card:         #ffffff;
--bg-card-hover:   #f1f3ef;
--bg-pill:         #eef1ec;
--bg-pill-active:  #e3ebe5;
```

### 어떤 화면에 어떤 토큰?
| 토큰 | 보이는 곳 |
| --- | --- |
| `--bg` | 페이지 바깥 전체. dot-grid 패턴이 위에 깔림 |
| `--bg-card` | Home 의 10 카드 / Projects 목록 카드 / Skills 카드 / Certs / Awards 등 모든 카드 |
| `--bg-card-hover` | hover 시 살짝 밝아짐 |
| `--bg-pill` | Topbar 의 `Available · Q3 2026` / theme toggle / 목차 pill / chip 들 |
| `--bg-pill-active` | 목차 pill 의 hover, theme toggle hover |

---

## 2. Border (테두리)

```css
--border:          #1c211f;   ← 기본 (카드 / pill / divider)
--border-strong:   #262d2a;   ← 강조 (theme toggle hover / hero CTA)
--border-accent:   #2a3a31;   ← 액센트 (현재 페이지 nav-item, LIVE 카드)
```

### 어디?
| 토큰 | 어디 |
| --- | --- |
| `--border` | 모든 카드 외곽선, pill 외곽선, divider |
| `--border-strong` | hover 시 외곽선 강조 |
| `--border-accent` | 현재 위치 nav 항목, LIVE 프로젝트 카드 (accent 색의 60% 섞은 톤) |

---

## 3. Text (글자색)

```css
--text:          #e6ece8;   ← 본문 (제목 / 강조 문구)
--text-muted:    #8a948f;   ← 부가 설명 (date / location / sub)
--text-faint:    #5e6663;   ← 더 흐림 (placeholder / hint)
--text-inverse:  #0b0d0c;   ← 액센트 배경 위 글자 (현재 거의 안 씀)
```

### 어디?
- `--text` — H1/H2/H3, 카드 title, KPI 값, post title
- `--text-muted` — 설명 문장, location, date, footer
- `--text-faint` — `mono faint` 클래스, 미미한 메타 정보

---

## 4. Accent (강조 / 액션)

```css
--accent:         #4ade80;                      ← 메인 강조 (Tailwind emerald-ish)
--accent-soft:    rgba(74, 222, 128, 0.12);    ← 액센트의 12% 투명 (배경 hint)
--accent-strong:  #22c55e;                      ← hover 시 더 진한 액센트
```

### 어디?
- `--accent` — `READ ON BLOG →` 같은 card-link, brand-dot, status ok, nav 활성 표시, LIVE 배지
- `--accent-soft` — brand-dot box-shadow glow
- `--accent-strong` — link hover, status badge text

### 강조 색 변경 레시피
**보라색 (Tailwind violet-400)** 으로 바꾸려면:
```css
--accent:         #a78bfa;
--accent-soft:    rgba(167, 139, 250, 0.12);
--accent-strong:  #8b5cf6;
--border-accent:  #6d28d966;
--ok:             #a78bfa;   /* 동시에 status ok 도 같이 바꾸려면 */
```

**오렌지 (Tailwind orange-400)**:
```css
--accent:         #fb923c;
--accent-soft:    rgba(251, 146, 60, 0.12);
--accent-strong:  #f97316;
--border-accent:  #c2410c66;
```

색상 더 자세한 레시피: [[colors]]

---

## 5. Status (상태)

```css
--ok:    #4ade80;   ← 정상 (Available, server ok, LIVE)
--warn:  #facc15;   ← 주의 (warning, expired soon)
--err:   #f87171;   ← 오류 (offline, failed)
--info:  #60a5fa;   ← 정보 (info badge)
```

이 값이 `StatusBadge status="ok|warn|err|info"` 에 매핑.

---

## 6. Spacing (여백)

```css
--space-0:  0;
--space-1:  4px;
--space-2:  8px
--space-3:  12px;
--space-4:  16px;
--space-5:  20px;
--space-6:  24px;
--space-7:  32px;
--space-8:  40px;
--space-9:  56px;
--space-10: 72px;
```

### 자주 쓰는 곳
| 토큰 | 쓰임 |
| --- | --- |
| `--space-2` 8px | Home grid gap, chip-row gap, card-list gap |
| `--space-3` 12px | nav 항목 사이, card 내부 sub-gap |
| `--space-4` 16px | card 내부 일반 gap, section-grid gap |
| `--space-5` 20px | card 내부 padding |
| `--space-6` 24px | page-header bottom margin |
| `--space-7` 32px | `.page` top padding |
| `--space-9` 56px | `.page` bottom padding, section header margin |

### 사이트 전체를 더 빡빡하게 / 더 여유롭게
모든 spacing 한 단계씩 줄이려면:
```css
--space-3:  10px;   /* 12 → 10 */
--space-4:  14px;   /* 16 → 14 */
--space-5:  18px;   /* 20 → 18 */
```
역으로 더 여유롭게는 위로 1단계씩 올림.

---

## 7. Radius (모서리)

```css
--radius-sm:    4px;     ← chip, code 인라인
--radius-md:    8px;     ← input, card 안 작은 박스
--radius-lg:    12px;    ← 메인 카드, project-detail-card
--radius-pill:  999px;   ← pill 모양 (CTA, nav, status badge, location)
```

### 모든 카드를 사각형으로 (radius 0)
```css
--radius-lg: 0;
```

### Pill 을 직사각형으로
```css
--radius-pill: 6px;
```

---

## 8. Shadow

```css
--shadow-card:  0 1px 0 rgba(0, 0, 0, 0.04);  ← 카드 미세 그림자
--shadow-pop:   0 4px 14px rgba(0, 0, 0, 0.20); ← 더 두드러진 그림자 (현재 미사용)
```

---

## 9. Typography (폰트)

```css
--font-sans:
  "Inter", system-ui, -apple-system, "Segoe UI", Roboto,
  "Helvetica Neue", Arial, "Apple SD Gothic Neo", "Pretendard",
  "Noto Sans KR", sans-serif;
--font-mono:
  "JetBrains Mono", ui-monospace, SFMono-Regular, "SF Mono",
  Menlo, Consolas, "Liberation Mono", monospace;
```

### Pretendard 우선으로
```css
--font-sans:
  "Pretendard", "Inter", system-ui, -apple-system,
  "Apple SD Gothic Neo", sans-serif;
```

### 자세히
[[typography]]

---

## 10. Text size (텍스트 크기)

```css
--text-xs:   11px;    ← chip, eyebrow label
--text-sm:   12.5px;  ← card-sub, mono faint
--text-base: 14px;    ← (현재 거의 미사용)
--text-md:   15px;    ← card-title, KPI label, 본문
--text-lg:   17px;    ← Stack card 의 chip, post title
--text-xl:   20px;    ← project-detail h2
--text-2xl:  26px;    ← page-title (Projects / Skills 등)
--text-3xl:  34px;    ← (Home hero h1 은 clamp 로 별도 처리)
```

### 글씨 전체 키우기
```css
--text-xs:   12px;
--text-sm:   13.5px;
--text-md:   16px;
--text-lg:   18px;
--text-xl:   22px;
--text-2xl:  28px;
```

---

## 11. Line-height & Letter-spacing

```css
--lh-tight:    1.25;    ← 제목용 (card-title)
--lh-snug:     1.4;     ← 한 줄 텍스트
--lh-normal:   1.55;    ← 본문 / description

--tracking-mono:   0.02em;
--tracking-label:  0.06em;   ← uppercase 라벨 자간
```

한국어가 빡빡하면 line-height 를 살짝 늘리세요 (1.55 → 1.65). 자세히 [[typography]].

---

## 12. Layout / 컨테이너

```css
--container-max:  1180px;            ← 사이트 최대 폭
--container-pad:  clamp(16px, 4vw, 32px);  ← 좌우 패딩 (반응형)
--topbar-h:       56px;              ← topbar 높이
--nav-h:          52px;              ← (현재 거의 미사용)
```

### 사이트 폭을 더 넓게 / 좁게
```css
--container-max: 1280px;   /* 더 넓게 */
--container-max: 960px;    /* 더 좁게 */
```

---

## 13. Transitions (모션)

```css
--t-fast:  120ms ease;    ← hover / pill 활성 표시
--t-base:  180ms ease;    ← 카드 hover 강조 등
```

페이지 전환 stagger 애니메이션은 이 토큰을 안 쓰고 layout.css 의 `@keyframes pageSlideUp` 에서 직접 정의. 자세히 [[animations]].

---

## 14. Dot grid (배경)

```css
--dot-color:    rgba(255, 255, 255, 0.045);
--dot-size:     1px;
--dot-spacing:  24px;
```

`body` 배경에 깔리는 격자점. 안 보이게 하려면 `--dot-color: transparent`.

---

## 15. 다크/라이트 모드 차이 (한눈에)

| 토큰 | 다크 | 라이트 |
| --- | --- | --- |
| --bg | `#0b0d0c` | `#f7f8f6` |
| --bg-card | `#0f1311` | `#ffffff` |
| --text | `#e6ece8` | `#161a18` |
| --text-muted | `#8a948f` | `#5b6360` |
| --accent | `#4ade80` | `#16a34a` |
| --border | `#1c211f` | `#e2e6e0` |
| --dot-color | `rgba(255,255,255,0.045)` | `rgba(0,0,0,0.06)` |

라이트 모드만 색을 따로 바꾸려면 `:root[data-theme="light"]` 블록 아래에서 변경.

---

연결: [[../hompage]] · [[colors]] · [[typography]] · [[cards]] · [[animations]]
