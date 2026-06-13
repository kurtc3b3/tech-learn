## What & When

**Vite** is a **fast dev server and production bundler** for modern frontends. It is the default way to scaffold **React + TypeScript** SPAs in this vault — HMR, ES modules, and optimized Rollup builds.

Use Vite when:

- Building a **SPA** that talks to [[API - FastAPI]] via [[Codes/JavaScript — axios]]
- You want the fastest local dev loop (vs heavier webpack setups)
- You do **not** need Next.js SSR/routing — see [[Commands/JavaScript — Next.js]] when you do

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev
```

Hub: [[JavaScript Development]] · Runtime: [[Commands/JavaScript — Node Toolchain]]

---

## Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  }
}
```

| Command | Role |
| --- | --- |
| `npm run dev` | Dev server (default `:5173`) |
| `npm run build` | Production bundle → `dist/` |
| `npm run preview` | Serve `dist/` locally |

---

## vite.config.ts

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    port: 5173,
    proxy: {
      "/api": {
        target: "http://localhost:8000",
        changeOrigin: true,
      },
    },
  },
});
```

**Proxy** avoids CORS during dev — requests to `/api/*` forward to [[API - FastAPI]].

---

## Environment Variables

Only variables prefixed with `VITE_` are exposed to client code:

```bash
# .env
VITE_API_URL=http://localhost:8000
```

```typescript
const baseUrl = import.meta.env.VITE_API_URL;
```

---

## Project Structure

```text
my-app/
  src/
    components/
    App.tsx
    main.tsx
  index.html          # entry HTML
  vite.config.ts
  tsconfig.json
  package.json
```

See [[Codes/JavaScript — React]], [[Codes/JavaScript — Tailwind CSS]].

---

## Build Output

```bash
npm run build
# dist/index.html + hashed JS/CSS assets
```

Serve `dist/` with nginx, S3, or [[K8S]] Ingress static hosting. API stays on FastAPI/Uvicorn.

---

## Vite vs Next.js

| Need | Prefer |
| --- | --- |
| SPA + external API | **Vite** |
| SSR, file routing, API routes | [[Commands/JavaScript — Next.js]] |
| assistant-ui scaffold | often Next — [[Codes/JavaScript — assistant-ui]] |

---

## Add Tailwind

See [[Codes/JavaScript — Tailwind CSS]] — `@tailwindcss/vite` plugin in `vite.config.ts`.

---

## Quick Reference

| Task | Command / config |
| --- | --- |
| Create | `npm create vite@latest -- --template react-ts` |
| Dev | `npm run dev` |
| Build | `npm run build` |
| Proxy API | `server.proxy` in config |
| Env | `VITE_*` in `.env` |
| Import alias | `resolve.alias` in config |

Docs: [vite.dev](https://vite.dev/)

---

## Related Notes

- [[Codes/JavaScript — React]]
- [[Codes/JavaScript — Tailwind CSS]]
- [[Codes/JavaScript — axios]]
- [[Commands/JavaScript — Node Toolchain]]
- [[Commands/JavaScript — Next.js]]
- [[API - FastAPI]]
- [[JavaScript Development]]

---

## Tags

#vite #react #typescript #bundler #dev-server #frontend #hmr #web
