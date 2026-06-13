## What & When

**Next.js** is a **React framework** with file-based routing, server components (App Router), API routes, SSR/SSG, and built-in production optimizations. Use it when the frontend needs **routing, SEO, or colocated API routes** — especially with [[Codes/JavaScript — assistant-ui]] chat apps.

```bash
npx create-next-app@latest my-app
# TypeScript: Yes, ESLint: Yes, Tailwind: Yes, App Router: Yes

cd my-app && npm run dev
```

Hub: [[JavaScript Development]] · Simpler SPA path: [[Commands/JavaScript — Vite]]

---

## App Router Layout

```text
app/
  layout.tsx       # root shell
  page.tsx         # /
  dashboard/
    page.tsx       # /dashboard
  api/
    chat/
      route.ts     # POST /api/chat
```

---

## Page (Server Component default)

```tsx
// app/page.tsx
export default function HomePage() {
  return (
    <main className="p-8">
      <h1 className="text-2xl font-bold">Dashboard</h1>
    </main>
  );
}
```

Client interactivity — add `"use client"` at top of file (hooks, events).

---

## Client Component + Data Fetch

```tsx
// app/users/page.tsx
"use client";

import { useEffect, useState } from "react";
import axios from "axios";

interface User {
  id: number;
  name: string;
}

export default function UsersPage() {
  const [users, setUsers] = useState<User[]>([]);

  useEffect(() => {
    axios.get("/api/users").then((r) => setUsers(r.data));
  }, []);

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

Or proxy to [[API - FastAPI]] from a Route Handler.

---

## API Route (BFF to FastAPI)

```typescript
// app/api/users/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  const res = await fetch(`${process.env.FASTAPI_URL}/users`, {
    cache: "no-store",
  });
  const data = await res.json();
  return NextResponse.json(data);
}
```

```bash
# .env.local
FASTAPI_URL=http://localhost:8000
```

---

## Environment Variables

| Prefix | Exposed to browser |
| --- | --- |
| `NEXT_PUBLIC_*` | ✅ client bundles |
| no prefix | server only (Route Handlers, RSC) |

```bash
NEXT_PUBLIC_APP_NAME=MyApp
FASTAPI_URL=http://localhost:8000
```

---

## Scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  }
}
```

Production: `npm run build && npm run start` (or deploy to Vercel).

---

## Next.js vs Vite

| Need | Prefer |
| --- | --- |
| File routing + SSR/SEO | **Next.js** |
| Lightweight SPA + FastAPI | [[Commands/JavaScript — Vite]] |
| AI chat UI scaffold | **Next.js** + [[Codes/JavaScript — assistant-ui]] |
| Static-only site | Vite or Next static export |

---

## Tailwind in Next

Enabled by `create-next-app` — see [[Codes/JavaScript — Tailwind CSS]]. Global styles in `app/globals.css`.

---

## Quick Reference

| Task | Pattern |
| --- | --- |
| Create | `npx create-next-app@latest` |
| Page | `app/route/page.tsx` |
| API | `app/api/x/route.ts` |
| Client | `"use client"` directive |
| Env (public) | `NEXT_PUBLIC_*` |
| Dev | `npm run dev` (:3000) |

Docs: [nextjs.org/docs](https://nextjs.org/docs)

---

## Related Notes

- [[Codes/JavaScript — React]]
- [[Codes/JavaScript — assistant-ui]]
- [[Codes/JavaScript — Tailwind CSS]]
- [[Codes/JavaScript — axios]]
- [[Commands/JavaScript — Vite]]
- [[Commands/JavaScript — Node Toolchain]]
- [[API - FastAPI]]
- [[JavaScript Development]]

---

## Tags

#nextjs #react #typescript #ssr #app-router #frontend #fullstack #web
