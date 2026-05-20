---
title: "hompage guide — Hero CTA 버튼 (Email/GitHub/Blog/Instagram/LinkedIn)"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:40:00+09:00
tags: [project, hompage, guide, hero, cta]
home_hub: hompage
---

# Hero CTA 버튼 편집

**[[../hompage|↑ hompage hub]]**

> Home 첫 화면의 자기소개 아래 둥근 버튼들 (`Email` / `GitHub` / `Blog` / `Instagram` / `LinkedIn`) 을 추가 / 제거 / 스타일링.

---

## 0. 어디 있는가

| 부분 | 파일 | 위치 |
| --- | --- | --- |
| URL 데이터 | `src/data/profile.ts` | `links:` 안 |
| 버튼 JSX 렌더링 | `src/pages/Home/Home.tsx` | `<div className="hero-ctas">` 안 |
| 아이콘 (SVG 인라인) | `src/pages/Home/Home.tsx` | 파일 하단 `function EmailIcon()` 등 |
| 버튼 스타일 | `src/styles/layout.css` | `.hero-cta` |

---

## 1. 현재 코드 (Home.tsx)

```tsx
<div className="hero-ctas" aria-label="contact links">
  <a className="hero-cta" href={profile.links.email}>
    <EmailIcon /> Email
  </a>
  <a className="hero-cta" href={profile.links.github} target="_blank" rel="noreferrer">
    <GhIcon /> GitHub
  </a>
  <a className="hero-cta" href={profile.links.blog} target="_blank" rel="noreferrer">
    <RssIcon /> Blog
  </a>
  <a className="hero-cta" href={profile.links.instagram} target="_blank" rel="noreferrer">
    <IgIcon /> Instagram
  </a>
  <a className="hero-cta" href={profile.links.linkedin} target="_blank" rel="noreferrer">
    <LiIcon /> LinkedIn
  </a>
</div>
```

---

## 2. 버튼 추가 — 단계별

### 예: Twitter 추가

#### Step 1 — profile.ts 에 URL 추가
`src/data/profile.ts`:
```typescript
links: {
  email: "mailto:oyuchan50@gmail.com",
  github: `https://github.com/${githubUsername}`,
  blog: "https://codingtips.tistory.com/",
  instagram: "https://www.instagram.com/oyuchan50/",
  linkedin: "https://www.linkedin.com/in/ohyuchan/",
  twitter: "https://twitter.com/yourhandle",   // ← 추가
},
```

#### Step 2 — Home.tsx hero CTA 블록에 한 덩어리 추가
LinkedIn 다음 줄에:
```tsx
<a className="hero-cta" href={profile.links.twitter} target="_blank" rel="noreferrer">
  <RssIcon /> Twitter
</a>
```

(`RssIcon` 은 임시. 아래 4. 에서 진짜 Twitter 아이콘 만들기)

#### Step 3 — 저장 + dev 서버에서 확인
```bash
npm run dev
```

#### Step 4 — 커밋
```bash
git add src/data/profile.ts src/pages/Home/Home.tsx
git commit -m "✨ hero CTA — Twitter 추가"
git push origin main
```

---

## 3. 버튼 제거

### 예: Instagram 제거

#### Step 1 — Home.tsx 에서 `<a>` 블록 4줄 삭제
```tsx
<a className="hero-cta" href={profile.links.instagram} target="_blank" rel="noreferrer">
  <IgIcon /> Instagram
</a>
```
이 4줄 통째로 삭제.

#### Step 2 — (선택) profile.ts 의 instagram 도 삭제
```typescript
links: {
  ...
  instagram: "https://www.instagram.com/oyuchan50/",   ← 이 줄 삭제 (선택)
},
```

남겨둬도 무방 (안 쓰니 문제 없음). 정리 차원에서 삭제 권장.

#### Step 3 — (선택) 사용 안 하는 `IgIcon` 함수도 삭제
Home.tsx 하단의:
```tsx
function IgIcon() {
  return (
    <svg width="14" height="14" ...>
      ...
    </svg>
  );
}
```

---

## 4. 새 아이콘 만들기

각 아이콘은 SVG 함수. 16px 정도의 line icon 권장.

### 예: Twitter / X 아이콘
Home.tsx 하단에 추가:
```tsx
function XIcon() {
  return (
    <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor" aria-hidden>
      <path d="M18.244 2H21l-7.155 8.176L22 22h-6.728l-5.27-6.882L3.94 22H1.181l7.65-8.74L1 2h6.89l4.762 6.292L18.244 2zm-1.18 18.4h1.532L7.024 3.5H5.387L17.064 20.4z" />
    </svg>
  );
}
```

### 예: 유튜브
```tsx
function YtIcon() {
  return (
    <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor" aria-hidden>
      <path d="M23.498 6.186a3.016 3.016 0 0 0-2.122-2.136C19.505 3.545 12 3.545 12 3.545s-7.505 0-9.377.505A3.017 3.017 0 0 0 .502 6.186C0 8.07 0 12 0 12s0 3.93.502 5.814a3.016 3.016 0 0 0 2.122 2.136c1.871.505 9.376.505 9.376.505s7.505 0 9.377-.505a3.015 3.015 0 0 0 2.122-2.136C24 15.93 24 12 24 12s0-3.93-.502-5.814zM9.545 15.568V8.432L15.818 12l-6.273 3.568z" />
    </svg>
  );
}
```

### SVG 아이콘 사이트
- https://lucide.dev — line icon (`<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">...`)
- https://simpleicons.org — brand icon (fill 기반)
- https://heroicons.com — 아이콘 모음

