## What & When

**React** is a **component-based UI library** for building interactive web apps. You compose **function components** with **hooks** (`useState`, `useEffect`, …) and render to the DOM via a root (typically Vite or Next.js in this vault).

Use React when:

- You need a **SPA** or dashboard against a [[API - FastAPI]] backend
- You want a rich component ecosystem (charts, chat UI, forms)
- You pair with **TypeScript** — [[Codes/TypeScript — Basics]]

Use server-rendered Python ([[Web — Flask]], [[Web — Django]]) when HTML-from-server is enough without a heavy client bundle.

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install && npm run dev
```

Hub: [[JavaScript Development]] · Styling: [[Codes/JavaScript — Tailwind CSS]] · HTTP: [[Codes/JavaScript — axios]]

---

## Hello Component

```tsx
// src/App.tsx
export default function App() {
  return (
    <main className="p-4">
      <h1>Hello React</h1>
    </main>
  );
}
```

```tsx
// src/main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

---

## State — `useState`

```tsx
import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount((c) => c + 1)}>
      Count: {count}
    </button>
  );
}
```

---

## Effects — `useEffect`

```tsx
import { useEffect, useState } from "react";
import axios from "axios";

interface User {
  id: number;
  name: string;
}

export function UserList() {
  const [users, setUsers] = useState<User[]>([]);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    axios
      .get<User[]>("/api/users")
      .then((res) => setUsers(res.data))
      .catch((err) => setError(err.message));
  }, []);   // empty deps = run once on mount

  if (error) return <p>Error: {error}</p>;
  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

Prefer **data libraries** (TanStack Query) in production; the pattern above shows the core hook model.

---

## Props & Composition

```tsx
interface CardProps {
  title: string;
  children: React.ReactNode;
}

export function Card({ title, children }: CardProps) {
  return (
    <section className="rounded border p-4">
      <h2 className="font-semibold">{title}</h2>
      {children}
    </section>
  );
}
```

---

## Forms (controlled inputs)

```tsx
import { useState, type FormEvent } from "react";

export function LoginForm({ onSubmit }: { onSubmit: (email: string) => void }) {
  const [email, setEmail] = useState("");

  function handleSubmit(e: FormEvent) {
    e.preventDefault();
    onSubmit(email);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="you@example.com"
      />
      <button type="submit">Sign in</button>
    </form>
  );
}
```

---

## Context (shared state)

```tsx
import { createContext, useContext, useState, type ReactNode } from "react";

const ThemeContext = createContext<"light" | "dark">("light");

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<"light" | "dark">("light");
  return (
    <ThemeContext.Provider value={theme}>
      {children}
      <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
        Toggle theme
      </button>
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  return useContext(ThemeContext);
}
```

---

## React vs Alternatives

| Need | Prefer |
| --- | --- |
| Component SPA + TS | **React** + Vite |
| Full-stack routing/SSR | [[Commands/JavaScript — Next.js]] |
| Server HTML only | [[Web — Flask]], [[Web — Django]] |
| AI chat UI primitives | [[Codes/JavaScript — assistant-ui]] |

---

## Project Layout (Vite + React + TS)

```text
src/
  components/
  hooks/
  api/           # axios wrappers
  App.tsx
  main.tsx
index.html
vite.config.ts
```

---

## Quick Reference

| Task | API |
| --- | --- |
| State | `useState(initial)` |
| Side effect | `useEffect(fn, deps)` |
| Memo | `useMemo`, `useCallback` |
| Ref | `useRef` |
| List keys | `key={item.id}` on siblings |
| Event | `onClick={() => ...}` |
| Conditional | `{ok && <Badge />}` |

---

## Related Notes

- [[Codes/TypeScript — Basics]]
- [[Codes/JavaScript — axios]]
- [[Codes/JavaScript — Tailwind CSS]]
- [[Codes/JavaScript — assistant-ui]]
- [[Commands/JavaScript — Vite]]
- [[Commands/JavaScript — Next.js]]
- [[API - FastAPI]]
- [[JavaScript Development]]

---

## Tags

#react #typescript #frontend #hooks #components #vite #spa #web
