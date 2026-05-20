---
title: "hompage 직접 수정 가이드 — FE 무지 기준"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T10:20:00+09:00
tags: [project, hompage, frontend, edit-guide, beginner]
home_hub: hompage
related:
  - "[[hompage]]"
  - "[[architecture]]"
  - "[[deploy]]"
  - "[[troubleshooting]]"
---

# hompage 직접 수정 가이드 — FE 무지 기준

**[[hompage|↑ hompage hub]]**

> "JS 도 TS 도 React 도 모르겠다" 사용자 기준. 케이스별로 어떤 파일의 어떤 줄을 바꾸면 되는지 그대로 적었습니다.

---

## 0. 시작하기 전 (반드시 5번 읽고 따라하세요)

### 0.1 환경 준비 (1번만)
```bash
cd ~/local-dev/hompage
npm install           # 처음 한 번. 5분 정도. 의존성 설치.
```

### 0.2 작업 흐름 (매번)
```bash
# 1) 작업 시작 전 — 최신 상태 받기
cd ~/local-dev/hompage
git pull origin main

# 2) 코드 수정 — 아래 케이스들 참고

# 3) 로컬에서 미리 보기
npm run dev
# → http://localhost:5173 (또는 5174) 열고 확인
# Ctrl+C 로 종료

# 4) 잘 됐으면 commit + push
git add .
git commit -m "✏️ 자기 이름 수정"   # 메시지 자유
git push origin main

# 5) 5분 기다리면 https://yule-studio.github.io/hompage/ 갱신
```

### 0.3 망쳤을 때 (rollback)
```bash
# 아직 commit 안 한 상태면
git checkout .              # 전부 원래대로

# 이미 commit 했지만 push 안 했으면
git reset --hard HEAD~1     # 직전 commit 취소

# 이미 push 했으면 — 이전 commit 으로 되돌리기
git log --oneline           # 돌아갈 commit hash 확인
git revert <commit-hash>    # 안전한 되돌리기 (새 commit 으로 취소)
git push origin main
```

---

## 1. 프로필 바꾸기

**파일**: `src/data/profile.ts`

전체 내용을 보면 다음과 같은 구조:

```typescript
const githubUsername = "codwithyc";

export const profile = {
  handle: "yuchan.oh",                              ← 화면 좌상단 표시
  name: "Oh Yuchan",                                 ← Home hero 큰 제목
  role: "Backend Developer & DevOps Engineer",       ← 이름 아래 설명
  location: "Seoul, KR",                             ← Seoul, KR pill
  status: { label: "Available · Q3 2026", tone: "ok" }, ← 우상단 status badge
  bio: "백엔드 / 인프라 / 자율 에이전트를 짓습니다. ...",
  links: {
    email: "mailto:oyuchan50@gmail.com",             ← Email 버튼 동작
    github: `https://github.com/${githubUsername}`,  ← 자동으로 codwithyc 채움
    blog: "https://codingtips.tistory.com/",         ← Blog 버튼
    instagram: "https://www.instagram.com/oyuchan50/",
    linkedin: "https://www.linkedin.com/in/ohyuchan/",
  },
  githubUsername,
  avatarUrl: `https://github.com/${githubUsername}.png?size=512`,
  uptimeDays: 412,
  region: "home / seoul",
};
```

### 바꾸기 규칙
- 큰 따옴표 `"..."` 안의 텍스트만 바꿉니다.
- 영어든 한국어든 OK.
- 콤마 `,` 와 세미콜론 `;` 는 그대로 두세요.

### 예시: 이름을 "오유찬" 으로
```typescript
name: "오유찬",
```

### GitHub 핸들 (codwithyc) 자체를 바꾸려면
첫 줄 `const githubUsername = "codwithyc";` 만 바꾸세요. 다른 곳은 자동 따라옵니다.

---

## 2. Hero 의 CTA 버튼 추가 / 제거

**파일**: `src/pages/Home/Home.tsx`

70번째 줄 근처에 다음과 같은 블록이 있습니다:

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

### 제거하려면
원하는 `<a>...</a>` 블록 한 덩어리를 삭제. (예: Instagram 제거 → Instagram `<a>` 시작부터 `</a>` 까지 4줄 삭제)

### 새 버튼 추가 (예: Twitter)
1. 먼저 `src/data/profile.ts` 의 `links:` 안에 `twitter: "https://twitter.com/..."` 한 줄 추가
2. `Home.tsx` 의 위 블록 끝에 비슷한 형태로 한 덩어리 추가:
   ```tsx
   <a className="hero-cta" href={profile.links.twitter} target="_blank" rel="noreferrer">
     <RssIcon /> Twitter
   </a>
   ```
   아이콘은 임의로 `RssIcon` 같은 기존 것 재사용. 새 SVG 아이콘 만들기는 복잡해서 권장 안 함.

---

## 3. 자격증 / 수상 / 홈랩 목록 바꾸기

같은 패턴 — `src/data/` 폴더 아래 파일들. 각 파일 한 줄씩 추가/삭제.

### `src/data/certs.ts` — 자격증
```typescript
export const certs = [
  { name: "CKA — Certified Kubernetes Administrator", year: 2025, status: "active" },
  { name: "AWS Solutions Architect — Associate", year: 2024, status: "active" },
  // 한 줄 추가 예시:
  { name: "정보처리기사", year: 2026, status: "active" },
];
```

- `status` 는 `"active"` (현재 유효) / `"expired"` (만료) 두 가지.

### `src/data/awards.ts` — 수상
```typescript
export const awards = [
  { title: "Open Source Contribution Award", year: 2025 },
  { title: "지방 기능 경기 대회 장려상", year: 2023 },
];
```

### `src/data/homelab.ts` — 홈서버 상태
```typescript
export const hosts = [
  { name: "pi5-edge", cpu: 8, mem: 16, status: "ok" },
  { name: "nas-01",  cpu: 4, mem: 32, status: "ok" },
];

