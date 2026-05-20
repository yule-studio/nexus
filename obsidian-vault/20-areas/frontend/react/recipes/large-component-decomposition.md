---
title: "Large Component Decomposition — 600~2000줄 컴포넌트 시니어 분리 패턴"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, refactoring, project-structure, recipes]
related:
  - "[[project-structure]]"
  - "[[role-based-layout]]"
created_at: 2026-05-18
---

# Large Component Decomposition — 600~2000줄 컴포넌트 시니어 분리 패턴

**[[recipes|↑ Recipes hub]]**

> 운영 코드에서 한 화면 = 한 파일 = 1000+ 줄 이 종종 보인다. "동작은 한다" 라는 이유로 두면, 새 기능 추가 / 회귀 추적 / 리뷰 비용이 폭발한다. 시니어가 어떤 경계로 잘라내는지 — 600~2000줄 화면을 7~10개 모듈로 쪼개는 reproducible 한 패턴.

## 1. 왜 분리해야 하나

### 왜 필요
- 한 파일 1000줄 = 깃 diff 가 50줄 변경에 950줄 컨텍스트. 리뷰어가 "어디가 바뀐 거지" 헷갈림.
- useState 8개 + filter / sort / localStorage CRUD + 폼 / 모달 / 테이블 / 차트 등이 한 파일에 섞이면 단위 테스트 불가능.
- 디자인 토큰 (색상 / 그림자) 이 인라인 헥사로 박혀 같은 색을 3-5번 중복 정의 — 디자인 시스템 update 시 누락 다발.

### 안 하면 문제
- 새 사람이 이 화면에 들어와서 "어디부터 읽어야 하지" 30분 헤매다 포기.
- bug fix 가 다른 영역에 누수 (한 useEffect 가 5개 책임 처리).
- 디자인 토큰 update 가 한 곳 놓치면 색이 섞임 (예: 어드민 파랑 + 학생 초록 한 페이지).

### 대안
- **현상 유지** — 사소한 fix 만 하고 큰 refactor 미루기. 일정 단기 OK, 장기 부채 폭발.
- **전면 rewrite** — 디자인 그대로 두고 src 만 새로 작성. 시간 많이 들고 회귀 위험.
- **점진 추출** (본 패턴) — 한 PR 안에서 commit 단위 (data / hook / sub-component) 로 6-8 sub 단계 분리. 디자인 픽셀 동일 유지하며 코드만 정리.

### 트레이드오프
- 본 패턴: 한 화면이 7-10개 파일이 됨 — 파일 갯수 ↑. 하지만 각 파일이 50-150줄 = 한눈에 들어옴. import 한 줄 더 보다는 한 화면 1000줄이 훨씬 비용 큼.

## 2. 분리 신호 — 언제 자르는가

다음 중 **3개 이상** 만나면 분리 후보 (단, 길이만으로 판단 X):

- useState **5개 이상** + 그 중 절반 이상이 같은 데이터 (예: form draft 의 4 필드)
- 같은 component 안에 fetch / filter / sort / render / 모달 / 폼 / 차트 / 테이블 중 **4가지 이상**
- 같은 헥사 색 / 같은 className 패턴이 **3번 이상 중복**
- 한 함수가 다른 도메인의 dataclass 를 직접 mutate
- "// 임시" / "// TODO" / "// FIXME" 가 한 파일에 **5개 이상**

길이 가이드 (`yule-studio-agent` 전역 규칙 상속):
- **700줄 초과** → warning, "왜 한 파일에 남겨야 하는지" PR 본문에 적지 않으면 검토자가 분리 요청
- **1000줄 초과** → 기본 분리 대상
- **1000줄 + 책임 2개 이상** → 분리 필수

## 3. 시니어 분리 청사진 (단일 화면 기준)

