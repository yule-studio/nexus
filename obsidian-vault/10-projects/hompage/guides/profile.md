---
title: "hompage guide — 프로필 (자기 소개) 편집"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:35:00+09:00
tags: [project, hompage, guide, profile, data]
home_hub: hompage
---

# 프로필 편집

**[[../hompage|↑ hompage hub]]**

> 이름 / 역할 / 위치 / SNS 링크 / 아바타 등 자기 소개 데이터. 한 파일에 모여 있고 단순함.

---

## 파일

`src/data/profile.ts` — 한 파일만 만지면 끝.

---

## 전체 내용

```typescript
const githubUsername = "codwithyc";

export const profile = {
  handle: "yuchan.oh",
  name: "Oh Yuchan",
  role: "Backend Developer & DevOps Engineer",
  location: "Seoul, KR",
  status: { label: "Available · Q3 2026", tone: "ok" as const },
  bio:
    "백엔드 / 인프라 / 자율 에이전트를 짓습니다. " +
    "최근에는 LLM 코딩 에이전트와 self-host 홈랩 자동화에 시간을 쏟고 있어요.",
  links: {
    email: "mailto:oyuchan50@gmail.com",
    github: `https://github.com/${githubUsername}`,
    blog: "https://codingtips.tistory.com/",
    instagram: "https://www.instagram.com/oyuchan50/",
    linkedin: "https://www.linkedin.com/in/ohyuchan/",
  },
  githubUsername,
  avatarUrl: `https://github.com/${githubUsername}.png?size=512`,
  uptimeDays: 412,
  region: "home / seoul",
};
```

---

## 필드별 의미 + 어디서 보이는지

| 필드 | 어디 표시 | 어떻게 보이는지 |
| --- | --- | --- |
| `githubUsername` | (자동으로 `links.github` / `avatarUrl` 에 사용) | 한 값 바꾸면 다 따라옴 |
| `handle` | Topbar 좌상단 | `yuchan.oh` 같은 짧은 핸들 |
| `name` | Home hero 큰 제목 | 화면 큰 글씨 |
| `role` | Home hero 이름 아래 | "Backend Developer & DevOps Engineer" |
| `location` | Home hero 의 둥근 pill | "● Seoul, KR" |
| `status.label` | Topbar 우상단 status badge | "● Available · Q3 2026" |
| `status.tone` | status badge 색 | `"ok"` 녹색 / `"warn"` 노랑 / `"err"` 빨강 / `"info"` 파랑 |
| `bio` | (현재 직접 표시 안 됨, 나중에 About 페이지 추가 시 사용 예정) | — |
| `links.email` | Home hero 의 `Email` 버튼 + Contact 페이지 | `mailto:...` 클릭 시 메일 앱 |
| `links.github` | Home hero 의 `GitHub` 버튼 + GitHub 카드 | 외부 링크 |
| `links.blog` | Home hero 의 `Blog` 버튼 | 외부 링크 |
| `links.instagram` | Home hero 의 `Instagram` 버튼 | 외부 링크 |
| `links.linkedin` | Home hero 의 `LinkedIn` 버튼 | 외부 링크 |
| `avatarUrl` | ProfileCard 의 사진 | GitHub 의 프로필 사진 자동 (`github.com/{username}.png`) |
| `uptimeDays` | (현재 직접 표시 안 됨) | — |
| `region` | Contact 페이지의 Terminal 카드 | `region=home / seoul` |

---

## 자주 하는 수정

### 1. 이름을 한국어로
```typescript
name: "오유찬",
```

### 2. 역할 (role) 바꾸기
```typescript
role: "Full-stack Developer",
// or
role: "Frontend Engineer · UI Design",
```

### 3. 위치 (location) 변경
```typescript
location: "Tokyo, JP",
// or
location: "원격 · 한국",
```

### 4. Status badge 변경
```typescript
status: { label: "Looking for new role", tone: "info" as const },
// 가능한 tone: "ok" | "warn" | "err" | "info" | "muted"
```

### 5. 이메일 변경
```typescript
links: {
  email: "mailto:hello@example.com",
  ...
}
```

### 6. SNS 추가 (예: Twitter)
2단계.
- (1) `profile.ts` `links:` 안에 추가:
```typescript
links: {
  ...
  twitter: "https://twitter.com/yourhandle",
},
```
- (2) `src/pages/Home/Home.tsx` 의 hero CTA 블록에 한 덩어리 추가 ([[hero-ctas|이쪽 가이드]])

### 7. SNS 제거
- (1) `profile.ts` `links:` 에서 해당 줄 삭제
- (2) `Home.tsx` hero CTA 에서 해당 `<a>` 블록 삭제

### 8. GitHub 핸들 자체 변경
첫 줄만 바꾸면 자동으로 github 링크 / 아바타 / GitHub stats 다 따라옴:
```typescript
const githubUsername = "new-handle";
```

단, fetcher 의 환경변수 (`.github/workflows/deploy.yml` 의 `GITHUB_STATS_USERNAME`, `GITHUB_PROJECTS_SOURCES`) 도 같이 변경.

### 9. 아바타 사진 변경
GitHub 프로필 사진 자동 사용. GitHub 의 프로필 사진을 바꾸면 자동 반영.

다른 URL 로 강제하려면:
```typescript
avatarUrl: "https://example.com/my-avatar.png",
```

### 10. 한국어 길이 자동 줄바꿈 안 됨
한 줄이 너무 길어서 두 줄로 나누고 싶으면 문자열 결합 사용:
```typescript
bio:
  "첫 번째 문장. " +
  "두 번째 문장 — 줄바꿈 명시. ",
```

---

## 코딩 규칙 (TS 라서 지킬 것)

- 모든 string 은 큰따옴표 `"..."` 또는 작은따옴표 `'...'` (둘 다 OK, 사이트 일관성 위해 큰따옴표 통일)
- 객체 항목 사이 콤마 `,` 필수
- 마지막 항목 뒤에도 콤마 OK (trailing comma)
- 들여쓰기 — 2 스페이스 (탭 X)
- 백틱 ``` ` ``` 는 변수 끼워넣기 (`` `https://github.com/${name}` ``) 할 때만

### 잘못된 예
```typescript
links: {
  email: "mailto:..."     ← 콤마 없음
  github: "..."           ← syntax error
}
```

### 잘된 예
```typescript
links: {
  email: "mailto:...",
  github: "...",
},
```

---

## 변경 후 흐름

```bash
cd ~/local-dev/hompage

# 1) dev 서버로 미리 보기
npm run dev
# 브라우저에서 변경 확인

# 2) 잘 됐으면 commit + push
git add src/data/profile.ts
git commit -m "✏️ 프로필 — 이름 / 역할 업데이트"
git push origin main

# 3) 5분 후 사이트 반영
```

---

## 망쳤을 때

```bash
# 아직 commit 안 했으면
git checkout src/data/profile.ts

# 이미 commit/push 했으면
git log --oneline                 # 직전 정상 commit hash 확인
git revert <bad-commit-hash>
git push origin main
```

---

연결: [[../hompage]] · [[../how-to-edit]] · [[hero-ctas]] · [[../styling/tokens]]
