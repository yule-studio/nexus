---
title: "hompage troubleshooting — 자주 만나는 문제"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T10:50:00+09:00
tags: [project, hompage, troubleshooting, beginner]
home_hub: hompage
related:
  - "[[hompage]]"
  - "[[how-to-edit]]"
  - "[[deploy]]"
---

# hompage troubleshooting

**[[hompage|↑ hompage hub]]**

> 실제로 부딪힌 문제와 그 해결. 새 문제 만나면 여기에 한 줄씩 추가하세요.

---

## A. 로컬 개발

### A1. `npm install` 이 너무 오래 걸려요
정상. 처음엔 3~5분. 캐시 있으면 30초.

### A2. `npm run dev` 했더니 빨간 에러
가장 흔한 원인: 데이터 파일 (`src/data/*.ts`) 수정하다가 syntax 깨짐.

#### 진단
에러 메시지 첫 줄에 파일명 + 줄번호 표시. 예:
```
src/data/profile.ts(13,3): error TS1005: ',' expected.
```
→ `profile.ts` 13번째 줄 3번째 칸 근처에 콤마 빠짐.

#### 자주 빠뜨리는 것
- 한 줄 끝 콤마 `,`
- 큰따옴표 `"..."` 짝 안 맞음
- 중괄호 `{ }` 짝 안 맞음

### A3. dev 서버는 떴는데 빈 화면
브라우저 콘솔 (Cmd+Option+I → Console) 에 에러 메시지 확인. 보통 import 경로 오타 또는 props 누락.

### A4. 색을 바꿨는데 안 반영됨
1. dev 서버 재시작 (Ctrl+C → `npm run dev`)
2. 브라우저 hard refresh (Cmd+Shift+R)
3. 그래도 안 되면 `tokens.css` 의 변수명을 잘못 적었을 가능성 (예: `--accent` → `--accent-color`)

### A5. `git push` 가 reject 됨
원인: 원격이 로컬보다 앞서 있음 (다른 곳에서 commit 됨).
```bash
git pull --rebase origin main
# 충돌 없으면 자동, 있으면 메시지 따라 해결
git push origin main
```

---

## B. 자동 데이터 (GitHub / Tistory)

### B1. 새 repo 가 사이트에 안 보임
체크리스트:
- [ ] repo 가 **public** 인지? (private 는 제외됨)
- [ ] fork 가 아닌지? (fork 는 자동 제외)
- [ ] archived 가 아닌지? (archived 도 제외)
- [ ] repo 이름이 `.` 으로 시작 안 하는지? (`.github` 등 자동 제외)
- [ ] hompage 의 last deploy 가 최근인지? (https://github.com/yule-studio/hompage/actions)

다 OK 인데도 안 보이면 → 수동 트리거: [[deploy#2-3-빈-commit-으로-강제]]

### B2. Python 비율이 너무 높음 / Go 가 안 보임
- 원인: yule-studio-agent 한 repo 가 9.7MB Python → 다른 언어가 < 1%
- 해결: 화면에 다 보이게 하려면 `MIN_PERCENT` 가 이미 0.01 로 설정돼 있어 모든 언어가 보임. 그래도 비율은 데이터의 진실 (Python 압도)
- 균형을 보고 싶으면 다른 언어 repo 에 코드를 더 push

### B3. Skills 의 AI · Agents 가 비어 있음
- 원인: 해당 카테고리 (Anthropic / OpenAI / MCP 등) framework 가 어떤 repo 의 deps / README / 코드에도 안 잡힘
- 해결: 빠른 방법 — 해당 repo 의 GitHub UI 에서 Topics 에 `anthropic`, `claude-code`, `mcp` 등 직접 추가

### B4. Blog 의 view count 가 0
- 원인: 현재 Tistory 테마가 글별 view count 를 public HTML 로 노출 안 함 (사이드바의 전체 누적만 있음)
- 대안: featured 는 "최신 글" 로 동작 중 — RSS 첫 항목
- 진짜 view count 가 필요하면 Tistory 테마 변경 (글 페이지에 viewCounter 노출하는 테마)

### B5. Blog description 에 `<p>` 같은 마크업이 보임
- 원인: 옛 `blog.json` 캐시 — stripHtml fix 전에 생성됨
- 해결: 로컬에서 `npm run blog:sync` 다시 → 또는 deploy 한 번 더

### B6. deploy 가 403 Forbidden 으로 실패
- 원인: PAT 가 yule-studio org 의 fine-grained PAT 366일 정책 위반
- 해결: [[deploy#5-시크릿-secrets-정리]] 참고해서 PAT 재발급 + org secret 업데이트

### B7. deploy 가 "Get Pages site failed: Not Found"
- 원인: GitHub Pages 가 repo 에 활성화 안 됨
- 해결: [[deploy#4-pages-설정]] 참고

---

## C. 화면 / UI

### C1. 카드 안 글자가 잘림
- 원인: card 가 `overflow: hidden` 인데 내용이 너무 길어서 마지막 줄이 짤림
- 해결: Blog 같은 경우 `-webkit-line-clamp` 로 줄 수 제한 + line-height 1.5+ + padding-bottom 2px (이미 적용됨)
- 새로 카드 추가하면서 잘림이 생기면 — 비슷한 패턴으로 line-clamp + line-height 적용

### C2. 페이지 전환 애니메이션이 안 나옴
- 원인: 브라우저가 `prefers-reduced-motion` 설정으로 애니메이션 꺼져 있음 (macOS: 시스템 설정 → 손쉬운 사용 → 동작 줄이기)
- 일부러 그런 거라면 OK. 강제로 다시 켜고 싶으면 OS 설정 해제

### C3. 모바일에서 깨짐
- 원인: 모바일 미디어쿼리 (`@media (max-width: 640px)`) 가 부족
- 해결: `src/styles/layout.css` 의 `@media (max-width: 640px)` 블록 보고 비슷한 패턴 추가 (agent 에게 부탁 권장)

### C4. 한국어 줄바꿈이 어색함
- 원인: 영어용 word-break 규칙
- 해결: 해당 element CSS 에 `word-break: keep-all` 추가 (이미 Blog featured 에 적용됨)

---

## D. Git / GitHub

### D1. commit author 가 잘못 찍힘 (예: 옛날 계정)
```bash
cd ~/local-dev/hompage
git config user.name "codwithyc"
git config user.email "su253294@gmail.com"
```
이미 등록된 commit 들을 바꾸려면 `git filter-branch` 또는 `git rebase --root -i` 가 필요한데 강력히 권장 안 함 (history 재작성, force-push 필요). 새 commit 부터만 맞으면 OK.

### D2. PR 머지 후 로컬 main 이 안 따라옴
```bash
git checkout main
git pull origin main
```

### D3. 실수로 main 에 직접 push 했는데 되돌리고 싶음
```bash
git log --oneline                  # 돌아갈 commit hash 확인
git revert <commit-hash>           # 안전 (새 취소 commit)
git push origin main
```
**절대 하지 말 것**: `git push --force` to main — 다른 사람의 작업까지 날아갈 수 있음

---

## E. 새 문제 만나면

1. 여기에 한 줄 추가 (`### EN. ...` 형식)
2. 가능하면 원인 + 해결 + 재현 조건 같이
3. agent 에게 부탁할 땐 이 문서 링크 + 만난 에러 메시지 같이

---

연결: [[hompage]] · [[architecture]] · [[how-to-edit]] · [[data-sources]] · [[deploy]]