복사 → `function NewIcon() { return (<svg ...>); }` 형태로 감싸기 → JSX 에서 `<NewIcon />` 호출.

### 주의
- `width="14" height="14"` 명시 (없으면 SVG 가 viewport 채움 → 화면 깨짐)
- `aria-hidden` 추가 (screen reader 읽기 제외)
- `class="..."` 아니라 `className="..."` (JSX 문법)

---

## 5. 버튼 스타일 변경

`src/styles/layout.css`:
```css
.hero-cta {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 9px 16px;
  border: 1px solid var(--border-strong);
  border-radius: var(--radius-pill);
  background: var(--bg-card);
  color: var(--text);
  font-size: var(--text-sm);
  font-weight: 500;
  transition: border-color var(--t-fast), background var(--t-fast), color var(--t-fast);
}
.hero-cta:hover {
  border-color: var(--border-accent);
  color: var(--accent);
  background: var(--bg-card-hover);
}
.hero-cta svg { width: 14px; height: 14px; flex-shrink: 0; }
```

### 더 크게
```css
.hero-cta {
  padding: 12px 20px;
  font-size: var(--text-md);
}
.hero-cta svg { width: 16px; height: 16px; }
```

### 사각형 (pill 아님)
```css
.hero-cta { border-radius: var(--radius-md); }
```

### 배경 채우기 (filled style)
```css
.hero-cta {
  background: var(--accent);
  color: var(--text-inverse);
  border-color: var(--accent);
}
.hero-cta:hover {
  background: var(--accent-strong);
  border-color: var(--accent-strong);
}
```

### 첫 번째 버튼만 강조 (primary)
```css
.hero-cta:first-child {
  background: var(--accent);
  color: var(--text-inverse);
  border-color: var(--accent);
}
```

### 버튼 사이 간격
`.hero-ctas` 컨테이너:
```css
.hero-ctas {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-2);   /* 8px — 현재 */
}
```
더 넓게: `gap: var(--space-3);` (12px).

---

## 6. 버튼 순서 바꾸기

Home.tsx 에서 `<a>` 블록 순서를 바꾸기만 하면 화면 순서도 바뀜.

```tsx
<div className="hero-ctas">
  <a className="hero-cta" href={profile.links.github}>...</a>   ← 1번
  <a className="hero-cta" href={profile.links.email}>...</a>    ← 2번
  ...
</div>
```

---

## 7. 모바일에서 버튼 행 처리

이미 `.hero-ctas { flex-wrap: wrap }` → 화면 좁으면 자동 줄바꿈.

모바일에서 더 작게 보이게:
```css
@media (max-width: 640px) {
  .hero-cta {
    padding: 7px 12px;
    font-size: var(--text-xs);
  }
}
```

---

## 8. 버튼에 hover tooltip 추가

JSX 의 `title` 속성:
```tsx
<a className="hero-cta" href={profile.links.email} title="hello@example.com">
  <EmailIcon /> Email
</a>
```

---

## 9. 외부 링크 vs 내부 링크

| 종류 | 패턴 |
| --- | --- |
| 외부 사이트 | `<a href="..." target="_blank" rel="noreferrer">` |
| 메일 / 전화 | `<a href="mailto:..." />` 또는 `<a href="tel:+82..." />` (target 생략) |
| 사이트 내부 페이지 | `<Link to="/projects">` (`a` 가 아니라 React Router 의 `Link`) |

---

## 10. 흔한 실수

| 증상 | 원인 |
| --- | --- |
| 아이콘이 화면 다 차지 | `<svg>` 에 width/height 안 넣음 |
| 버튼이 안 보임 | href 의 따옴표 누락 / 다른 곳의 syntax error |
| TypeScript 에러 `profile.links.twitter` 없음 | `profile.ts` 에 추가 안 함 (Step 1 빠뜨림) |
| 클릭해도 새 탭 안 열림 | `target="_blank"` 빠뜨림 |

---

연결: [[../hompage]] · [[profile]] · [[../styling/cards]] · [[../styling/typography]]
