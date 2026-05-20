---
title: "hompage guide — 리스트 데이터 (자격증/수상/Homelab/Calendar)"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T11:45:00+09:00
tags: [project, hompage, guide, data]
home_hub: hompage
---

# 리스트 데이터 편집

**[[../hompage|↑ hompage hub]]**

> `src/data/` 폴더의 자격증 / 수상 / Homelab / Calendar / Skills 파일 편집. 패턴 동일.

---

## 0. 공통 패턴

모든 데이터 파일은 다음과 같은 형태:

```typescript
export const items = [
  { name: "...", year: 2024, status: "active" },
  { name: "...", year: 2023, status: "expired" },
  // 한 줄 = 한 항목
];
```

### 규칙
- 항목 = `{ ... }` 한 줄 (또는 여러 줄)
- 항목 사이에 콤마 `,`
- 마지막 항목 뒤 콤마 OK
- 문자열은 `"..."` 으로 감쌈

---

## 1. `data/certs.ts` — 자격증

### 현재
```typescript
export const certs = [
  { name: "CKA — Certified Kubernetes Administrator", year: 2025, status: "active" },
  { name: "AWS Solutions Architect — Associate", year: 2024, status: "active" },
  { name: "Terraform Associate", year: 2024, status: "active" },
];
```

### 필드
| 필드 | 의미 | 가능한 값 |
| --- | --- | --- |
| `name` | 자격증 이름 | 자유 텍스트 (긴 이름은 `name.split(" (")[0]` 으로 카드에서 자름) |
| `year` | 취득 연도 | 숫자 4자리 |
| `status` | 상태 | `"active"` (현재 유효) / `"expired"` (만료) |

### 자격증 추가
```typescript
export const certs = [
  ...,
  { name: "정보처리기사", year: 2026, status: "active" },
];
```

### 자격증 제거
해당 줄 삭제. 다른 줄 영향 없음.

### 보이는 곳
- Home Certs 카드 — 최대 3개 (4번째부터 안 보임)
- Certs 페이지 (`/certs`) — 전체 목록

### Home 카드에서 더 많이 보이게
`src/pages/Home/Home.tsx` 에서 `certs.slice(0, 3)` 의 숫자를 키움. 단 sm 카드 (~150px 본문) 라 3개가 한계.

---

## 2. `data/awards.ts` — 수상

### 현재
```typescript
export const awards = [
  { title: "Open Source Contribution Award", year: 2025 },
  { title: "지방 기능 경기 대회 장려상", year: 2023 },
];
```

### 필드
| 필드 | 의미 |
| --- | --- |
| `title` | 수상명 |
| `year` | 연도 |

### 추가
```typescript
{ title: "전국 SW 콘테스트 우수상", year: 2026 },
```

### 보이는 곳
- Home Awards 카드 — `awards[0]` 최신 1개
- Awards 페이지 (`/awards`) — 전체 목록

---

## 3. `data/homelab.ts` — 자가호스팅

### 현재
```typescript
export const hosts = [
  { name: "pi5-edge", cpu: 8, mem: 16, status: "ok" },
  { name: "nas-01",   cpu: 4, mem: 32, status: "ok" },
  // ...
];

export const services = [
  { name: "Grafana", status: "ok" },
  { name: "Prometheus", status: "ok" },
  // ...
];
```

### `hosts` 필드
| 필드 | 의미 | 값 |
| --- | --- | --- |
| `name` | 호스트 이름 | 자유 |
| `cpu` | 코어 수 | 숫자 |
| `mem` | RAM (GB) | 숫자 |
| `status` | 상태 | `"ok"` / `"warn"` / `"err"` |

### `services` 필드
| 필드 | 의미 | 값 |
| --- | --- | --- |
| `name` | 서비스 이름 | 자유 |
| `status` | 상태 | `"ok"` / `"warn"` / `"err"` |

### 추가 / 제거
한 줄씩 추가/삭제. 패턴 동일.

### 상태 색
- `"ok"` → 녹색 (`--ok`)
- `"warn"` → 노랑 (`--warn`)
- `"err"` → 빨강 (`--err`)

색 자체는 [[../styling/tokens]] 의 status 토큰에서.

### 보이는 곳
- Home Homelab 카드 — `hosts.slice(0, 3)` 3개
- Home Services 카드 — `services.slice(0, 2)` 2개
- Homelab 페이지 (`/homelab`) — 전체

