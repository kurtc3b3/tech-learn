## What & When

**assistant-ui** is an open-source **React/TypeScript** library for **production AI chat UIs** — Thread, Message, Composer, streaming, markdown, tool-call rendering, and adapters for Vercel AI SDK, LangGraph, and [[AI — MCP]]-adjacent backends.

Use assistant-ui when:

- You ship a **ChatGPT-style** or **copilot** interface in React/Next
- You want composable primitives instead of building scroll/stream/markdown from scratch
- Your backend is [[API - FastAPI]], Vercel AI SDK, [[AI — LangChain]], or [[AI — LangGraph]]

```bash
npx assistant-ui@latest create    # new Next.js project
npx assistant-ui@latest init      # add to existing project

# or manual:
npm install @assistant-ui/react @assistant-ui/react-ai-sdk
```

Docs: [assistant-ui.com](https://www.assistant-ui.com/) · Hub: [[JavaScript Development]], [[AI]]

---

## Vercel AI SDK Integration

```tsx
// app/page.tsx (Next.js App Router)
"use client";

import { AssistantRuntimeProvider } from "@assistant-ui/react";
import { useChatRuntime } from "@assistant-ui/react-ai-sdk";
import { Thread } from "@/components/assistant-ui/thread";

export default function ChatPage() {
  const runtime = useChatRuntime({
    api: "/api/chat",   // Next.js route or proxy to FastAPI
  });

  return (
    <AssistantRuntimeProvider runtime={runtime}>
      <div className="h-dvh">
        <Thread />
      </div>
    </AssistantRuntimeProvider>
  );
}
```

API route forwards to your LLM provider or [[API - FastAPI]] backend.

---

## Proxy to FastAPI Backend

Next.js route handler:

```typescript
// app/api/chat/route.ts
import { NextRequest } from "next/server";

export async function POST(req: NextRequest) {
  const body = await req.json();

  const res = await fetch(`${process.env.FASTAPI_URL}/chat`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });

  return new Response(res.body, {
    headers: { "Content-Type": res.headers.get("Content-Type") ?? "text/plain" },
  });
}
```

Use streaming responses from FastAPI ([[API - FastAPI — Server-Sent Events (SSE)]]) when the UI expects token streams.

---

## Composable Primitives

| Component | Role |
| --- | --- |
| `Thread` | Message list + scroll |
| `Composer` | User input, attachments |
| `Message` | Single turn rendering |
| `ThreadList` | Multi-conversation sidebar |
| `ActionBar` | Copy, regenerate, etc. |

Style with [[Codes/JavaScript — Tailwind CSS]]; CLI can copy shadcn/ui theme files.

---

## Integration Packages

| Backend | Package |
| --- | --- |
| Vercel AI SDK | `@assistant-ui/react-ai-sdk` |
| LangGraph / LangChain | `@assistant-ui/react-langgraph`, `@assistant-ui/react-langchain` |
| Custom stream | `@assistant-ui/react-data-stream` |
| A2A / AG-UI | `@assistant-ui/react-a2a`, `@assistant-ui/react-ag-ui` |
| Google ADK | `@assistant-ui/react-google-adk` |

See [[AI]] for agent stack; [[Codes/JavaScript — React]] for component basics.

---

## Features (built-in)

- Streaming token display + auto-scroll
- Markdown + syntax highlighting
- File attachments
- Tool / generative UI slots
- Keyboard shortcuts, a11y patterns

---

## assistant-ui vs DIY React

| Need | Prefer |
| --- | --- |
| Production chat UX fast | **assistant-ui** |
| Fully custom non-chat UI | [[Codes/JavaScript — React]] only |
| Python-only demo | Streamlit / Gradio (outside vault) |
| IDE agent browser | [[Commands/CLI — agent-browser]] |

---

## Quick Reference

| Task | Command / import |
| --- | --- |
| Scaffold | `npx assistant-ui@latest create` |
| Add to app | `npx assistant-ui@latest init` |
| Runtime | `useChatRuntime({ api: "/api/chat" })` |
| Provider | `<AssistantRuntimeProvider runtime={runtime}>` |
| UI | `<Thread />` |

---

## Related Notes

- [[AI]]
- [[Codes/JavaScript — React]]
- [[Codes/JavaScript — Tailwind CSS]]
- [[Commands/JavaScript — Next.js]]
- [[API - FastAPI]]
- [[AI — LangGraph]]
- [[JavaScript Development]]

---

## Tags

#assistant-ui #react #typescript #ai #chat #llm #nextjs #tailwind #frontend #copilot
