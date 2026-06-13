## What & When

**agent-browser** is Vercel Labs' **Rust CLI** for browser automation aimed at **AI coding agents** (Cursor, Claude Code, Codex, Copilot). It uses a **snapshot + ref** model: `snapshot -i` returns a compact accessibility tree with refs like `@e1`, `@e2` — far fewer tokens than full DOM JSON.

Use agent-browser when:

- An LLM agent in the IDE needs **deterministic browser control** via shell commands
- You want **token-efficient** page state for the model (~200–400 tokens vs 3k–5k DOM)
- You need sessions, auth state, network capture, or streaming without writing Playwright scripts
- You integrate via **Agent Skills** (`npx skills add vercel-labs/agent-browser`)

For Python Playwright scripts see [[Browser Automation — Playwright]]. For stealth Firefox see [[Browser Automation — Camoufox]]. For Rust CDP + MCP see [[Browser Automation — Obscura]]. Overview: [[Browser Automation]], [[AI]].

```bash
npm install -g agent-browser
agent-browser install    # Chrome for Testing (first run)
# macOS: brew install agent-browser
```

Site: [agent-browser.dev](https://agent-browser.dev) · Repo: [vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser)

---

## agent-browser vs Playwright vs Obscura MCP

| Need | Prefer | Why |
| --- | --- | --- |
| Full test suite / locators | [[Browser Automation — Playwright]] | Rich API, tracing |
| IDE agent shell loop | **agent-browser** | Refs, compact snapshot, skills |
| Python stealth scrape | [[Browser Automation — Camoufox]] | Anti-detect Firefox |
| MCP browser tools (CSS) | [[Browser Automation — Obscura]] | `browser_click` by selector |
| NL test authoring | [[Browser Automation — Auto-Browse]] | TypeScript `auto()` |

**Architecture:** Rust CLI → native daemon → **CDP** → Chrome for Testing. No Playwright install required for end users.

---

## Core Workflow (Agent Loop)

```bash
agent-browser open https://example.com/login
agent-browser snapshot -i          # interactive elements only
# [ref=e1] textbox "Email"
# [ref=e2] textbox "Password"
# [ref=e3] button "Sign in"

agent-browser fill @e1 "user@example.com"
agent-browser fill @e2 "secret"
agent-browser click @e3
agent-browser wait --load networkidle
agent-browser snapshot -i
agent-browser screenshot page.png
agent-browser close
```

Re-snapshot after every navigation or DOM change — refs are invalid after page updates.

---

## Snapshot & Refs

```bash
agent-browser snapshot           # full a11y tree
agent-browser snapshot -i        # buttons, inputs, links only (preferred)
agent-browser snapshot -i --json # machine-readable for agents
```

Refs (`@e1`, `@e2`) point to elements from the **latest** snapshot — deterministic for LLMs without CSS knowledge.

### Alternative selectors

```bash
agent-browser click "button[type=submit]"
agent-browser fill "#email" "user@example.com"
```

CSS works when structure is known; refs are preferred for agent autonomy.

---

## Sessions & Parallel Agents

```bash
agent-browser --session agent1 open https://site-a.com
agent-browser --session agent2 open https://site-b.com

agent-browser --session agent1 snapshot -i
agent-browser --session agent2 snapshot -i
```

Isolated browser contexts per session name — useful for multi-tab agent swarms.

---

## Auth, Storage, Network

```bash
agent-browser open https://app.example.com
# ... login flow ...
agent-browser state save auth.json

agent-browser open https://app.example.com --state auth.json

agent-browser network                    # list requests
agent-browser network --filter xhr
agent-browser cookies
```

Other commands: `scroll`, `hover`, `select`, `upload`, `download`, `tab new`, `frame`, `eval`, `video start/stop`, `stream` (live view).

---

## Headed Debugging

```bash
agent-browser open https://example.com --headed
# or: AGENT_BROWSER_HEADED=1 agent-browser open ...
```

Use headed mode to debug ref mismatches; switch back to headless for CI/agent runs.

---

## Connect to Existing Chrome / CDP

```bash
agent-browser --cdp 9222 open about:blank
agent-browser --auto-connect open https://example.com
```

Pair with [[Browser Automation — Obscura]] CDP server or local Chrome with remote debugging.

---

## AI Agent Integration

### Agent Skills (recommended)

```bash
npx skills add vercel-labs/agent-browser
```

Adds skill docs so Cursor/Claude Code know the snapshot-ref workflow.

### Cursor rule snippet

```markdown
## Browser Automation

Use `agent-browser` for web tasks. Run `agent-browser --help` for commands.

1. `agent-browser open <url>`
2. `agent-browser snapshot -i` — get refs (@e1, @e2)
3. `agent-browser click @e1` / `fill @e2 "text"`
4. Re-snapshot after page changes
```

### Natural language (single shot)

```bash
agent-browser chat "fill the login form with test credentials and click submit"
agent-browser chat --headed "debug why checkout fails"
```

---

## Command Chaining

```bash
agent-browser open example.com && \
  agent-browser snapshot -i && \
  agent-browser fill @e1 "value" && \
  agent-browser click @e2 && \
  agent-browser wait --load networkidle && \
  agent-browser get url
```

---

## Environment Variables (common)

| Variable | Role |
| --- | --- |
| `AGENT_BROWSER_HEADED` | Show browser window |
| `AGENT_BROWSER_COLOR_SCHEME` | `dark` / `light` |
| `AGENT_BROWSER_HIDE_SCROLLBARS` | Screenshot consistency |

---

## Quick Reference

| Task | Command |
| --- | --- |
| Install | `npm i -g agent-browser && agent-browser install` |
| Open | `agent-browser open <url>` |
| Snapshot | `agent-browser snapshot -i` |
| Click | `agent-browser click @e1` |
| Fill | `agent-browser fill @e2 "text"` |
| Wait | `agent-browser wait --load networkidle` |
| Screenshot | `agent-browser screenshot out.png` |
| Session | `agent-browser --session name open <url>` |
| JSON | `agent-browser snapshot -i --json` |
| Close | `agent-browser close` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Browser Automation — Obscura]]
- [[Browser Automation — Camoufox]]
- [[Browser Automation — Auto-Browse]]
- [[AI — MCP]]
- [[AI]]
- [[CLI]]

---

## Tags

#cli #agent-browser #browser-automation #ai-agents #rust #cdp #cursor #scraping #web