---

## 4. `data/events.ts` — 캘린더 이벤트

### 현재
```typescript
export const events = [
  {
    title: "K-Devcon Korea 2026",
    date: "2026-09-12",
    time: "14:00",
    kind: "conf",
  },
  // ...
];
```

### 필드
| 필드 | 의미 | 값 |
| --- | --- | --- |
| `title` | 이벤트명 | 자유 |
| `date` | 날짜 | ISO `YYYY-MM-DD` 형식 |
| `time` | 시간 | `HH:MM` 또는 생략 (전일 이벤트면 안 적어도 됨) |
| `kind` | 종류 | `"conf"` / `"meetup"` / `"deadline"` / `"personal"` 등 자유 |

### 보이는 곳
- Home Calendar 카드 — `events[0]` 다음 이벤트 1개
- Calendar 페이지 (`/calendar`) — 전체

### 정렬
배열 순서 그대로 표시. 날짜 순으로 정렬 안 됨 → 직접 위에서부터 가까운 날짜 순으로 적기.

---

## 5. `data/skills.ts` — 자기평가 스킬

### 현재
```typescript
export const skills = [
  {
    group: "Languages",
    items: [
      { name: "Python", level: 92, note: "agent / data / FastAPI" },
      { name: "TypeScript", level: 80, note: "React / Node" },
      // ...
    ],
  },
  {
    group: "Backend",
    items: [
      { name: "FastAPI", level: 88 },
      // ...
    ],
  },
  {
    group: "DevOps",
    items: [
      { name: "Docker", level: 90 },
      // ...
    ],
  },
  {
    group: "AI / Agents",
    items: [
      { name: "LLM API (Anthropic / OpenAI)", level: 85 },
      // ...
    ],
  },
];
```

### 필드
| 필드 | 의미 |
| --- | --- |
| `group` | 카테고리 이름 (Languages / Backend / DevOps / AI / Agents) |
| `items[].name` | 스킬 이름 |
| `items[].level` | 자기평가 0~100 (bar 너비로 표시) |
| `items[].note` | 부가 설명 (선택) |

### 중요 — Skills 페이지의 실제 동작
Languages 와 Backend / DevOps / AI 는 **GitHub repo 자동 추출 우선**. `data/skills.ts` 는 토픽이 0개일 때만 fallback.

따라서:
- **Languages** — GitHub repo 의 코드 바이트 비율로 자동
- **Backend / DevOps / AI** — GitHub repo 의 topics + manifest + README + path + code-search 로 자동
- 해당 카테고리에 hit 0 이면 `data/skills.ts` 의 같은 그룹 표시

직접 데이터로 통제하고 싶으면:
- 해당 repo 의 GitHub UI 에서 Topics 에 `fastapi`, `docker`, `anthropic` 등 추가
- `scripts/fetch-github-topics.mjs` 의 `CATEGORIES` 매핑에 추가 (agent 부탁 권장)

---

## 6. `data/profile.ts` — 프로필 (별도)

자세히 [[profile]].

---

## 7. (deprecated) `data/posts.ts` — 블로그 글

원래 하드코딩 블로그 글 목록이었으나 Tistory RSS 자동 동기화로 대체. 현재 미사용. 안 만져도 됨.

블로그 동기화는 자동 — 자세히 [[../data-sources#blog]].

---

## 8. (deprecated) `data/projects.ts` — 프로젝트 목록

원래 하드코딩이었으나 GitHub 자동 동기화로 대체. 현재 미사용.

LIVE 설정 등 프로젝트 관련은 [[projects]].

---

## 9. 공통 — 변경 후 흐름

```bash
cd ~/local-dev/hompage
npm run dev                     # 로컬 미리보기
# 잘 됐으면
git add src/data/certs.ts       # 또는 awards.ts / homelab.ts ...
git commit -m "✏️ 자격증 — 정보처리기사 추가"
git push origin main
# 5분 후 사이트 반영
```

---

## 10. 망쳤을 때

```bash
git checkout src/data/certs.ts   # 변경 취소 (미커밋)
```

또는 dev 서버가 빨간 에러를 보여줘서 파일명 + 줄번호 알려줌 → 그 위치 가서 콤마 / 따옴표 확인.

---

연결: [[../hompage]] · [[profile]] · [[projects]] · [[../styling/cards]]
