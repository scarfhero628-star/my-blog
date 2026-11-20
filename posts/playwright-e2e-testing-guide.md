---
title: Playwright E2E 테스트 완전 가이드 — 브라우저 자동화로 실제 사용자 경험을 검증하는 법
date: 2026-11-20
tags: [Playwright, E2E 테스트, 테스팅, 자동화, 웹 개발]
description: Playwright로 End-to-End 테스트를 작성해 실제 브라우저 환경에서 애플리케이션의 핵심 흐름을 자동으로 검증하는 방법을 소개합니다.
---

단위 테스트는 함수 하나가 올바르게 동작하는지 검증하지만, 사용자가 실제로 겪는 버그는 여러 컴포넌트가 맞물리는 순간에 터지는 경우가 많습니다. **E2E(End-to-End) 테스트**는 실제 브라우저를 띄우고 사용자 행동을 그대로 재현해 전체 흐름을 검증합니다. 그 도구로 가장 주목받는 것이 Microsoft가 오픈소스로 공개한 **Playwright**입니다.

이 글에서는 Playwright의 핵심 개념부터 실전 패턴까지 단계별로 살펴봅니다.

---

## E2E 테스트란 무엇인가?

테스트 피라미드를 떠올려 보면 맨 아래에 **단위 테스트**, 중간에 **통합 테스트**, 맨 위에 **E2E 테스트**가 있습니다.

- **단위 테스트**: 개별 함수·컴포넌트의 동작 검증. 빠르고 비용이 낮다.
- **통합 테스트**: 여러 모듈이 연결됐을 때의 동작 검증.
- **E2E 테스트**: 실제 브라우저에서 전체 애플리케이션 흐름 검증. 느리지만 사용자 관점과 가장 가깝다.

E2E 테스트는 개수를 줄이고 **핵심 사용자 여정(critical user journey)**에만 집중하는 것이 좋습니다. 로그인 → 상품 검색 → 결제처럼 비즈니스에 치명적인 흐름이 그 대상입니다.

---

## Playwright를 선택하는 이유

기존 E2E 도구로는 Cypress와 Selenium이 있습니다. Playwright가 최근 급부상한 이유는 다음과 같습니다.

| 특징 | Playwright | Cypress |
|------|-----------|---------|
| 브라우저 지원 | Chromium, Firefox, WebKit | Chromium 계열만 (Firefox 베타) |
| 병렬 실행 | 기본 지원 | 유료 플랜 필요 |
| 여러 탭/팝업 | 지원 | 제한적 |
| API 모킹 | 내장 `route()` | 플러그인 필요 |
| 언어 지원 | JS/TS, Python, Java, C# | JS/TS만 |

특히 Playwright는 **자동 대기(Auto-wait)** 기능 덕분에 `sleep()`이나 임의의 타임아웃 없이도 안정적인 테스트를 작성할 수 있습니다.

---

## 설치와 기본 설정

```bash
npm init playwright@latest
```

설치 과정에서 브라우저, TypeScript 여부, CI 설정을 물어봅니다. 완료하면 `playwright.config.ts`가 생성됩니다.

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 13'] } },
  ],
});
```

`baseURL`을 설정해 두면 테스트 코드에서 상대 경로(`/login`)만 써도 됩니다.

---

## 첫 번째 테스트 작성

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test('로그인 성공 흐름', async ({ page }) => {
  await page.goto('/login');

  await page.getByLabel('이메일').fill('user@example.com');
  await page.getByLabel('비밀번호').fill('password123');
  await page.getByRole('button', { name: '로그인' }).click();

  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByText('안녕하세요')).toBeVisible();
});

test('잘못된 비밀번호 오류 메시지', async ({ page }) => {
  await page.goto('/login');

  await page.getByLabel('이메일').fill('user@example.com');
  await page.getByLabel('비밀번호').fill('wrongpassword');
  await page.getByRole('button', { name: '로그인' }).click();

  await expect(page.getByRole('alert')).toContainText('이메일 또는 비밀번호가 올바르지 않습니다');
});
```

### 로케이터 우선순위

Playwright는 다양한 방식으로 요소를 찾을 수 있습니다. 접근성이 좋은 순서대로 사용하는 것을 권장합니다.

```typescript
// 권장 순서
page.getByRole('button', { name: '제출' })   // ARIA role + 텍스트
page.getByLabel('이메일')                     // label과 연결된 input
page.getByPlaceholder('이메일을 입력하세요')  // placeholder
page.getByText('안녕하세요')                  // 텍스트 내용
page.getByTestId('submit-button')             // data-testid 속성

// 비권장
page.locator('.btn-primary')                  // CSS 클래스 (변경에 취약)
page.locator('#submit')                       // id (맥락에 따라 변할 수 있음)
```

---

## Page Object Model(POM) 패턴

