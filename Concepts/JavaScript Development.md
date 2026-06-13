**Key Points:**

- **Node.js is the runtime** — JavaScript on the server, in CLIs, and as the engine behind frontend tooling; manage versions with **nvm** — [[Commands/JavaScript — Node Toolchain]].
- **TypeScript for app code** — static types on React/Next apps; plain JS for scripts and quick tools — [[Codes/TypeScript — Basics]], [[Codes/JavaScript — Basics]].
- **React is the default UI library** — component model for SPAs and dashboards; pair with **Vite** (dev) or **Next.js** (full-stack) — [[Codes/JavaScript — React]].
- **FastAPI backend + React frontend** — JSON over HTTP with **axios**; CORS and OpenAPI client generation from [[API - FastAPI]].
- **Tailwind for styling** — utility-first CSS; common with React, Vite, and assistant-ui — [[Codes/JavaScript — Tailwind CSS]].
- **AI chat UI** — **assistant-ui** for production chat/copilot components wired to [[AI]] backends.
- **Browser automation (Node)** — **Puppeteer** (Chrome CDP) or **WebdriverIO** (WebDriver tests); Python stack in [[Browser Automation]].

# JavaScript Development — Overview & Frontend Stack

## What is JavaScript Development (in this vault)?

**JavaScript development** here covers the **Node + TypeScript + React** ecosystem for building web UIs, calling [[API - FastAPI]] backends, charting data, and running browser tests — complementing the Python-centric notes in [[Python Development]].

Typical outcomes:

- **SPA or dashboard** — React + Vite + Tailwind consuming a REST API
- **Full-stack app** — Next.js API routes or server components + external Python API
- **AI chat UI** — assistant-ui + Vercel AI SDK or [[AI — LangChain]] backend
- **E2E tests** — WebdriverIO or Puppeteer against staging
- **Charts** — ApexCharts in React admin views

---

## Stack Decision Flow

```mermaid
flowchart TD
    START[New frontend?] --> FULL{Need SSR / routing / API routes?}
    FULL -->|yes| NEXT[[Commands/JavaScript — Next.js]]
    FULL -->|no| VITE[[Commands/JavaScript — Vite]] + [[Codes/JavaScript — React]]
    VITE --> API{Backend?}
    NEXT --> API
    API -->|Python REST| FAST[[API - FastAPI]] + [[Codes/JavaScript — axios]]
    API -->|same repo| NEXT
    VITE --> STYLE{Styling?}
    STYLE -->|utility CSS| TW[[Codes/JavaScript — Tailwind CSS]]
    STYLE -->|component lib| TW
    VITE --> AIUI{AI chat surface?}
    AIUI -->|yes| AUI[[Codes/JavaScript — assistant-ui]]
    VITE --> E2E{Browser tests?}
    E2E -->|WebDriver suite| WDIO[[Codes/JavaScript — WebdriverIO]]
    E2E -->|Chrome scripts| PUP[[Codes/JavaScript — Puppeteer]]
```

---

## Tool Roles

| Layer | Tool | References |
| --- | --- | --- |
| Runtime | Node.js | [[Commands/JavaScript — Node Toolchain]] |
| Version manager | nvm | [[Commands/JavaScript — Node Toolchain]] |
| Package managers | npm, yarn | [[Commands/JavaScript — Node Toolchain]] |
| Language | JavaScript, TypeScript | [[Codes/JavaScript — Basics]], [[Codes/TypeScript — Basics]] |
| UI library | React | [[Codes/JavaScript — React]] |
| Dev server / bundler | Vite | [[Commands/JavaScript — Vite]] |
| Full-stack framework | Next.js | [[Commands/JavaScript — Next.js]] |
| HTTP client | axios | [[Codes/JavaScript — axios]] |
| CSS | Tailwind CSS | [[Codes/JavaScript — Tailwind CSS]] |
| AI chat UI | assistant-ui | [[Codes/JavaScript — assistant-ui]] |
| Charts | ApexCharts | [[Codes/JavaScript — ApexCharts]] |
| Browser automation | Puppeteer, WebdriverIO | [[Codes/JavaScript — Puppeteer]], [[Codes/JavaScript — WebdriverIO]] |

---

## JavaScript in the Broader Landscape

| Concern | Python vault note | JavaScript note |
| --- | --- | --- |
| JSON API | [[API - FastAPI]] | [[Codes/JavaScript — axios]] |
| Server-rendered HTML | [[Web — Flask]], [[Web — Django]] | Next.js SSR or React SPA |
| Load testing (JS) | — | [[Load Testing — k6]] |
| Browser scrape (Python) | [[Browser Automation — Playwright]] | [[Codes/JavaScript — Puppeteer]] |
| AI agents / RAG | [[AI]] | [[Codes/JavaScript — assistant-ui]] |
| OpenAPI client gen | [[API - FastAPI — OpenAPI Specification]] | `openapi-typescript`, Orval |

---

## Recommended Learning Path

1. **Node toolchain** — install Node via nvm, npm/yarn basics — [[Commands/JavaScript — Node Toolchain]]
2. **JavaScript fundamentals** — types, async, modules — [[Codes/JavaScript — Basics]]
3. **TypeScript** — interfaces, generics, strict mode — [[Codes/TypeScript — Basics]]
4. **React** — components, hooks, data fetching — [[Codes/JavaScript — React]]
5. **Vite project** — fast local dev — [[Commands/JavaScript — Vite]]
6. **Call your API** — axios + env base URL — [[Codes/JavaScript — axios]]
7. **Style** — Tailwind utilities — [[Codes/JavaScript — Tailwind CSS]]
8. **Optional:** Next.js for routing/SSR — [[Commands/JavaScript — Next.js]]
9. **Optional:** assistant-ui for [[AI]] chat — [[Codes/JavaScript — assistant-ui]]
10. **Optional:** ApexCharts, Puppeteer, WebdriverIO as needed

---

## Related Notes

### Language & UI

- [[Codes/JavaScript — Basics]]
- [[Codes/TypeScript — Basics]]
- [[Codes/JavaScript — React]]

### Tooling

- [[Commands/JavaScript — Node Toolchain]]
- [[Commands/JavaScript — Vite]]
- [[Commands/JavaScript — Next.js]]

### Libraries

- [[Codes/JavaScript — axios]]
- [[Codes/JavaScript — Tailwind CSS]]
- [[Codes/JavaScript — assistant-ui]]
- [[Codes/JavaScript — ApexCharts]]

### Browser automation (Node)

- [[Codes/JavaScript — Puppeteer]]
- [[Codes/JavaScript — WebdriverIO]]

### Connected vault notes

- [[API - FastAPI]]
- [[Web]]
- [[AI]]
- [[Browser Automation]]
- [[Load Testing — k6]]
- [[Python Development]]

---

## Tags

#javascript #typescript #react #nodejs #frontend #vite #nextjs #tailwind #web
