## What & When

**WebdriverIO (WDIO)** is a **Node.js** test automation framework built on the **WebDriver** protocol. It runs **cross-browser** E2E tests (Chrome, Firefox, Safari via drivers) with a Mocha/Jasmine/Cucumber runner, Page Object pattern, and CI integrations.

Use WebdriverIO when:

- You need a **structured E2E test suite** for a React/Next app
- Tests must run on **multiple browsers** in CI (Selenium Grid, BrowserStack, Sauce)
- Your team prefers **WebDriver** over CDP-only tools

Use [[Codes/JavaScript — Puppeteer]] for quick Node scrape scripts; [[Browser Automation — Playwright]] for Python E2E.

```bash
npm create wdio@latest .
```

Hub: [[JavaScript Development]]

---

## Project Scaffold

```bash
npm create wdio@latest .
# choose: standalone, Mocha, spec files, chromedriver
```

Typical layout:

```text
test/
  specs/
    login.e2e.ts
  pageobjects/
    login.page.ts
wdio.conf.ts
```

---

## Basic Spec (Mocha)

```typescript
// test/specs/home.e2e.ts
describe("Home page", () => {
  it("shows welcome heading", async () => {
    await browser.url("/");
    const heading = await $("h1");
    await expect(heading).toHaveText("Welcome");
  });
});
```

Run:

```bash
npx wdio run wdio.conf.ts
```

---

## Page Object Pattern

```typescript
// test/pageobjects/login.page.ts
class LoginPage {
  get email() {
    return $("#email");
  }
  get password() {
    return $("#password");
  }
  get submit() {
    return $('button[type="submit"]');
  }

  async open() {
    await browser.url("/login");
  }

  async login(email: string, password: string) {
    await this.email.setValue(email);
    await this.password.setValue(password);
    await this.submit.click();
  }
}

export default new LoginPage();
```

```typescript
// test/specs/login.e2e.ts
import LoginPage from "../pageobjects/login.page";

describe("Login", () => {
  it("redirects to dashboard", async () => {
    await LoginPage.open();
    await LoginPage.login("user@example.com", "secret");
    await expect(browser).toHaveUrl(expect.stringContaining("/dashboard"));
  });
});
```

---

## API Testing (optional)

WDIO can also hit REST APIs in the same suite — useful with [[API - FastAPI]] smoke tests alongside UI flows:

```typescript
import axios from "axios";

it("health endpoint is up", async () => {
  const { status } = await axios.get("http://localhost:8000/health");
  expect(status).toBe(200);
});
```

For load thresholds use [[Load Testing — k6]] instead.

---

## wdio.conf.ts (essentials)

```typescript
export const config = {
  runner: "local",
  specs: ["./test/specs/**/*.ts"],
  maxInstances: 5,
  capabilities: [{
    browserName: "chrome",
    "goog:chromeOptions": {
      args: process.env.CI ? ["headless", "disable-gpu"] : [],
    },
  }],
  baseUrl: "http://localhost:5173",
  logLevel: "info",
  framework: "mocha",
  reporters: ["spec"],
  mochaOpts: { timeout: 60_000 },
};
```

Point `baseUrl` at [[Commands/JavaScript — Vite]] dev server or staging URL.

---

## WebdriverIO vs Puppeteer vs Playwright

| Need | Prefer |
| --- | --- |
| Cross-browser E2E suite (Node) | **WebdriverIO** |
| Chrome-only Node automation | [[Codes/JavaScript — Puppeteer]] |
| Python browser tests | [[Browser Automation — Playwright]] |
| JS load test in CI | [[Load Testing — k6]] |

---

## Quick Reference

| Task | API |
| --- | --- |
| Open URL | `browser.url("/path")` |
| Find element | `$("css")`, `$$("css")` |
| Assert | `expect(el).toHaveText("x")` |
| Click | `await el.click()` |
| Type | `await el.setValue("text")` |
| Wait | `await el.waitForDisplayed()` |
| Run | `npx wdio run wdio.conf.ts` |

---

## Related Notes

- [[Codes/JavaScript — Puppeteer]]
- [[Browser Automation — Playwright]]
- [[Codes/JavaScript — React]]
- [[Commands/JavaScript — Vite]]
- [[Unit Testing - pytest]] — Python unit tests (complement E2E)
- [[JavaScript Development]]

---

## Tags

#webdriverio #wdio #e2e #testing #javascript #selenium #webdriver #react #ci
