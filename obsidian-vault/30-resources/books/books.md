---
title: "Books — 개발 서적 카탈로그 (hub)"
kind: knowledge
status: current
created_at: 2026-05-26T19:00:00+09:00
updated_at: 2026-06-02T11:30:00+09:00
tags: [resources, books, hub]
home_hub: resources
---

# Books — 개발 서적 카탈로그

**[[../resources|↑ 30-resources]]**

> 외부 자료 요약 X. 책 + 구매 링크만. 카테고리별 분리 + 입문/중급/고급 marker.

## 1. 카테고리

| 카테고리 | 진입점 |
| --- | --- |
| Backend | [[backend/backend]] |
| Frontend | [[frontend/frontend]] |
| DevOps / Infra | [[devops/devops]] |
| Architecture | [[architecture/architecture]] |
| Database / Data | [[database/database]] |
| Computer Science | [[computer-science/computer-science]] |
| Security | [[security/security]] |
| AI / ML | [[ai-ml/ai-ml]] |
| Soft Skills / Career | [[soft-skills/soft-skills]] |
| Testing / Quality | [[testing/testing]] |
| 🆓 **Free / Open 무료본** | [[free-open-books]] |

> [[free-open-books]] — 합법 무료 공개본만 모은 cross-cutting 인덱스. 공식 읽기
> 링크 + (있으면) PDF/GitHub 소스 + **라이선스(클론 가능 여부)**. 불법 사본 X.

## 2. 표준 노트 구조

각 카테고리 노트는 다음 구조:

```markdown
---
title: "<카테고리> — 책 카탈로그"
kind: knowledge
status: current
created_at: ...
tags: [resources, books, <category>]
home_hub: resources
---

# <카테고리>

## 입문 (Beginner)
- **<제목>** (저자, 출판사, 년도)
  - 주제: <1줄>
  - 공식: [<출판사 공식 site>](...)  (있으면)
  - 🇰🇷 [<한국어판 검색>](https://search.kyobobook.co.kr/search?keyword=...)
  - 원서: [<출판사 또는 Amazon>](...)

## 중급 / 고급 / 무료
- ...
```

## 3. 구매 링크 컨벤션

- **한국어판** 🇰🇷: `https://search.kyobobook.co.kr/search?keyword=<제목>` (교보문고 검색 — 책 제목 또는 저자명으로 검색)
- **원서**: 출판사 공식 site URL 우선 (O'Reilly / Manning / Pragmatic / No Starch / Wiley / Apress) → 없으면 Amazon
- **무료 공개**: 공식 site URL (검증 가능)
- **공식 docs** 가 사실상 책 역할: 공식 site 로 표시 (예: K8s, React, Next.js, Tailwind)

### 왜 검색 URL?
- 책 별 product ID 가 시간 따라 바뀜 (재출간 / 개정 / 한국 시장 철수)
- 검색 URL = 영원히 작동 + 최신 결과 + 유사 책 같이 노출

## 4. marker

- ⭐ = 해당 카테고리 essential
- 🆕 = 최근 5년 (2020+) 출간
- 🇰🇷 = 한국어판 있음
- 🆓 = 무료 공개
- 📖 = 두꺼움 (1000+ 페이지)

## 5. 운영 규칙

1. 외부 요약 / 발췌 작성 금지 — 본 hub 는 카탈로그 only.
2. 깊이 학습한 책은 별도 노트 (`30-resources/books/<category>/<book-slug>.md`) 로 추출 후 요약.
3. 죽은 링크 발견 시 즉시 교체 또는 제거.
4. 새 책 추가 시 marker + 한 줄 주제 필수.

## 6. 관련

- [[../resources]] — 30-resources hub
- [[../../00-inbox/links/general/books]] — 빠른 reference 링크 (요약 X)
- [[../articles]] / [[../lectures]] / [[../official-docs]] — 다른 자료 카테고리