```
src/pages/<role>/_pages/<feature>/
├── <Feature>.tsx                  # shell — 모달 open 상태 + sub-component 조립 (얇음, 30-80줄)
├── <feature>-data.ts              # 타입 + mock + 디자인 상수 (STATUS_STYLE 등)
├── <feature>-utils.ts             # 순수 함수 (generateCode 등)
├── use<Feature>.ts                # 상태 + filter / sort / CRUD / 통계 hook
├── <Feature>Header.tsx            # 헤더 (제목 + CTA 버튼)
├── <Feature>Stats.tsx             # KPI 카드 그리드
├── <Feature>Filters.tsx           # 검색 / 필터 / 정렬
├── <Feature>Table.tsx             # 테이블 + 행 + empty state
├── <Feature>AddModal.tsx          # 폼 모달
└── <Feature>AddSuccessModal.tsx   # 결과 모달
```

각 파일 책임 1개 + 100~200줄 = 한눈에. shell 은 30-80줄 = 전체 구성을 한 화면에 본다.

## 4. 분리 순서 (commit 단위)

큰 컴포넌트를 한 PR 안에 분리할 때:

```
C1. data + utils 모듈 (의존성 없음, 가장 안전)
C2. hook (data 만 의존)
C3. presentational sub-component 들 (hook 사용)
C4. 큰 모달 / 큰 sub-section (별도 commit)
C5. shell + router 등록
C6. CHANGELOG + vault 노트
```

각 commit 이 빌드 가능 (typecheck 통과) — bisect 가능. 마지막 shell commit 까지는 원본 컴포넌트가 그대로 살아 있어도 OK (import 만 안 하면 됨).

## 5. data module 분리 패턴

**Before** (한 파일 안):
```tsx
const STATUS_STYLE: Record<string, any> = { ... }
const PLAN_FEES: Record<string, number> = { ... }
const INIT_DATA = [ ... ]
// + Component
```

**After** (`<feature>-data.ts`):
```ts
export type Status = 'active' | 'trial' | 'unpaid'

export const STATUS_STYLE: Record<Status, { color: string; bg: string; label: string }> = { ... }
export const INIT_DATA: Item[] = [ ... ]
```

- `any` → 명시적 타입 (시니어 1번 룰).
- `Record<Status, T>` 로 status 값 추가 시 컴파일 에러 = 회귀 방지.
- mock data 분리는 후속 React Query / Supabase 이식 시 한 줄만 갈아끼면 됨.

## 6. hook 분리 패턴

**Before** (컴포넌트 안에):
```tsx
const [items, setItems] = useState(...)
const [search, setSearch] = useState('')
const [filter, setFilter] = useState('all')
// + 30줄 filter + sort 로직
// + useEffect localStorage
```

**After** (`use<Feature>.ts`):
```ts
export function useFeature() {
  const [items, setItems] = useState(...)
  // ...

  const filtered = useMemo(() => { ... }, [items, search, filter])
  const stats = useMemo(() => { ... }, [items])

  const add = (draft) => { ... }

  return { items, filtered, stats, search, setSearch, filter, setFilter, add }
}
```

- `useMemo` 로 매 render 재계산 막음.
- CRUD action 은 hook 안 — 컴포넌트는 호출만.
- 반환 객체는 group 별로 (data / 상태 / action). 호출부에서 destructure 깔끔.

## 7. presentational component 분리 패턴

같은 UI 패턴 (KPI 카드 / 필터 토글 / 코드 복사 행 등) 이 2-3번 중복되면 → sub-component 로 추출 + props 로 variant 받기.

```tsx
// Before: 4개 카드 인라인 (180줄)
<div className="card-main">...</div>
<div className="card-active">...</div>
<div className="card-trial">...</div>
<div className="card-unpaid">...</div>

// After: 그리드 + 카드 컴포넌트
<StatsGrid>
  <MainStatCard total={stats.total} />
  <SimpleStatCard label="활성" value={stats.active} color="green" />
  <SimpleStatCard label="체험" value={stats.trial} color="blue" />
  <UnpaidStatCard count={stats.unpaid} />
</StatsGrid>
```

