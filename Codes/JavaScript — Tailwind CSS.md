## What & When

**Tailwind CSS** is a **utility-first** CSS framework — you style components with composable classes (`flex`, `p-4`, `text-sm`) instead of writing separate CSS files. It pairs naturally with **React + Vite** and is the default styling path for **assistant-ui** shadcn themes.

```bash
npm install -D tailwindcss @tailwindcss/vite
```

Tailwind v4 uses the Vite plugin; v3 used `tailwind.config.js` + PostCSS — check your template version.

Hub: [[JavaScript Development]] · UI: [[Codes/JavaScript — React]]

---

## Vite + Tailwind v4

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
});
```

```css
/* src/index.css */
@import "tailwindcss";
```

Import in `main.tsx`: `import "./index.css"`.

---

## Utility Examples

```tsx
export function ProfileCard() {
  return (
    <div className="mx-auto max-w-md rounded-lg border border-gray-200 bg-white p-6 shadow-sm">
      <h2 className="text-lg font-semibold text-gray-900">Alice</h2>
      <p className="mt-2 text-sm text-gray-600">Backend engineer</p>
      <button className="mt-4 rounded bg-blue-600 px-4 py-2 text-sm font-medium text-white hover:bg-blue-700">
        Follow
      </button>
    </div>
  );
}
```

---

## Responsive & State Variants

```tsx
<div className="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">
  {/* cards */}
</div>

<button className="bg-blue-600 hover:bg-blue-700 disabled:opacity-50">
  Submit
</button>

<input className="border-gray-300 focus:border-blue-500 focus:ring-2 focus:ring-blue-200" />
```

Prefix pattern: `{breakpoint}:`, `hover:`, `focus:`, `dark:` (with dark mode config).

---

## Flex & Layout (common)

| Classes | Effect |
| --- | --- |
| `flex items-center justify-between` | horizontal bar |
| `flex flex-col gap-4` | vertical stack |
| `min-h-screen` | full viewport height |
| `container mx-auto px-4` | centered content |
| `truncate` | ellipsis overflow |
| `sr-only` | screen-reader only |

---

## `@apply` (optional)

Extract repeated utilities into a component class:

```css
@layer components {
  .btn-primary {
    @apply rounded bg-blue-600 px-4 py-2 text-white hover:bg-blue-700;
  }
}
```

Prefer utilities in JSX for most React code; use `@apply` sparingly.

---

## Tailwind vs Alternatives

| Need | Prefer |
| --- | --- |
| Utility-first React apps | **Tailwind** |
| Component library (MUI, Chakra) | separate design system |
| Django/Flask templates | Bootstrap or plain CSS — [[Web — Django]] |
| assistant-ui default theme | Tailwind + shadcn — [[Codes/JavaScript — assistant-ui]] |

---

## Dark Mode

```tsx
<html className="dark">
  {/* dark:bg-gray-900 dark:text-white on elements */}
</html>
```

Toggle with a class on `<html>` or use `prefers-color-scheme` strategy in config.

---

## Quick Reference

| Task | Classes |
| --- | --- |
| Padding | `p-4`, `px-6`, `py-2` |
| Margin | `m-4`, `mt-8`, `mb-2` |
| Text size | `text-sm`, `text-lg`, `text-xl` |
| Font weight | `font-medium`, `font-bold` |
| Colors | `text-gray-600`, `bg-white` |
| Rounded | `rounded`, `rounded-lg`, `rounded-full` |
| Shadow | `shadow`, `shadow-md` |

Docs: [tailwindcss.com](https://tailwindcss.com/docs)

---

## Related Notes

- [[Codes/JavaScript — React]]
- [[Commands/JavaScript — Vite]]
- [[Codes/JavaScript — assistant-ui]]
- [[JavaScript Development]]

---

## Tags

#tailwind #css #react #vite #frontend #utility-first #styling #web
