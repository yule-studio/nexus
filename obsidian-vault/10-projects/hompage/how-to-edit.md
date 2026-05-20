---
title: "hompage how-to-edit — 가이드 색인"
kind: project-doc
project: hompage
status: shipped
created_at: 2026-05-20T10:20:00+09:00
tags: [project, hompage, frontend, edit-guide, index]
home_hub: hompage
related:
  - "[[hompage]]"
  - "[[architecture]]"
  - "[[deploy]]"
  - "[[troubleshooting]]"
---

# hompage 직접 수정 가이드 — 색인

**[[hompage|↑ hompage hub]]**

> FE 무지 사용자가 보고 자가 수정. 작업 종류 → 자세한 가이드 문서로 연결.

---

## 0. 시작하기 전

### 0.1 환경 준비 (1번만)
```bash
cd ~/local-dev/hompage
npm install           # 5분
```

### 0.2 매 작업 흐름
```bash
git pull origin main          # 1) 최신 상태 받기
# 2) 코드 수정 (아래 가이드 참고)
npm run dev                   # 3) http://localhost:5173 미리 보기
# 4) 잘 됐으면
git add .
git commit -m "✏️ 변경 내용"
git push origin main          # 5) 5분 후 사이트 갱신
```

### 0.3 망쳤을 때
```bash
git checkout .                # 모든 변경 취소 (미커밋)
```
또는:
```bash
git log --oneline
git revert <bad-commit-hash>
git push origin main
```

---

## 1. 자주 하는 작업 — 가이드 문서

### Hero / 자기 소개
- **이름·역할·위치·SNS 변경** → [[guides/profile|guides/profile]]
- **Hero CTA 버튼 추가/제거/스타일** → [[guides/hero-ctas|guides/hero-ctas]]

### 데이터 (자격증 / 수상 / Homelab / Calendar)
- **자격증·수상·홈랩·이벤트 목록 편집** → [[guides/lists-data|guides/lists-data]]

### 프로젝트
- **새 프로젝트 등장 / LIVE 배지 설정 / 필터 / 정렬** → [[guides/projects|guides/projects]]

### Skills
- **언어 비율 / 토픽 카테고리 / 자기평가 fallback** → [[guides/skills|guides/skills]]

---

## 2. 스타일 (디자인) 변경 — styling 문서

### 색·폰트·여백·반지름 정의
- **모든 토큰 (CSS 변수) 사전** → [[styling/tokens|styling/tokens]]

### 색
- **색상 변경 레시피 (액센트 / 배경 / 카드 / status)** → [[styling/colors|styling/colors]]

### 폰트 / 텍스트
- **폰트 패밀리 / 크기 / line-height / 한국어 줄바꿈** → [[styling/typography|styling/typography]]

### 카드
- **Card 컴포넌트 해부 / span 변형 / scoped 클래스** → [[styling/cards|styling/cards]]

### 레이아웃
- **Page → Container → Grid 4단 위계 / 12-col grid** → [[styling/layout-grid|styling/layout-grid]]

### 애니메이션
- **페이지 전환 stagger / hover 효과** → [[styling/animations|styling/animations]]

### 반응형
- **모바일 / 태블릿 / 데스크탑 break-point** → [[styling/responsive|styling/responsive]]

---

## 3. 페이지별 해부 — pages 문서

- **Home dashboard 10 카드 해부** → [[pages/home|pages/home]]
- (Projects / Skills / 기타 페이지 anatomy 는 추후 추가 예정 — 우선 [[guides/projects]] / [[guides/skills]] 참고)

---

## 4. 데이터 / 배포

### 데이터 소스
- **각 화면이 어디서 데이터를 가져오는지** → [[data-sources]]

### 배포
- **push → 사이트 갱신 timeline / 실시간 트리거 / 시크릿** → [[deploy]]

### 트러블슈팅
- **자주 만나는 문제 19가지** → [[troubleshooting]]

---

## 5. 작업 흐름 별 권장 문서 조합

### "사이트 디자인 톤을 바꾸고 싶다"
1. [[styling/tokens|tokens]] — 어떤 토큰이 있는지 큰 그림
2. [[styling/colors|colors]] — 액센트 색 한 줄 바꿔보기
3. (선택) [[styling/typography|typography]] — 폰트 / 크기 조정

### "내 정보를 다 업데이트"
1. [[guides/profile|profile]] — 이름 / 역할 / SNS
2. [[guides/lists-data|lists-data]] — 자격증 / 수상
3. [[guides/projects|projects]] — repo About → Website 로 LIVE 표시

### "Home 화면에 새 카드 추가"
1. [[pages/home|pages/home]] §5 "카드 추가하기"
2. [[styling/cards|cards]] — 카드 anatomy
3. [[styling/layout-grid|layout-grid]] — span / grid

### "글자 / 사진 잘림 해결"
1. [[styling/typography|typography]] §6 한국어 descender 잘림
2. [[styling/cards|cards]] §10 잘림 방지 패턴

### "실시간 갱신 셋업"
1. [[deploy]] §2.4 "실시간 트리거"
2. [[deploy]] §5 "시크릿"

---

## 6. 절대 만지지 말 것

| 파일 | 왜 |
| --- | --- |
| `node_modules/` | 외부 도구 캐시 |
| `dist/` | 빌드 결과물 (자동 생성) |
| `package-lock.json` | 의존성 자물쇠 |
| `public/*.json` | GitHub Actions 가 덮어씀 |
| `vite.config.ts` / `tsconfig.json` | 빌드 설정 |
| `.github/workflows/*.yml` | CI 깨질 수 있음 (deploy 외 단순 환경변수만 OK) |
| `scripts/*.mjs` | Node.js + GitHub API. agent 부탁 권장 |
| `src/App.tsx` | 라우터 (새 페이지 추가 시만, 그 외 손대지 마세요) |

이런 파일 손대고 싶을 땐 agent 에게 부탁 — 변경 의도만 말하면 PR 까지 만들어 줍니다.

---

## 7. 검증 명령 (커밋 전)

```bash
npm run typecheck    # TS 타입 검사 (콤마 / 따옴표 syntax 잡힘)
npm run lint         # ESLint 검사
npm run build        # 실제 빌드 시도
```

빨간 에러 없이 다 통과해야 push.

---

연결: [[hompage]] · [[architecture]] · [[data-sources]] · [[deploy]] · [[troubleshooting]]
