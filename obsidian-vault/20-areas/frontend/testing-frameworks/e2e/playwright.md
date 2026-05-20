---
title: "Playwright — E2E 테스트"
kind: knowledge
project: frontend
status: current
tags: [frontend, testing, playwright, e2e]
---

# Playwright

**[[../testing-frameworks|↑ Testing hub]]**

> 실제 브라우저 (Chromium / Firefox / Webkit) 에서 자동화. Microsoft 제작.

---

## 1. 셋업

```bash
npm init playwright@latest
```

대화형 옵션:
- TypeScript? → Yes
- 폴더 이름? → tests (default)
- GitHub Actions? → Yes (CI 자동 생성)
- Browsers 다운로드? → Yes

폴더 구조:
```
tests/
├─ example.spec.ts
└─ ...
playwright.config.ts
```

---

## 2. 첫 테스트

`tests/login.spec.ts`:
```typescript
import { test, expect } from "@playwright/test";

test("로그인 성공", async ({ page }) => {
  await page.goto("http://localhost:3000/login");

  await page.getByLabel("이메일").fill("test@example.com");
  await page.getByLabel("비밀번호").fill("12345678");
  await page.getByRole("button", { name: "로그인" }).click();

  await expect(page).toHaveURL(/.*dashboard/);
  await expect(page.getByText("환영합니다")).toBeVisible();
});
```

```bash
npx playwright test
npx playwright test --ui        # GUI 모드
npx playwright test --headed    # 브라우저 보이게
npx playwright test --debug     # 디버그
```

---

## 3. 핵심 — page

```typescript
await page.goto("/");
await page.goto("https://example.com");

// element 찾기
const button = page.getByRole("button", { name: "제출" });
const input = page.getByLabel("이메일");
const link = page.getByText("자세히");
const elem = page.locator(".my-class");
const byTestId = page.getByTestId("submit");
```

### 액션
```typescript
await button.click();
await input.fill("hello");
await input.type("hello", { delay: 100 });    // 천천히
await page.keyboard.press("Enter");
await link.hover();
await page.screenshot({ path: "shot.png" });
```

### 검증 (assertions)
```typescript
await expect(page).toHaveURL("/dashboard");
await expect(page).toHaveTitle(/Dashboard/);

await expect(elem).toBeVisible();
await expect(elem).toHaveText("Hello");
await expect(elem).toHaveAttribute("href", "/");
await expect(elem).toHaveCount(3);
await expect(input).toHaveValue("hi");
```

---

## 4. RTL 과 같은 쿼리 우선순위

```typescript
// 1) Role / Label (권장)
page.getByRole("button", { name: "..." });
page.getByLabel("이메일");
page.getByRole("textbox", { name: "검색" });

// 2) Text
page.getByText("Submit");

// 3) Placeholder / alt
page.getByPlaceholder("이메일");
page.getByAltText("로고");

// 4) TestId (최후)
page.getByTestId("submit-btn");

// 5) CSS / XPath (가능하면 피하기)
page.locator(".btn-primary");
page.locator("xpath=//button");
```

---

## 5. 자동 대기 (auto-waiting)

```typescript
await button.click();   // 보이고 클릭 가능할 때까지 자동 대기
```

기본 timeout 30s. 명시:
```typescript
await elem.click({ timeout: 10000 });
await expect(elem).toBeVisible({ timeout: 5000 });
```

---

## 6. 네트워크 mock

```typescript
test("API mock", async ({ page }) => {
  await page.route("**/api/users", (route) =>
    route.fulfill({
      json: [{ name: "테스트" }],
    })
  );

  await page.goto("/users");
  await expect(page.getByText("테스트")).toBeVisible();
});
```

### 응답 가로채기
```typescript
await page.route("**/api/login", async (route) => {
  const request = route.request();
  console.log(request.postData());

  await route.fulfill({ status: 401, json: { error: "Unauthorized" } });
});
```

---

## 7. 인증 (storageState)

매 테스트마다 로그인하면 느림. 한 번만:

### setup 파일
`tests/setup/auth.setup.ts`:
```typescript
import { test as setup, expect } from "@playwright/test";

setup("authenticate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("이메일").fill("test@example.com");
  await page.getByLabel("비밀번호").fill("password");
  await page.getByRole("button", { name: "로그인" }).click();

  await expect(page).toHaveURL("/dashboard");

  // 저장
  await page.context().storageState({ path: "playwright/.auth/user.json" });
});
```

### 설정
`playwright.config.ts`:
```typescript
projects: [
  { name: "setup", testMatch: /.*\.setup\.ts/ },
  {
    name: "logged-in",
    use: { storageState: "playwright/.auth/user.json" },
    dependencies: ["setup"],
  },
],
```

