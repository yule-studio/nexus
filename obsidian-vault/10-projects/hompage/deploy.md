---
title: "hompage deploy — push 부터 사이트 갱신까지"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T10:40:00+09:00
tags: [project, hompage, deploy, ci-cd]
home_hub: hompage
related:
  - "[[hompage]]"
  - "[[architecture]]"
  - "[[troubleshooting]]"
---

# hompage deploy

**[[hompage|↑ hompage hub]]**

> "push 하면 언제 사이트에 반영되나요?" 의 답 + 트리거 / 수동 / 트러블슈팅.

---

## 1. 배포 타임라인

| 단계 | 누가 | 소요 시간 | 누적 |
| --- | --- | --- | --- |
| 1. `git push origin main` | 사용자 | 1초 | 0:01 |
| 2. GitHub 가 push 이벤트 감지 → workflow 큐잉 | GitHub | 1~5초 | 0:06 |
| 3. Actions runner 부팅 + `npm ci` | GitHub Actions | 40초 | 0:50 |
| 4. `github:stats` | fetcher | 2초 | 0:52 |
| 5. `github:projects` (15+ repos README 까지) | fetcher | 3초 | 0:55 |
| 6. `github:languages` | fetcher | 3초 | 0:58 |
| 7. `github:topics` (5-pass — code-search 가 가장 느림) | fetcher | 1~3분 | 2:55 |
| 8. `blog:sync` (RSS + 각 글 페이지 fetch) | fetcher | 30초 | 3:25 |
| 9. `vite build` | vite | 30초 | 3:55 |
| 10. Pages 업로드 + CDN propagate | GitHub Pages | 30~60초 | 4:25~4:55 |
| **합계** | — | — | **약 5분** |

브라우저 캐시까지 고려하면 사용자가 새 데이터를 보기까지 **약 5~10분**.

## 2. 트리거 방식 3가지

### 2.1 자동 — main 에 push
가장 흔한 케이스. workflow 가 `on: push: branches: [main]` 으로 자동 발사.

```bash
git add .
git commit -m "✏️ 자기소개 수정"
git push origin main
```

### 2.2 수동 — Actions 탭에서 "Run workflow"
push 없이 트리거하고 싶을 때 (예: 의도적으로 fetcher 재실행).

1. https://github.com/yule-studio/hompage/actions
2. 좌측 `Deploy (GitHub Pages)` 클릭
3. 우측 `Run workflow` 드롭다운 → `Run workflow` 버튼

또는 CLI:
```bash
gh workflow run "Deploy (GitHub Pages)" --ref main
```

### 2.3 빈 commit 으로 강제
fetcher 데이터만 다시 받고 싶을 때:
```bash
cd ~/local-dev/hompage
git pull
git commit --allow-empty -m "🌏 데이터 강제 갱신"
git push origin main
```

### 2.4 실시간 트리거 (다른 repo 가 hompage 갱신)
원리: 외부 repo 가 hompage 의 `repository_dispatch` endpoint 를 POST 호출.

#### 셋업 절차 (1번만)
1. **fine-grained PAT 발급** (https://github.com/settings/personal-access-tokens/new)
   - Repository access: `yule-studio/hompage` 만
   - Permissions:
     - Contents → Read-only
     - Actions → Read and write
   - Lifetime: ≤ 366 days (yule-studio org 정책)
2. **시크릿 등록**
   - yule-studio 의 모든 org repo 에서 쓸 수 있게: https://github.com/organizations/yule-studio/settings/secrets/actions → "New organization secret" → Name: `HOMPAGE_DISPATCH_TOKEN`, Value: 위 PAT
   - 개인 repo (codwithyc/*) 는 각 repo settings → Secrets → Actions 에 같은 이름으로 추가
3. **notify 워크플로 복사**
   - hompage repo 의 `.github/notify-hompage.example.yml` 을 추적할 각 repo 의 `.github/workflows/notify-hompage.yml` 로 복사
   - 그 repo 에 push → 5분 후 hompage 갱신 자동

#### 어떤 repo 에 적용할지
빠른 갱신이 필요한 repo 만 (예: yule-studio-agent 같은 적극 개발 repo). 거의 안 변하는 repo 는 굳이 안 해도 됨 — hompage 의 main push 때마다 어차피 fetch 됨.

## 3. workflow 파일 보기

### `.github/workflows/deploy.yml` 핵심 부분
```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
  repository_dispatch:
    types: [github-data-refresh]
```
- `push` — 자동 트리거 (main 에 commit 도착)
- `workflow_dispatch` — Actions UI 의 수동 버튼
- `repository_dispatch` — 외부 webhook (실시간 트리거용)

### `.github/workflows/ci.yml`
PR / push 시 lint + typecheck + build 검사. 깨지면 알람.

## 4. Pages 설정 (1번만 해두면 끝)

이미 활성화됨:
- Source: GitHub Actions
- Build type: workflow
- URL: https://yule-studio.github.io/hompage/

만약 Settings → Pages 가 비활성화돼 있으면:
1. https://github.com/yule-studio/hompage/settings/pages
2. Source: `GitHub Actions` 선택
3. Save

또는 `gh api` 로:
```bash
gh api --method POST /repos/yule-studio/hompage/pages -f build_type=workflow
```

## 5. 시크릿 (Secrets) 정리

| 이름 | 어디 | 용도 |
| --- | --- | --- |
| `GH_STATS_TOKEN` | yule-studio org-level | fetcher 들이 GitHub API 호출용 PAT (366d 이내, codwithyc 발급) |
| `HOMPAGE_DISPATCH_TOKEN` | (선택) org-level 또는 각 추적 repo | 실시간 트리거용 PAT |

### PAT 갱신 (12개월마다)
1. 만료 임박 알림이 GitHub 에서 옴
2. https://github.com/settings/personal-access-tokens/{id} → "Regenerate"
3. 새 값으로 org secret 업데이트

## 6. 트러블슈팅

### deploy 가 실패함
1. https://github.com/yule-studio/hompage/actions → 실패한 run 클릭 → 어느 step 인지 확인
2. 흔한 원인:
   - **GitHub API 403** → PAT 만료 / 권한 부족. 위 5. 참고
   - **Pages "Not Found"** → Pages 가 비활성화. 위 4. 참고
   - **fetcher 0 hits** → 외부 서비스 일시 장애. 보통 재시도 (수동 트리거) 면 OK

### 사이트는 떴는데 데이터가 안 바뀜
1. 브라우저 hard refresh: Cmd+Shift+R (macOS), Ctrl+Shift+R (Windows)
2. 시크릿 창에서 열어보기
3. 그래도 안 되면 CDN 캐시 — 5~10분 더 기다림

### deploy 가 너무 오래 걸림 (>10분)
- code-search 가 가장 느림 (1~3분). 사용 안 하려면 `scripts/fetch-github-topics.mjs` 의 `CODE_SEARCH_PATTERNS` 를 빈 배열로
- 추적 repo 가 너무 많으면 `GITHUB_PROJECTS_MAX=10` 으로 제한

자세한 트러블슈팅은 [[troubleshooting]] 참고.

---

연결: [[hompage]] · [[architecture]] · [[data-sources]] · [[troubleshooting]]
