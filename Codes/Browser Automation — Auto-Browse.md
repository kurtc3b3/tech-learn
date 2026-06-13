## What & When

**Auto-Browse** (`@auto-browse/auto-browse`) is a **TypeScript** library for **AI-enabled browser automation** on top of Playwright. You describe actions in **natural language** — `auto("click the Sign in button")` — instead of writing selectors; the LLM resolves elements and executes steps.

Use Auto-Browse when:

- Playwright tests or scripts need **NL commands** for flaky or changing UIs
- You want **BDD-style** steps with `playwright-bdd` and plain-English scenarios
- Developers keep Playwright for stable paths and delegate exploratory steps to `auto()`
- You already use **OpenAI, Anthropic, Google AI, or Ollama** in a TS/Node stack

For Python agent loops use [[AI — LangChain]] tools + [[Browser Automation — Playwright]], or [[Commands/CLI — agent-browser]] from the IDE. For stealth scraping see [[Browser Automation — Camoufox]]. See [[Browser Automation]] and [[AI]].

```bash
npm install @auto-browse/auto-browse
# Requires Playwright >= 1.53
npx playwright install chromium
```

Repo: [github.com/auto-browse/auto-browse-ts](https://github.com/auto-browse/auto-browse-ts)

---

## Auto-Browse vs agent-browser vs Playwright

| Need | Prefer | Why |
| --- | --- | --- |
| Typed locators, CI E2E | [[Browser Automation — Playwright]] | Deterministic, no LLM cost |
| NL in Playwright tests | **Auto-Browse** | `auto()` inside `@playwright/test` |
| IDE agent shell refs | [[Commands/CLI — agent-browser]] | Snapshot `@e1` without TS code |
| Python scrape pipeline | [[Browser Automation — Playwright]] / [[Browser Automation — Camoufox]] | Native Python stack |
| MCP browser server | [[Browser Automation — Obscura]] | CSS-selector MCP tools |

**Mental model:** Playwright owns the browser; Auto-Browse adds an **LLM layer** for element resolution and NL actions.

---

## Environment

```bash
# .env — pick one provider
OPENAI_API_KEY=sk-...
# ANTHROPIC_API_KEY=...
# GOOGLE_GENERATIVE_AI_API_KEY=...
```

Supported providers: OpenAI (default), Google AI, Anthropic, Ollama (local).

---

## Standalone Mode

Full form automation without `@playwright/test`:

```typescript
import { chromium } from "playwright";
import { auto } from "@auto-browse/auto-browse";

async function main() {
  const browser = await chromium.launch({ headless: false });
  const page = await browser.newPage();
  await page.goto("https://example.com/login");

  await auto("type user@example.com into the email field", { page });
  await auto("type mypassword into the password field", { page });
  await auto("click the Sign in button", { page });

  await page.waitForURL("**/dashboard**");
  console.log(await page.title());
  await browser.close();
}

main();
```

---

## Playwright Test Mode

```typescript
import { test, expect } from "@playwright/test";
import { auto } from "@auto-browse/auto-browse";

test("checkout with NL steps", async ({ page }) => {
  await page.goto("https://shop.example.com");

  await auto("add the first product to cart", { page });
  await auto("open the cart and proceed to checkout", { page });
  await auto("fill shipping with name Test User and city Austin", { page });

  await expect(page.getByText("Order confirmed")).toBeVisible();
});
```

---

## Auto-Detection Mode

When running inside Playwright test context, omit `page` — Auto-Browse detects it:

```typescript
import { test } from "@playwright/test";
import { auto } from "@auto-browse/auto-browse";

test("search", async ({ page }) => {
  await page.goto("https://example.com");
  await auto('search for "browser automation"');
  await auto("click the first result");
});
```

---

## BDD with playwright-bdd

```gherkin
Feature: Login
  Scenario: User signs in
    Given I am on the login page
    When I auto "enter valid credentials and submit"
    Then I should see the dashboard
```

```typescript
import { auto } from "@auto-browse/auto-browse";
// wire `auto` step definition in playwright-bdd config
```

Use NL for volatile UI; keep `Given`/`Then` assertions deterministic with Playwright locators.

---

## Hybrid Pattern (Recommended for Production)

```typescript
import { test, expect } from "@playwright/test";
import { auto } from "@auto-browse/auto-browse";

test("hybrid checkout", async ({ page }) => {
  await page.goto("https://shop.example.com");

  // Stable path — no LLM
  await page.getByRole("link", { name: "Catalog" }).click();
  await expect(page.locator(".product-card")).toHaveCount(20);

  // Flaky promo banner — NL once
  await auto("dismiss any cookie or promo popup if visible", { page });

  await page.locator(".product-card").first().click();
  await page.getByRole("button", { name: "Add to cart" }).click();
});
```

Cache repeated flows in Playwright; reserve `auto()` for layout drift and one-off exploration.

---

## Provider Configuration

```typescript
import { auto, configureAutoBrowse } from "@auto-browse/auto-browse";

configureAutoBrowse({
  provider: "anthropic",
  model: "claude-sonnet-4-5",
});

await auto("click Submit", { page });
```

Swap models with one config change — useful for cost vs quality tradeoffs in CI.

---

## AI Agent Stacks

| Layer | Tool |
| --- | --- |
| NL browser in TS tests | **Auto-Browse** |
| IDE shell automation | [[Commands/CLI — agent-browser]] |
| MCP tools | [[Browser Automation — Obscura]], [[AI — MCP]] |
| Multi-step Python agent | [[AI — LangGraph]] + Playwright tools |
| Stealth fetch | [[Browser Automation — Camoufox]] |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `npm i @auto-browse/auto-browse` |
| Import | `import { auto } from "@auto-browse/auto-browse"` |
| NL action | `await auto("click Sign in", { page })` |
| In test | `await auto("fill email with test@example.com")` |
| Providers | OpenAI, Anthropic, Google AI, Ollama |
| Playwright min | 1.53+ |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Commands/CLI — agent-browser]]
- [[Browser Automation — Obscura]]
- [[Browser Automation — Camoufox]]
- [[AI]]
- [[AI — LangChain]]

---

## Tags

#browser-automation #auto-browse #playwright #typescript #ai-agents #nl #testing #web #llm
