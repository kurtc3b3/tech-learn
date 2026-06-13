## What & When

**JavaScript** is the language of the browser and of **Node.js** on the server. In this vault it is the foundation for React apps, Vite/Next tooling, axios clients against [[API - FastAPI]], and Node-side automation ([[Codes/JavaScript — Puppeteer]]).

Use modern **ES modules** (`import`/`export`), **`async`/`await`**, and **`fetch`** (or [[Codes/JavaScript — axios]]) for HTTP. Prefer **TypeScript** for application code — [[Codes/TypeScript — Basics]].

Hub: [[JavaScript Development]].

---

## Run JavaScript

```bash
node script.js              # run a file
node --watch script.js      # restart on change (Node 18+)
node -e "console.log(2+2)"  # one-liner
```

In browsers, use `<script type="module" src="app.js">` or a bundler ([[Commands/JavaScript — Vite]]).

---

## Variables & Types

```javascript
const name = "Alice";           // constant binding
let count = 0;                  // reassignable
// avoid var in new code

const items = [1, 2, 3];
const user = { id: 1, role: "admin" };

typeof "text";    // "string"
typeof 42;        // "number"
typeof true;      // "boolean"
typeof undefined; // "undefined"
typeof null;      // "object" (historical quirk)
```

**Truthy / falsy:** `false`, `0`, `""`, `null`, `undefined`, `NaN` are falsy; everything else is truthy.

---

## Functions

```javascript
function add(a, b) {
  return a + b;
}

const multiply = (a, b) => a * b;

async function fetchJson(url) {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
```

---

## Destructuring & Spread

```javascript
const { id, role } = user;
const [first, ...rest] = items;

const merged = { ...user, role: "editor" };
const combined = [...items, 4];
```

---

## Modules (ESM)

```javascript
// math.js
export function sum(a, b) {
  return a + b;
}
export default sum;

// app.js
import sum, { sum as add } from "./math.js";
```

In Node, set `"type": "module"` in `package.json` or use `.mjs` extension.

---

## Arrays & Objects (common patterns)

```javascript
items.map((x) => x * 2);
items.filter((x) => x > 1);
items.find((x) => x === 2);
items.reduce((acc, x) => acc + x, 0);

Object.keys(user);
Object.entries(user).map(([k, v]) => `${k}=${v}`);
```

---

## Promises & Async

```javascript
// Promise chain
fetch("/api/health")
  .then((r) => r.json())
  .then(console.log)
  .catch(console.error);

// Preferred: async/await
async function load() {
  try {
    const data = await fetchJson("/api/users");
    return data;
  } catch (err) {
    console.error(err);
    throw err;
  }
}

await Promise.all([fetchJson("/a"), fetchJson("/b")]);
```

Pair with [[Codes/JavaScript — axios]] for interceptors and typed responses in apps.

---

## JSON

```javascript
const obj = { name: "Alice", active: true };
const text = JSON.stringify(obj);
const parsed = JSON.parse(text);
```

---

## Error Handling

```javascript
try {
  await risky();
} catch (err) {
  if (err instanceof TypeError) {
    // handle
  }
  throw err;   // rethrow if not handled
}
```

---

## Browser vs Node

| API | Browser | Node |
| --- | --- | --- |
| DOM | `document`, `window` | — |
| HTTP | `fetch`, [[Codes/JavaScript — axios]] | `fetch` (18+), `http`, axios |
| Files | File API | `fs/promises` |
| Env vars | — | `process.env` |

---

## Quick Reference

| Task | Syntax |
| --- | --- |
| Constant | `const x = 1` |
| Arrow fn | `(a, b) => a + b` |
| Async | `async function f() { await ... }` |
| Import | `import x from "./mod.js"` |
| Export | `export function f() {}` |
| Optional chain | `user?.address?.city` |
| Nullish coalesce | `value ?? "default"` |

---

## Related Notes

- [[Codes/TypeScript — Basics]]
- [[Codes/JavaScript — React]]
- [[Codes/JavaScript — axios]]
- [[Commands/JavaScript — Node Toolchain]]
- [[JavaScript Development]]
- [[Load Testing — k6]]

---

## Tags

#javascript #nodejs #esm #async #frontend #basics #web
