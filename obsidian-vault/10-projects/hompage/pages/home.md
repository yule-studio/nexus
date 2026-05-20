---
title: "hompage page — Home (대시보드 해부)"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T12:00:00+09:00
tags: [project, hompage, page, home, dashboard]
home_hub: hompage
---

# Home 페이지 해부

**[[../hompage|↑ hompage hub]]**

> 홈 화면의 각 카드 위치 + 어느 파일이 만드는지 + 어떻게 손대는지.

---

## 0. 화면 구조

```
┌─ Topbar (sticky) ───────────────────────────────────┐
│ ● yuchan.oh                  ● Available · Q3 2026  🌙 DARK │
└──────────────────────────────────────────────────────┘

┌─ Home shell (720px 가운데) ───────────────────────────┐
│                                                       │
│  Hero                                                 │
│  Oh Yuchan                                            │
│  Backend Developer · DevOps Engineer  ● Seoul, KR    │
│  [Email] [GitHub] [Blog] [Instagram] [LinkedIn]      │
│  ─────────────────────────────────────────────────   │
│                                                       │
│  Nav: ● Home  ● Projects  ● Skills ...               │
│                                                       │
│  Dashboard Grid (12-col)                              │
│  ┌────────┬──────────┬──────────┐                    │
│  │Profile │ GitHub   │ Awards   │                    │
│  │        ├──────────┼──────────┤                    │
│  │        │ Certs    │ Projects │                    │
│  ├────────┼──────────┴──────────┤                    │
│  │ Stack  │ Blog (featured)      │                    │
│  ├────────┼──────────┬──────────┤                    │
│  │Homelab │ Calendar │ Services │                    │
│  └────────┴──────────┴──────────┘                    │
└──────────────────────────────────────────────────────┘

┌─ Footer ──────────────────────────────────────────────┐
│ © 2026 yule-studio · self-hosted     build: dev      │
└──────────────────────────────────────────────────────┘
```

---

## 1. 파일

| 영역 | 파일 |
| --- | --- |
| Topbar (전체 페이지) | `src/components/Topbar/Topbar.tsx` |
| Hero + 카드 grid | `src/pages/Home/Home.tsx` |
| ProfileCard | `src/components/ProfileCard/ProfileCard.tsx` |
| Footer | `src/components/Layout/Layout.tsx` |
| 데이터 | `src/data/profile.ts` / `awards.ts` / `certs.ts` / `homelab.ts` / `events.ts` / `skills.ts` |
| 동적 데이터 hook | `src/hooks/useProjects.ts` / `useBlog.ts` / `useGithubLanguages.ts` / `useGithubTopics.ts` |

---

## 2. Hero

```tsx
<section className="hero" aria-label="profile">
  <h1>{profile.name}</h1>
  <div className="hero-row">
    <span>{profile.role}</span>
    <span className="dot" aria-hidden />
    <span className="hero-loc">{profile.location}</span>
  </div>
  <div className="hero-ctas" aria-label="contact links">
    <a className="hero-cta" href={profile.links.email}><EmailIcon /> Email</a>
    <a className="hero-cta" href={profile.links.github} target="_blank">...</a>
    ...
  </div>
</section>
```

