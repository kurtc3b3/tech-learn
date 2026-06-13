## What & When

**axios** is a promise-based **HTTP client** for browser and Node. It adds interceptors, automatic JSON transforms, timeout config, and cleaner error objects than raw `fetch` — the default choice for React apps calling [[API - FastAPI]] backends.

```bash
npm install axios
```

Hub: [[JavaScript Development]] · UI: [[Codes/JavaScript — React]]

---

## Basic GET / POST

```typescript
import axios from "axios";

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL ?? "http://localhost:8000",
  timeout: 10_000,
  headers: { "Content-Type": "application/json" },
});

interface User {
  id: number;
  email: string;
}

// GET
const { data: users } = await api.get<User[]>("/users");

// POST
const { data: created } = await api.post<User>("/users", {
  email: "alice@example.com",
});
```

---

## Error Handling

```typescript
import axios, { isAxiosError } from "axios";

try {
  await api.get("/protected");
} catch (err) {
  if (isAxiosError(err)) {
    console.error(err.response?.status, err.response?.data);
  } else {
    throw err;
  }
}
```

---

## Interceptors (auth token)

```typescript
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      window.location.href = "/login";
    }
    return Promise.reject(err);
  }
);
```

---

## React Hook Pattern

```typescript
// src/api/users.ts
export async function fetchUsers(): Promise<User[]> {
  const { data } = await api.get<User[]>("/users");
  return data;
}
```

```tsx
// in component — with useEffect or TanStack Query
useEffect(() => {
  fetchUsers().then(setUsers).catch(setError);
}, []);
```

---

## axios vs fetch vs httpx

| | axios | `fetch` | [[Python — httpx Package]] |
| --- | --- | --- | --- |
| Runtime | Browser, Node | Browser, Node | Python |
| JSON auto | ✅ | manual `.json()` | ✅ |
| Interceptors | ✅ | ❌ | middleware-style |
| Timeouts | built-in | `AbortSignal` | ✅ |
| Best for | React → FastAPI | minimal deps | Python services |

---

## FastAPI CORS Reminder

When the React dev server runs on a different origin (e.g. `:5173` → `:8000`), enable CORS on [[API - FastAPI]]:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## Quick Reference

| Task | Code |
| --- | --- |
| Instance | `axios.create({ baseURL })` |
| GET | `api.get<T>(url)` |
| POST | `api.post<T>(url, body)` |
| PUT/PATCH | `api.put`, `api.patch` |
| DELETE | `api.delete(url)` |
| Query params | `api.get("/search", { params: { q: "x" } })` |
| Upload | `FormData` + `headers: { "Content-Type": "multipart/form-data" }` |

---

## Related Notes

- [[Codes/JavaScript — React]]
- [[API - FastAPI]]
- [[API - FastAPI — OpenAPI Specification]]
- [[Codes/TypeScript — Basics]]
- [[JavaScript Development]]

---

## Tags

#axios #javascript #typescript #http #react #fastapi #api #frontend
