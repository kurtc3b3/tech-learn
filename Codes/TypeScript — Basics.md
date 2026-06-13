## What & When

**TypeScript** is **JavaScript with static types** — compiles to JS, catches errors at build time, and powers most React/Next/Vite projects in this vault.

Use TypeScript when:

- Building **React** or **Next.js** apps ([[Codes/JavaScript — React]], [[Commands/JavaScript — Next.js]])
- Sharing **API contracts** with [[API - FastAPI]] (OpenAPI → generated types)
- You want IDE autocomplete and refactor safety on non-trivial frontends

Use plain [[Codes/JavaScript — Basics]] for one-off Node scripts or when tooling does not need types.

```bash
npm install -D typescript
npx tsc --init
```

Hub: [[JavaScript Development]].

---

## Minimal Setup

`tsconfig.json` (Vite/React starter defaults):

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "react-jsx",
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

Compile check only: `npx tsc --noEmit`.

---

## Basic Types

```typescript
let count: number = 0;
let name: string = "Alice";
let active: boolean = true;
let ids: number[] = [1, 2, 3];
let tuple: [string, number] = ["age", 30];

// Inference — prefer when obvious
const port = 3000;   // number
```

---

## Interfaces & Types

```typescript
interface User {
  id: number;
  email: string;
  role?: "admin" | "user";   // optional + union
}

type ApiResponse<T> = {
  data: T;
  error?: string;
};

function greet(user: User): string {
  return `Hello, ${user.email}`;
}
```

Use `interface` for object shapes; `type` for unions, intersections, and aliases.

---

## Generics

```typescript
function first<T>(items: T[]): T | undefined {
  return items[0];
}

async function getJson<T>(url: string): Promise<T> {
  const res = await fetch(url);
  return res.json() as Promise<T>;
}

interface Paginated<T> {
  items: T[];
  total: number;
}
```

---

## Enums & `as const`

```typescript
// Prefer string unions over numeric enums in app code
type Status = "pending" | "done" | "failed";

const ROLES = ["admin", "user"] as const;
type Role = (typeof ROLES)[number];
```

---

## Narrowing

```typescript
function printId(id: string | number) {
  if (typeof id === "string") {
    console.log(id.toUpperCase());
  } else {
    console.log(id.toFixed(0));
  }
}
```

---

## React + TypeScript

```typescript
import { useState, type FormEvent } from "react";

interface Props {
  title: string;
  onSave: (value: string) => void;
}

export function Editor({ title, onSave }: Props) {
  const [text, setText] = useState("");

  function handleSubmit(e: FormEvent) {
    e.preventDefault();
    onSave(text);
  }

  return (
    <form onSubmit={handleSubmit}>
      <h1>{title}</h1>
      <input value={text} onChange={(e) => setText(e.target.value)} />
    </form>
  );
}
```

See [[Codes/JavaScript — React]].

---

## API Types from OpenAPI

Generate types from [[API - FastAPI]] OpenAPI schema:

```bash
npx openapi-typescript http://localhost:8000/openapi.json -o src/api/schema.d.ts
```

```typescript
import type { paths } from "./api/schema";

type UserList = paths["/users"]["get"]["responses"][200]["content"]["application/json"];
```

---

## TypeScript vs JavaScript

| Need | Use |
| --- | --- |
| React/Next app | **TypeScript** |
| k6 load script | JavaScript — [[Load Testing — k6]] |
| Quick Node script | JavaScript or `tsx` for TS on the fly |
| Shared backend types | OpenAPI codegen |

---

## Quick Reference

| Task | Syntax |
| --- | --- |
| Type param | `function f<T>(x: T)` |
| Optional prop | `name?: string` |
| Union | `string \| number` |
| Readonly | `readonly id: number` |
| Import type | `import type { User } from "./types"` |
| Assert | `value as string` (use sparingly) |
| Strict null | enable `"strict": true` |

---

## Related Notes

- [[Codes/JavaScript — Basics]]
- [[Codes/JavaScript — React]]
- [[Commands/JavaScript — Vite]]
- [[Commands/JavaScript — Next.js]]
- [[API - FastAPI — OpenAPI Specification]]
- [[JavaScript Development]]

---

## Tags

#typescript #javascript #react #types #frontend #strict #web