### 편집 가이드
- 이름 / 역할 / 위치 / SNS → [[../guides/profile]]
- CTA 버튼 추가/제거 / 스타일 → [[../guides/hero-ctas]]
- 폰트 크기 → [[../styling/typography#3-home-hero-의-큰-이름-폰트]]
- 좌우 여백 → `.home-shell .hero` ([[../styling/layout-grid#8-hero-home-의-자기소개]])

---

## 3. Nav (목차)

```tsx
<Navigation />
```

Navigation 컴포넌트 → [[../styling/layout-grid#9-nav-목차-pill]]

### 항목 추가/제거
`src/components/Navigation/Navigation.tsx`:
```tsx
const ITEMS = [
  { to: "/",          label: "Home" },
  { to: "/projects",  label: "Projects" },
  ...
];
```

---

## 4. Dashboard Grid — 10 카드 차례대로

### 4.1 ProfileCard
- **span**: sm (4col × 3row)
- **컴포넌트**: `<ProfileCard />`
- **데이터**: `profile.avatarUrl` (GitHub 아바타), `profile.name` 끝 부분
- **편집**: 사진 → `profile.ts` `avatarUrl`. 디자인 → `src/components/ProfileCard/ProfileCard.tsx`

### 4.2 GitHub 카드
- **span**: sm
- **className**: `home-github-card`
- **데이터**: `githubStats` (`public/github-stats.json` 자동)
- **표시**: REPOS / STARS / PRS / COMMITS / CONTRIB / ISSUES
- **편집**:
  - KPI 추가 → `src/data/githubStats.ts` 의 `githubStatItems` 배열
  - 라벨만 바꾸기 → 같은 배열의 `label`
  - 카드 자체 스타일 → `.home-github-card`

### 4.3 Awards 카드
- **span**: sm
- **데이터**: `data/awards.ts` 의 `awards[0]` (최신)
- **표시**: LATEST · 연도 + title
- **편집**: [[../guides/lists-data#2-data-awards-ts-수상]]

### 4.4 Certs 카드
- **span**: sm
- **데이터**: `data/certs.ts` `certs.slice(0, 3)` 처음 3개
- **표시**: 자격증 3개 (이름 + 연도)
- **편집**: [[../guides/lists-data#1-data-certs-ts-자격증]]

### 4.5 Projects 카드 (Selected)
- **span**: sm
- **데이터**: `useProjects()` 에서 active+shipped 의 처음 2개 (overflow 방지)
- **표시**: 01/02 + 이름 + 연도
- **편집**:
  - 보여줄 개수 → `selectedProjects.slice(0, 2)` 의 숫자
  - 정렬 / 필터 기준 → `selectedProjects = projects.filter(...)` 변경

### 4.6 Blog 카드 (Featured · Latest)
- **span**: md (8col × 3row — Stack 옆에 함께)
- **데이터**: `useBlog().featured` — Tistory RSS 의 최신 글
- **표시**: FEATURED · 날짜 + 제목 + 설명 + read on blog 링크
- **편집**:
  - 글자 잘림 → [[../styling/typography#6-한국어-descender-잘림]]
  - description 길이 → `scripts/fetch-blog.mjs` 의 `.slice(0, 220)` 숫자
  - 다른 블로그 → workflow 의 `BLOG_URL`

### 4.7 Stack 카드 (Top langs)
- **span**: sm
- **className**: `home-stack-card`
- **데이터**: `useGithubLanguages().languages.slice(0, 4)` 상위 4개
- **표시**: language chip + 색 dot
- **편집**:
  - 갯수 → `slice(0, 4)` 숫자 (Stack 카드 폭 한계 — 4~5)
  - bar / percent 표시 → 토글 가능 (`{lang.percent > 0 ? ... : null}`)
  - 자세히 → [[../guides/skills#3-home-의-stack-카드]]

### 4.8 Homelab 카드
- **span**: sm
- **데이터**: `data/homelab.ts` 의 `hosts.slice(0, 3)`
- **표시**: 호스트 3개 + 상태 dot
- **편집**: [[../guides/lists-data#3-data-homelab-ts-자가호스팅]]

### 4.9 Calendar 카드
- **span**: sm
- **데이터**: `data/events.ts` 의 `events[0]` (다음)
- **표시**: 날짜 / 시간 + 제목
- **편집**: [[../guides/lists-data#4-data-events-ts-캘린더-이벤트]]

### 4.10 Services 카드
- **span**: sm
- **className**: `home-services-card`
- **데이터**: `data/homelab.ts` `services.slice(0, 2)`
- **표시**: 2개 서비스 + ok 배지 + footer 의 99.98%
- **편집**: [[../guides/lists-data#3-data-homelab-ts-자가호스팅]]

---

## 5. 카드 추가하기

Dashboard 끝에 새 카드 추가:

`src/pages/Home/Home.tsx` 의 `<div className="grid">` 안 마지막에:
```tsx
<Card span="sm" hoverable ariaLabel="my-new">
  <div className="card-head">
    <span className="label">/ my new</span>
    <span className="chip">v1</span>
  </div>
  <h3 className="card-title">My Card</h3>
  <p className="card-sub">설명.</p>
  <div className="card-body">
    {/* 본문 */}
  </div>
  <div className="card-footer">
    <Link to="/projects" className="card-link">자세히</Link>
  </div>
</Card>
```

자동으로 grid 의 다음 빈 자리 (4col×3row) 에 배치.

---

## 6. 카드 제거

해당 `<Card>...</Card>` 블록 통째로 삭제. 다른 카드들이 자동으로 빈 자리 채움.

### 예: Services 카드 제거
```tsx
{/* row 10-12 : Homelab + Calendar + Services snapshot */}
<Card span="sm" hoverable ariaLabel="homelab">...</Card>
<Card span="sm" hoverable ariaLabel="calendar">...</Card>
{/* 아래 Services 통째 삭제 */}
{/*
<Card span="sm" hoverable ariaLabel="services" className="home-services-card">
  ...
</Card>
*/}
```

`/* ... */` 로 주석 처리해도 OK (나중에 복구 쉬움). 영구 제거면 그냥 삭제.

---

## 7. 카드 순서 바꾸기

JSX 안 `<Card>` 의 등장 순서 = 화면 grid 순서. 위에서 아래로 자동 배치.

원하는 위치로 `<Card>...</Card>` 블록 옮기기.

---

## 8. 카드 크기 변경 (sm → md / xl)

```tsx
<Card span="sm">    →    <Card span="md">
<Card span="md">    →    <Card span="xl">
```

span 종류 → [[../styling/cards#3-span-변형-크기]]

---

## 9. Home grid 자체 변경

`src/styles/layout.css` 의 `.home-shell .grid`:
```css
.home-shell .grid {
  grid-auto-rows: 68px;   /* 행 높이 — 줄이면 카드 더 빡빡 */
  gap: 6px;               /* 카드 사이 간격 */
}
```

전체 grid 동작 → [[../styling/layout-grid#5-grid-12-column-dashboard]]

---

## 10. 데이터가 안 나옴

| 카드 | 비어 보이는 경우 | 원인 / 해결 |
| --- | --- | --- |
| ProfileCard | 사진 안 나옴 | `profile.avatarUrl` URL 잘못 / GitHub 사진 비공개 |
| GitHub KPI | 모든 값이 0 | `public/github-stats.json` 가 fallback. deploy 다시 |
| Awards / Certs / Homelab | 비어 보임 | `data/*.ts` 의 배열이 비어 있음 |
| Projects | "sync pending" | `public/projects.json` 가 fallback. deploy 다시 |
| Blog | 옛날 글 | `public/blog.json` 가 stale. deploy 다시 |
| Stack | "items" 만 나오고 chip 4 개가 다 자기평가 텍스트 | `useGithubLanguages` 가 비어서 fallback. deploy 다시 |

---

## 11. Home 만의 특별 wrapper — home-shell

Home.tsx 는 다른 페이지와 다르게 `<div className="home-shell">` 으로 wrap.

```tsx
return (
  <div className="home-shell">
    <section className="hero">...</section>
    <Navigation />
    <div className="grid">...</div>
  </div>
);
```

→ `.home-shell .nav`, `.home-shell .grid`, `.home-shell .hero` 같은 scoped CSS 가 적용. 다른 페이지는 page-shell 사용 (`Layout.tsx` 에서 자동 wrap).

---

연결: [[../hompage]] · [[../architecture]] · [[../guides/profile]] · [[../guides/hero-ctas]] · [[../styling/cards]]