테스트가 많아지면 같은 셀렉터가 여러 파일에 흩어집니다. **Page Object Model**은 각 페이지의 UI를 클래스로 캡슐화해 중복을 없애고 유지보수를 쉽게 만듭니다.

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorAlert: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('이메일');
    this.passwordInput = page.getByLabel('비밀번호');
    this.submitButton = page.getByRole('button', { name: '로그인' });
    this.errorAlert = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';

test('로그인 성공 흐름', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password123');

  await expect(page).toHaveURL('/dashboard');
});
```

---

## API 모킹으로 테스트 안정성 높이기

실제 API 서버에 의존하는 테스트는 네트워크 상태나 서버 데이터에 따라 결과가 달라집니다. `page.route()`로 네트워크 요청을 가로채 모킹할 수 있습니다.

```typescript
test('상품 목록 렌더링', async ({ page }) => {
  // API 응답을 모킹
  await page.route('**/api/products', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: '노트북', price: 1500000 },
        { id: 2, name: '마우스', price: 50000 },
      ]),
    });
  });

  await page.goto('/products');

  await expect(page.getByText('노트북')).toBeVisible();
  await expect(page.getByText('1,500,000원')).toBeVisible();
});

test('API 오류 시 에러 화면 표시', async ({ page }) => {
  await page.route('**/api/products', (route) =>
    route.fulfill({ status: 500 })
  );

  await page.goto('/products');

  await expect(page.getByText('데이터를 불러오지 못했습니다')).toBeVisible();
});
```

---

## 인증 상태 재사용 (storageState)

로그인 후 보호된 페이지를 테스트할 때마다 로그인 과정을 반복하면 시간이 낭비됩니다. Playwright의 `storageState`로 인증 상태를 저장하고 재사용할 수 있습니다.

```typescript
// e2e/auth.setup.ts
import { test as setup } from '@playwright/test';

setup('인증 상태 저장', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('이메일').fill('user@example.com');
  await page.getByLabel('비밀번호').fill('password123');
  await page.getByRole('button', { name: '로그인' }).click();
  await page.waitForURL('/dashboard');

  // 쿠키와 localStorage를 파일로 저장
  await page.context().storageState({ path: 'e2e/.auth/user.json' });
});
```

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: /auth\.setup\.ts/,
    },
    {
      name: 'authenticated',
      dependencies: ['setup'],
      use: {
        storageState: 'e2e/.auth/user.json', // 저장된 인증 상태 사용
      },
    },
  ],
});
```

---

## 디버깅 도구

테스트가 실패했을 때 원인을 파악하는 도구가 풍부합니다.

```bash
# UI 모드: 시각적으로 테스트 실행·디버깅
npx playwright test --ui

# 헤드리스 OFF + 슬로우 모션: 눈으로 확인
npx playwright test --headed --slow-mo=500

# Playwright Inspector: 브레이크포인트
PWDEBUG=1 npx playwright test

# 특정 테스트만 실행
npx playwright test e2e/login.spec.ts
```

실패한 테스트는 **trace 파일**이 자동으로 저장됩니다.

```bash
npx playwright show-trace test-results/trace.zip
```

Trace Viewer에서 각 단계의 스크린샷, 네트워크 요청, 콘솔 로그를 타임라인으로 확인할 수 있습니다.

---

## CI/CD 연동

GitHub Actions 예시입니다.

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run E2E tests
        run: npx playwright test

      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

`if: always()`로 테스트가 실패해도 리포트가 업로드되도록 합니다.

---

## 실전 팁

### 1. 테스트 격리를 철저히

각 테스트는 서로 독립적이어야 합니다. 테스트 간 데이터를 공유하면 순서에 따라 결과가 달라집니다. `beforeEach`에서 상태를 초기화하세요.

### 2. `data-testid`는 최후의 수단

`getByRole`이나 `getByLabel`로 선택할 수 없을 때만 `data-testid`를 사용하세요. 의미 있는 셀렉터가 접근성도 함께 개선합니다.

### 3. 너무 많은 E2E 테스트를 작성하지 않기

E2E 테스트는 느립니다. 핵심 흐름 10~20개 정도를 집중적으로 커버하고, 세부 로직은 단위·통합 테스트에 맡기세요.

### 4. Flaky 테스트 즉시 수정

간헐적으로 실패하는 테스트는 팀 전체의 신뢰를 갉아먹습니다. `retries` 설정으로 CI에서 재시도하되, 근본 원인을 반드시 파악하세요.

---

## 마치며

Playwright는 단순한 브라우저 자동화 도구를 넘어 **현대 웹 테스팅의 표준**에 가깝게 성장했습니다. 자동 대기, 강력한 셀렉터, 풍부한 디버깅 도구 덕분에 안정적인 테스트를 비교적 쉽게 작성할 수 있습니다.

단위 테스트가 "코드가 맞게 작동하는가"를 검증한다면, E2E 테스트는 "사용자가 원하는 대로 작동하는가"를 검증합니다. 두 가지를 균형 있게 갖추면 배포할 때마다 "혹시 뭔가 깨진 건 아닐까?" 하는 불안감에서 해방될 수 있습니다.