→ 모든 logged-in 테스트가 자동으로 로그인 상태로 시작.

---

## 8. fixtures (재사용 가능 setup)

```typescript
import { test as base } from "@playwright/test";

type MyFixtures = {
  authedPage: Page;
};

export const test = base.extend<MyFixtures>({
  authedPage: async ({ page }, use) => {
    await page.goto("/login");
    await page.getByLabel("이메일").fill("test@example.com");
    // ...
    await use(page);
  },
});

// 사용
test("dashboard", async ({ authedPage }) => {
  await authedPage.goto("/dashboard");
  // 이미 로그인 됨
});
```

---

## 9. 병렬 실행

기본은 파일 별 병렬. test.describe.serial 로 순차:
```typescript
test.describe.serial("순서 중요", () => {
  test("1번", async ({ page }) => {});
  test("2번", async ({ page }) => {});
});
```

### worker 수 조정
`playwright.config.ts`:
```typescript
export default defineConfig({
  workers: process.env.CI ? 2 : undefined,   // CI 에선 2, 로컬은 자동
});
```

---

## 10. 다른 브라우저

`playwright.config.ts`:
```typescript
projects: [
  { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  { name: "firefox", use: { ...devices["Desktop Firefox"] } },
  { name: "webkit", use: { ...devices["Desktop Safari"] } },
  { name: "mobile-chrome", use: { ...devices["Pixel 5"] } },
  { name: "mobile-safari", use: { ...devices["iPhone 12"] } },
],
```

모든 브라우저 / 디바이스에서 자동 실행.

---

## 11. screenshot / video

```typescript
// screenshot 자동
use: {
  screenshot: "only-on-failure",   // 실패 시만
  video: "retain-on-failure",
  trace: "on-first-retry",
}
```

`playwright-report/` 에 저장.

---

## 12. trace viewer

```bash
npx playwright show-trace trace.zip
```

각 step 의 스크린샷 / DOM 스냅샷 / 네트워크 모두 시각.

---

## 13. 디버깅

```bash
npx playwright test --debug
```

브라우저 + Playwright Inspector 같이 열림. step-by-step.

VSCode 확장:
- Playwright Test for VSCode → 좌측 사이드바에서 테스트 / 디버그

---

## 14. CI (GitHub Actions)

```yaml
# .github/workflows/playwright.yml
name: Playwright Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run build
      - run: npm run start &
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

---

## 15. 자주 만드는 패턴

### A. 로그인 → CRUD
```typescript
test("게시글 작성 → 수정 → 삭제", async ({ authedPage }) => {
  await authedPage.goto("/posts/new");
  await authedPage.getByLabel("제목").fill("테스트");
  await authedPage.getByRole("button", { name: "저장" }).click();

  await expect(authedPage).toHaveURL(/.*posts\/\d+/);

  // 수정
  await authedPage.getByRole("button", { name: "수정" }).click();
  await authedPage.getByLabel("제목").fill("수정됨");
  await authedPage.getByRole("button", { name: "저장" }).click();

  await expect(authedPage.getByText("수정됨")).toBeVisible();

  // 삭제
  authedPage.on("dialog", (dialog) => dialog.accept());
  await authedPage.getByRole("button", { name: "삭제" }).click();
  await expect(authedPage).toHaveURL(/.*posts$/);
});
```

### B. 검색 / 필터
```typescript
test("검색", async ({ page }) => {
  await page.goto("/products");
  await page.getByPlaceholder("검색").fill("아이폰");
  await page.keyboard.press("Enter");

  const results = page.getByRole("article");
  await expect(results).toHaveCount(5);
});
```

### C. Visual diff (시각 회귀)
```typescript
test("페이지 캡처", async ({ page }) => {
  await page.goto("/");
  await expect(page).toHaveScreenshot("home.png");
});
```

처음엔 baseline 생성, 다음부턴 비교.

---

## 16. 함정

| 증상 | 원인 |
| --- | --- |
| 테스트가 flaky (가끔 실패) | auto-wait 빠뜨림 / race condition |
| `page.evaluate` 안에서 outer 변수 안 됨 | sandbox. 인자로 전달 |
| dev 서버가 안 떠있어 fail | `webServer` config 추가 |
| 너무 느림 | `--workers=4` / fixture 재사용 |
| 인증 매번 다시 | storageState 사용 |

`webServer` 자동 시작:
```typescript
webServer: {
  command: "npm run dev",
  port: 3000,
  reuseExistingServer: !process.env.CI,
},
```

---

## 17. 다음 단계

1. **[[../unit/vitest|Vitest]]** — Unit 테스트
2. **[[../unit/mocking|Mocking]]** — MSW 와 함께

---

연결: [[../testing-frameworks|↑ hub]] · [[../unit/vitest]]