variant 가 3 이상이면 `scheme` prop 으로 분기 (`'accent' | 'neutral' | 'warning'`).

## 8. 모달 분리 패턴

모달 1개당 1파일. 내부도 sub-component 로:

```tsx
// AddModal.tsx
export function AddModal({ open, onClose, onSubmit }) {
  const [draft, setDraft] = useState(EMPTY_DRAFT)
  if (!open) return null

  return (
    <Backdrop onClick={onClose}>
      <ModalShell>
        <ModalHeader onClose={onClose} />
        <Body>
          <FieldGroup>...</FieldGroup>
          <PlanPicker .../>
          <OptionsBox .../>
        </Body>
        <ModalFooter onCancel={onClose} onSubmit={handleSubmit} />
      </ModalShell>
    </Backdrop>
  )
}
```

- 모달이 자체 form state 소유 — 부모는 onSubmit (draft) 만 받음.
- AntD `Modal` 사용해도 같은 분리 원칙 적용.

## 9. shell 의 책임

shell 은 다음만:
- sub-component 조립 (위→아래 시각 순서)
- 모달 open/close state (UI shell 책임)
- hook 호출 + sub-component 에 prop 분배

shell 안에 비즈니스 로직 / 필터 계산 / 데이터 변환 — 즉시 hook 으로 빼야 하는 신호.

```tsx
export function MasterAcademies() {
  const academies = useAcademies()      // 상태 / 통계 / CRUD
  const [addOpen, setAddOpen] = useState(false)
  const [success, setSuccess] = useState(null)

  return (
    <div className="px-8 py-7">
      <AcademiesHeader onAddClick={() => setAddOpen(true)} />
      <AcademiesStats stats={academies.stats} />
      <AcademiesFilters {...academies} />
      <ResultsSummary {...academies} />
      <AcademiesTable rows={academies.filtered} />
      <AcademyAddModal open={addOpen} onClose={...} onSubmit={...} />
      <AcademyAddSuccessModal open={!!success} academy={success} onClose={...} />
    </div>
  )
}
```

50-80줄. 화면 전체 구조가 한눈에. 새 사람이 들어와서 30초 안에 파악.

## 10. 함정

1. **분리하면서 디자인 바꾸기** — 절대 금지. 분리 = 코드 구조만, 시각 결과 동일.
2. **모달 state 를 shell 에 두기** — form draft 를 shell 에 두면 모달 close 시 reset 로직이 shell 에 흘러들어옴. 모달 자체에.
3. **shell 안에 useMemo 폭증** — 그건 hook 으로.
4. **sub-component 의 props 가 10개 이상** — sub-component 가 너무 많은 책임. 더 분리.
5. **same variant 3번 미만에 컴포넌트 만들기** — overengineering. 3번 이상 중복 발견 후 추출.
6. **data module 에 컴포넌트 import** — 순환 의존. data 는 순수 module (외부 의존 X).
7. **utils 가 비순수** — `Date.now()` / `Math.random()` / fetch — utils 에 들어가면 테스트 깨짐. 그건 hook 안에.

## 11. 단계별 측정 (분리 결과 검증)

| 측정 | Before | After (목표) |
| ---- | ------ | ----------- |
| shell 줄 수 | 600-2000 | 30-80 |
| 가장 큰 sub-component | (전체) | 100-200 |
| useState 갯수 in shell | 8-15 | 0-3 (모달 open 등 UI shell 만) |
| 같은 헥사 색 중복 | 3-5번 | 0 (theme 토큰 통일) |
| 같은 className 패턴 중복 | 3-5번 | 0 (sub-component 추출) |

## 12. 관련

- [[project-structure]] — 디렉토리 / 책임 분리 기준
- [[role-based-layout]] — role 별 Layout 패턴 (본 패턴의 한 응용)
- [[react-query]] — mock data → real fetch 교체 시