export const services = [
  { name: "Grafana", status: "ok" },
  { name: "Prometheus", status: "ok" },
  // ...
];
```
- `status` 는 `"ok"` / `"warn"` / `"err"` 3가지.

### `src/data/skills.ts` — 자기평가 스킬
GitHub topic 자동 추출이 잘 안 채우는 카테고리만 fallback 으로 사용됨. 가급적 GitHub repo 의 topics 를 채우는 게 진짜 데이터.

### `src/data/events.ts` — 캘린더 일정
```typescript
export const events = [
  { title: "K-Devcon Korea 2026", date: "2026-09-12", time: "14:00", kind: "conf" },
];
```

---

## 4. 프로젝트 목록 추가 / 제거

**원칙**: Projects 페이지의 카드들은 **GitHub 에서 자동으로 가져옵니다.** 직접 수정 불필요.

### 새 프로젝트를 hompage 에 노출시키려면
1. GitHub 에 새 public repo 만들기
2. README.md 작성
3. (선택) repo 의 About → Website 에 배포 URL 추가 → "Live" 배지 자동 부여
4. (선택) repo 의 About → Topics 에 `fastapi`, `docker` 같은 키워드 추가 → Skills 페이지에도 자동 분류
5. main 에 push → 다음 hompage deploy (약 5분) 후 사이트 갱신

### 자동으로 제외되는 repo
- private repo
- fork 된 repo
- archived 된 repo
- `.github` 같이 이름이 `.` 으로 시작하는 repo

이 규칙은 `scripts/fetch-github-projects.mjs` 에 있음 — 보통 안 만짐.

---

## 4-A. 프로젝트를 LIVE 로 표시하기

Projects 페이지의 카드는 두 가지 상태:
- **LIVE 배지 (강조 테두리)** — 실제 배포된 사이트가 있는 repo
- **상태 라벨 (active / shipped / archived 등)** — 그 외

이 차이는 **GitHub repo 의 "Website" (homepage) 필드 한 줄로 결정** 됩니다. 코드 수정 0줄.

### 방법 1 — GitHub UI 에서 (가장 쉬움)
1. https://github.com/{owner}/{repo} 접속 (예: `https://github.com/yule-studio/hompage`)
2. 우측 상단 **About** 박스 옆의 ⚙️ (톱니바퀴) 아이콘 클릭
3. **Website** 필드에 배포 URL 입력
   - 예: `https://yule-studio.github.io/hompage/`
   - 예: `https://my-app.vercel.app`
   - 예: `https://example.com`
4. **Save changes** 클릭
5. 다음 hompage deploy (약 5분) 후 사이트의 Projects 페이지에서 그 카드에 **LIVE** 배지 + 청록 테두리 등장 + footer 에 `↗ live` 링크

### 방법 2 — gh CLI 로 (여러 repo 한 번에)
```bash
gh repo edit yule-studio/hompage --homepage "https://yule-studio.github.io/hompage/"
gh repo edit yule-studio/yule-studio-agent --homepage "https://yule-agent.example.com"
```

### LIVE 해제 (배포 내렸을 때)
**Website** 필드를 비우면 (또는 ⚙️ 에서 **Save** 후 빈 값) 다음 deploy 부터 LIVE 배지 사라지고 일반 상태로 돌아감.
```bash
gh repo edit yule-studio/some-repo --homepage ""
```

### 어떤 기준으로 LIVE/Code-only 가 결정되나
`scripts/fetch-github-projects.mjs` 에서 GitHub API 의 `repo.homepage` 필드를 받아 다음과 같이 처리:

```javascript
const homepageUrl = typeof repo.homepage === "string" && repo.homepage.trim()
  ? repo.homepage.trim()
  : undefined;

// status 자동 결정:
//   archived  → "archived"
//   homepage 있음 → "shipped"
//   그 외      → "active"
```

그리고 사이트 측 (`src/pages/Projects/Projects.tsx`) 에서:
```typescript
function isLive(p: Project): boolean {
  return Boolean(p.homepageUrl);
}
```
→ `homepage` 가 비어 있으면 Code-only, 있으면 LIVE.

