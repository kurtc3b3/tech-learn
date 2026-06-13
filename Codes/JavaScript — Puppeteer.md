## What & When

**Puppeteer** is a **Node.js** library that controls **Chrome/Chromium** over the **Chrome DevTools Protocol (CDP)**. Use it for scraping, PDF/screenshot generation, and E2E-style automation when your stack is JavaScript-first.

Use Puppeteer when:

- You need **Node** browser scripts (not Python [[Browser Automation — Playwright]])
- You connect to **remote CDP** ([[Browser Automation — Obscura]], Chrome `--remote-debugging-port`)
- You want **`puppeteer-core`** without bundling Chrome

For cross-browser **test suites**, prefer [[Codes/JavaScript — WebdriverIO]]. For Python scraping, see [[Browser Automation]].

```bash
npm install puppeteer          # includes Chromium download
npm install puppeteer-core     # connect to existing Chrome
```

Hub: [[JavaScript Development]] · Compare: [[Browser Automation — Playwright]]

---

## Basic Scrape

```javascript
import puppeteer from "puppeteer";

const browser = await puppeteer.launch({ headless: true });
const page = await browser.newPage();

await page.goto("https://books.toscrape.com", {
  waitUntil: "domcontentloaded",
});

const titles = await page.$$eval("article h3 a", (els) =>
  els.map((el) => el.textContent?.trim())
);

console.log(titles);
await browser.close();
```

---

## Screenshot & PDF

```javascript
await page.goto("https://example.com", { waitUntil: "networkidle2" });
await page.screenshot({ path: "page.png", fullPage: true });
await page.pdf({ path: "page.pdf", format: "A4" });
```

---

## Connect to Remote CDP

Use with [[Browser Automation — Obscura]] or headless Chrome:

```javascript
import puppeteer from "puppeteer-core";

const browser = await puppeteer.connect({
  browserWSEndpoint: "ws://127.0.0.1:9222/devtools/browser",
});

const page = await browser.newPage();
await page.goto("https://example.com");
console.log(await page.title());
await browser.disconnect();
```

Use **`puppeteer-core`**, not `puppeteer`, when an external browser is already running.

---

## Login Flow

```javascript
await page.goto("https://example.com/login");
await page.type("#email", "user@example.com");
await page.type("#password", "secret");
await page.click('button[type="submit"]');
await page.waitForNavigation({ waitUntil: "networkidle2" });
```

Modern API: `page.locator('#email').fill('...')` in Puppeteer v22+.

---

## Intercept Network

```javascript
await page.setRequestInterception(true);
page.on("request", (req) => {
  if (req.resourceType() === "image") {
    req.abort();
  } else {
    req.continue();
  }
});
```

---

## Puppeteer vs Playwright vs WebdriverIO

| Need | Prefer |
| --- | --- |
| Python E2E / scrape | [[Browser Automation — Playwright]] |
| Node CDP scripts | **Puppeteer** |
| Multi-browser test framework | [[Codes/JavaScript — WebdriverIO]] |
| Rust CDP server | [[Browser Automation — Obscura]] |
| IDE agent CLI | [[Commands/CLI — agent-browser]] |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Launch | `puppeteer.launch({ headless: true })` |
| New page | `browser.newPage()` |
| Navigate | `page.goto(url, { waitUntil: "domcontentloaded" })` |
| Select | `page.$`, `page.$$`, `page.$eval` |
| Click | `page.click("selector")` |
| Content | `page.content()` |
| Connect CDP | `puppeteer.connect({ browserWSEndpoint })` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Browser Automation — Obscura]]
- [[Codes/JavaScript — WebdriverIO]]
- [[JavaScript Development]]

---

## Tags

#puppeteer #javascript #nodejs #cdp #scraping #browser-automation #chrome #e2e
