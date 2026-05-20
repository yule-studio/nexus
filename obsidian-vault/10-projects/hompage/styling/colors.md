---
title: "hompage styling — 색상 변경 레시피"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:05:00+09:00
tags: [project, hompage, styling, colors, css]
home_hub: hompage
related:
  - "[[tokens]]"
  - "[[../hompage]]"
---

# 색상 변경 레시피

**[[../hompage|↑ hompage hub]]** · **[[tokens|토큰 사전]]**

> "이 부분 색만 바꾸고 싶어요" 케이스별 정확한 위치 + 값.

---

## 0. 어떤 부분의 색을 바꾸고 싶나요?

| 영역 | 어디 만지기 |
| --- | --- |
| 전체 배경 (검은색) | [[#1-페이지-배경]] |
| 카드 배경 | [[#2-카드-배경]] |
| 강조 색 (녹색 액센트) | [[#3-강조-색-액센트]] |
| 글자 색 | [[#4-글자-색]] |
| 테두리 | [[#5-테두리]] |
| status (ok/warn/err) | [[#6-status-색]] |
| 특정 카드만 다른 색 | [[#7-특정-카드만-색-다르게]] |
| 라이트 모드만 따로 | [[#8-라이트-모드만-따로]] |

모든 작업은 `src/styles/tokens.css` 또는 `src/styles/layout.css` 만 수정. 코드 (TSX) 안 만짐.

---

## 1. 페이지 배경

### 현재
다크: `#0b0d0c` (거의 검정에 가까운 짙은 회색)

### 위치
`src/styles/tokens.css`:
```css
:root,
:root[data-theme="dark"] {
  --bg: #0b0d0c;          /* 여기 */
  ...
}
```

### 변경 예
| 원하는 분위기 | 값 |
| --- | --- |
| 더 깊은 검정 | `#000000` |
| 미드나잇 블루 | `#0a0e1a` |
| 다크 그린 | `#0a120e` |
| 다크 그레이 | `#1a1a1a` |
| 갈색 토 | `#120e0a` |

### 함께 조정 권장
배경을 너무 어둡게 / 밝게 만들면 카드 배경 (`--bg-card`) 도 비슷한 방향으로 조정해야 카드 윤곽이 보임. 예: `--bg: #000` 이면 `--bg-card: #0a0a0a` 정도.

---

## 2. 카드 배경

### 현재
- 카드 기본: `--bg-card: #0f1311`
- hover 시: `--bg-card-hover: #131816`

### 위치
`src/styles/tokens.css` `--bg-card` / `--bg-card-hover`.

### 변경 레시피
**카드를 더 두드러지게 (배경보다 살짝 밝게)**:
```css
--bg-card:       #181c1a;   /* 배경 #0b0d0c 보다 +13 정도 */
--bg-card-hover: #1e2421;
```

**카드를 거의 안 보이게 (배경과 같이)**:
```css
--bg-card:       transparent;
--bg-card-hover: rgba(255,255,255,0.03);
```
이러면 border 만으로 카드 구분.

---

## 3. 강조 색 (액센트)

가장 자주 바꾸는 항목. **메뉴 활성, READ ON BLOG → 링크, LIVE 배지, brand-dot, status ok** 등 곳곳에서 사용.

### 현재
녹색 (Tailwind emerald 계열):
```css
--accent:         #4ade80;
--accent-soft:    rgba(74, 222, 128, 0.12);
--accent-strong:  #22c55e;
--border-accent:  #2a3a31;
```

### 변경 레시피 (Tailwind 색 기준)

#### 보라 (violet)
```css
--accent:         #a78bfa;
--accent-soft:    rgba(167, 139, 250, 0.12);
--accent-strong:  #8b5cf6;
--border-accent:  #6d28d966;
```

#### 시안 (cyan)
```css
--accent:         #22d3ee;
--accent-soft:    rgba(34, 211, 238, 0.12);
--accent-strong:  #06b6d4;
--border-accent:  #0e749066;
```

#### 핑크 (pink)
```css
--accent:         #f472b6;
--accent-soft:    rgba(244, 114, 182, 0.12);
--accent-strong:  #ec4899;
--border-accent:  #be185d66;
```

#### 오렌지 (orange)
```css
--accent:         #fb923c;
--accent-soft:    rgba(251, 146, 60, 0.12);
--accent-strong:  #f97316;
--border-accent:  #c2410c66;
```

#### 옐로우 (amber)
```css
--accent:         #facc15;
--accent-soft:    rgba(250, 204, 21, 0.12);
--accent-strong:  #eab308;
--border-accent:  #a1640966;
```

#### 빨강 (red)
```css
--accent:         #f87171;
--accent-soft:    rgba(248, 113, 113, 0.12);
--accent-strong:  #ef4444;
--border-accent:  #99182066;
```

### 색 골라 만드는 법
1. Tailwind color picker (https://tailwindcss.com/docs/customizing-colors) 에서 `400` 단계 = 메인, `500` 단계 = strong 사용
2. `--accent-soft` 는 메인색의 RGB 값으로 변환 (예: `#a78bfa` → `rgba(167, 139, 250, 0.12)`)
3. `--border-accent` 는 진한 색 (`700` 단계) + `66` (40% 투명)

### 함께 바꿀까 말까
`--ok` 도 액센트와 같은 값. status badge 의 ok 도 같이 바뀌게 하려면:
```css
--ok: #a78bfa;   /* accent 와 같이 보라로 */
```

분리하고 싶으면 `--ok` 는 녹색 유지 (디폴트 `#4ade80`).

---

## 4. 글자 색

### 현재
```css
--text:        #e6ece8;   /* 본문 */
--text-muted:  #8a948f;   /* 부가 설명 */
--text-faint:  #5e6663;   /* 흐림 */
```

### 변경 예

#### 더 부드러운 본문 (살짝 따뜻)
```css
--text:        #eae5dc;
--text-muted:  #948d82;
```

#### 거의 흰색 (강한 대비)
```css
--text:        #ffffff;
--text-muted:  #a3a3a3;
--text-faint:  #6b6b6b;
```

#### 차가운 파랑 톤
```css
--text:        #e0eaf5;
--text-muted:  #7b8ba0;
--text-faint:  #4d5868;
```

---

## 5. 테두리

### 현재
```css
--border:         #1c211f;   /* 카드, pill 외곽선 */
--border-strong:  #262d2a;   /* hover 시 */
--border-accent:  #2a3a31;   /* 액티브 nav, LIVE 카드 */
```

### 변경 예

#### 더 또렷한 테두리 (대비 강)
```css
--border:         #2a2f2c;
--border-strong:  #3c4340;
```

#### 거의 안 보이게 (미니멀)
```css
--border:         #131816;
--border-strong:  #1c211f;
```

---

## 6. Status 색

`StatusBadge status="ok|warn|err|info"` 에서 사용.

### 현재
```css
--ok:    #4ade80;   /* 녹색 */
--warn:  #facc15;   /* 노랑 */
--err:   #f87171;   /* 빨강 */
--info:  #60a5fa;   /* 파랑 */
```

### 변경 예 (더 차분한 톤)
```css
--ok:    #34d399;   /* emerald-400 */
--warn:  #fbbf24;   /* amber-400 */
--err:   #fb7185;   /* rose-400 */
--info:  #818cf8;   /* indigo-400 */
```

### 같이 바뀌는 것
- Home 의 GitHub `active` 배지
- Homelab 의 host 상태 dot
- Services 의 ok/warn/err 배지

---

## 7. 특정 카드만 색 다르게

특정 카드 (예: Home 의 GitHub 카드만) 만 다른 색으로 하려면 className 으로 scoped 스타일.

### Home 의 GitHub 카드만 보라 테두리
`src/styles/layout.css` 끝에 추가:
```css
.home-github-card {
  border-color: #8b5cf6;
}
.home-github-card .card-title {
  color: #a78bfa;
}
```

### Home 의 Stack 카드만 배경 그라데이션
```css
.home-stack-card {
  background: linear-gradient(
    180deg,
    color-mix(in srgb, var(--accent) 8%, var(--bg-card)) 0%,
    var(--bg-card) 100%
  );
}
```

### 어떤 className 이 어디?
- `home-github-card` — Home GitHub 카드
- `home-services-card` — Home Services 카드
- `home-stack-card` — Home Stack 카드
- `project-card` — Projects 목록 카드 (전부)
- `project-card--live` — Projects 카드 중 LIVE 인 것
- `project-detail-card` — Project 상세 페이지 큰 카드

className 찾는 법:
1. 브라우저에서 해당 요소 우클릭 → 검사 (Inspect)
2. element 의 `class="..."` 확인

---

## 8. 라이트 모드만 따로

라이트 모드는 `:root[data-theme="light"]` 블록에서 별도 정의.

### 라이트 모드 액센트만 다른 색
`tokens.css` 의 라이트 블록에서:
```css
:root[data-theme="light"] {
  --accent:         #7c3aed;   /* 라이트 모드는 violet-600 */
  --accent-strong:  #6d28d9;
  --border-accent:  #c4b5fd;
}
```

다크 모드는 영향 안 받음.

### 라이트 모드만 더 흰 배경
```css
:root[data-theme="light"] {
  --bg:       #ffffff;
  --bg-card:  #fafafa;
}
```

---

## 9. 한 줄 모음 — 자주 같이 바꾸는 그룹

### A. "톤 다운" (전체적으로 더 차분하게)
```css
--bg:             #0a0c0b;
--bg-card:        #0e1110;
--text:           #d8dedb;
--text-muted:     #7c8580;
--accent:         #6ee7b7;       /* emerald-300 — 톤 다운 */
--accent-strong:  #34d399;
--border:         #181d1b;
```

### B. "비비드" (더 화려하게)
```css
--accent:         #00ffaa;       /* 네온 그린 */
--accent-soft:    rgba(0, 255, 170, 0.20);
--accent-strong:  #00ddaa;
--border-accent:  #00ffaa66;
```

### C. "스튜디오 그레이" (회색만)
```css
--bg:        #0c0c0c;
--bg-card:   #161616;
--accent:    #d4d4d4;
--accent-strong: #ffffff;
--border:    #262626;
--border-accent: #525252;
```

---

## 10. 변경 후 확인

```bash
cd ~/local-dev/hompage
npm run dev   # http://localhost:5173 열고 확인
```

문제 없으면:
```bash
git add src/styles/tokens.css
git commit -m "🎨 액센트 색 → 보라"
git push origin main
# 5분 후 사이트 반영
```

---

연결: [[../hompage]] · [[tokens]] · [[typography]] · [[cards]]