### 빠른 확인 — 현재 LIVE 인 repo 보기
```bash
gh repo list yule-studio --json name,homepageUrl --jq '.[] | select(.homepageUrl != "") | .name'
gh repo list codwithyc  --json name,homepageUrl --jq '.[] | select(.homepageUrl != "") | .name'
```

### 일반적인 패턴
| repo 종류 | Website 에 넣을 값 | 결과 |
| --- | --- | --- |
| GitHub Pages 로 배포한 정적 사이트 | `https://{owner}.github.io/{repo}/` | LIVE |
| Vercel 배포한 Next.js / React | `https://{project}.vercel.app` | LIVE |
| Netlify 배포 | `https://{project}.netlify.app` | LIVE |
| 자체 도메인 / 자가 호스트 | `https://example.com` | LIVE |
| 학습용 / 샘플 / 라이브러리 / 백엔드만 (UI 없음) | (비워둠) | Code-only |
| README 자기 자신 (예: `codwithyc/codwithyc` 프로필 repo) | (비워둠 권장) | Code-only |

---

## 5. 색이나 폰트 바꾸기

**파일**: `src/styles/tokens.css`

상단에 모든 색·여백·폰트가 변수 한 곳에 모여 있습니다:

```css
:root {
  --bg: #050607;                     ← 배경 검은색
  --bg-card: #0d1012;                ← 카드 배경
  --text: #e7e9eb;                   ← 본문 글자
  --text-muted: #8f9aa3;             ← 흐릿한 글자
  --accent: #5eead4;                 ← 강조 (Live 배지 / 링크) — 청록색
  --accent-strong: #2dd4bf;          ← hover 시 더 진한 색
  --border: #1b2024;
  --border-accent: #2dd4bf66;        ← accent 의 50% 투명

  --font-mono: "JetBrains Mono", monospace;
  --font-sans: "Inter", -apple-system, sans-serif;

  --space-1: 4px;
  --space-2: 8px;
  ...
}
```

### 강조 색 (현재 청록) → 보라색으로 바꾸기
```css
--accent: #a78bfa;          ← 보라
--accent-strong: #8b5cf6;
--border-accent: #8b5cf666;
```

### 글씨 더 크게
```css
--text-md: 16px;        ← 14px → 16px
--text-lg: 19px;        ← 17px → 19px
```

### Dark/Light 자동 모드
이미 작동합니다 — 시스템 설정 + 우상단 toggle 버튼.

---

## 6. 목차 항목 추가

**파일**: `src/components/Navigation/Navigation.tsx`

```tsx
const ITEMS = [
  { to: "/",          label: "Home" },
  { to: "/projects",  label: "Projects" },
  { to: "/skills",    label: "Skills" },
  { to: "/awards",    label: "Awards" },
  { to: "/certs",     label: "Certs" },
  { to: "/homelab",   label: "Homelab" },
  { to: "/calendar",  label: "Calendar" },
  { to: "/contact",   label: "Contact" },
];
```

새 항목 추가하려면 한 줄 추가:
```tsx
{ to: "/about",     label: "About" },
```

그런데 새로운 페이지 자체를 만들어야 해서 추가 작업 필요 (라우터 등록 + 페이지 컴포넌트). FE 무지 사용자에겐 권장 안 함. **기존 페이지의 텍스트만 바꾸는 것을 우선** 하세요.

---

## 7. 자주 발생하는 실수

| 증상 | 원인 / 해결 |
| --- | --- |
| `npm run dev` 했더니 빨간 글자가 좌악 나옴 | 콤마 `,` 를 깜빡 안 적었거나, 따옴표 `"` 짝이 안 맞는 경우. 에러 메시지 첫 줄에 파일명·줄번호 표시됨. |
| 사이트가 안 갱신됨 | (1) `git push` 했는지 확인 (2) GitHub Actions 가 성공했는지 https://github.com/yule-studio/hompage/actions 확인 (3) 브라우저 hard refresh (Cmd+Shift+R) |
| 한국어가 깨짐 | 파일 저장 시 UTF-8 인코딩 확인 (VSCode 라면 우하단에 표시). 거의 자동으로 됨. |
| 색이 안 바뀜 | tokens.css 만 바꾸고 dev 서버를 안 껐다 켰으면 hot-reload 가 잘 안 잡는 경우 있음. Ctrl+C → `npm run dev` 재시작. |

---

## 8. 직접 손대지 말 것 (위험도 높음)

- `src/App.tsx` — 라우터 등록. 페이지 추가/삭제 시만.
- `src/components/Card/Card.tsx` — 카드 컴포넌트 코어. 잘못 만지면 모든 카드 깨짐.
- `vite.config.ts` / `tsconfig.json` — 빌드 설정.
- `.github/workflows/*.yml` — 잘못 만지면 deploy 가 안 돔.
- `scripts/*.mjs` — Node.js + GitHub API 코드. 잘못 만지면 데이터 수집 실패.

이런 파일 손대고 싶을 땐 agent (이 vault 의 engineering-agent) 에게 부탁하세요. 변경 의도만 말하면 PR 까지 만들어 줍니다.

---

연결: [[hompage]] · [[architecture]] · [[data-sources]] · [[deploy]] · [[troubleshooting]]
