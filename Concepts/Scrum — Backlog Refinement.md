**Key Points:**

- **Refinement is ongoing collaboration** — not an official Scrum event, but essential in practice.
- **Budget ~10% of Sprint capacity** — typically once or twice per Sprint, mid-Sprint timing.
- **Five to eight attendees** — Product Owner, Developers, Scrum Master; stakeholders only when needed.
- **Split until estimable** — patterns by role, workflow, data, happy path vs edge cases.
- **Spikes answer unknowns** — timeboxed research, not shippable Increment.

# Scrum — Backlog Refinement

Part of [[Scrum]]. Concept-only.

---

## What Is Backlog Refinement?

Also called grooming historically. The Product Owner and Developers **prepare** [[Scrum — Product Backlog]] items for future Sprints: clarity, size, estimates, and order.

Most teams hold a **recurring session** — not prescribed in the Scrum Guide, but standard in practice.

---

## Goals

- Clarify requirements and acceptance criteria
- Split oversized items
- Estimate relative effort (story points)
- Remove obsolete items
- Reorder by latest learning
- Surface dependencies and risks

---

## Cadence and Size

| Guideline | Detail |
| --- | --- |
| **Frequency** | 1–2 times per Sprint |
| **Duration** | 1–2 hours max per session |
| **Timing** | Mid-Sprint (avoid first/last day crunch) |
| **Capacity** | ~10% of Sprint effort (~2 h/week on a 2-week Sprint) |

---

## Who Attends

| Role | Role in session |
| --- | --- |
| **Product Owner** | Presents items, answers business questions |
| **Scrum Master** | Facilitates, timebox, clarifies process |
| **Developers** | Ask questions, estimate, flag technical risk |
| **Stakeholders** | Occasionally, for complex domains only |

Keep the group **small** (about 5–8) for speed.

---

## Session Flow

### Before (Product Owner)

- Reorder backlog
- Pick top 5–10 items to discuss
- Draft acceptance criteria
- Flag dependencies
- Share agenda early

### During

| Phase | Activity |
| --- | --- |
| Opening (~5 min) | Context, Sprint Goal reminder, agenda |
| Per item (~15–20 min) | Present → clarify → acceptance criteria → estimate → Ready or defer |
| Prioritization check (~10 min) | Order still correct? remove/park items? |
| Close (~5 min) | List Ready items, follow-ups, next session |

### Per-item micro-flow

```
Present → Questions → Acceptance criteria → Estimate → Ready ✅ or More work 🔄
```

---

## Splitting Stories

| Pattern | Example |
| --- | --- |
| **By user role** | Customer vs admin account stories |
| **By workflow step** | Address → delivery → payment → confirm |
| **By data variation** | Card vs PayPal vs saved card |
| **Happy path vs edge cases** | Reset works vs expired link error |
| **By quality slice** | Search works v1 → under 2s v2 |

If estimates stay at 13–21 or team says “?” — split further or **spike**.

---

## Estimation and Planning Poker

1. Story read aloud  
2. Each Developer picks a card privately (Fibonacci: 1, 2, 3, 5, 8, 13, 21, ?, ☕)  
3. Reveal together  
4. Discuss large gaps — they signal hidden complexity  
5. Re-estimate until aligned enough to plan  

| Card | Meaning |
| --- | --- |
| 1–3 | Small, well understood |
| 5–8 | Medium |
| 13–21 | Too large — split |
| ? | Need more information |
| ☕ | Pause — fatigue or rabbit hole |

Wide spread is **useful data**, not failure.

---

## Spikes

When unknowns block estimation, add a **timeboxed research** item:

- **Question:** e.g. which mapping API fits compliance needs?
- **Timebox:** e.g. one day
- **Output:** recommendation + refined stories — not production Increment

---

## Red Flags

| Warning | Action |
| --- | --- |
| Instant unanimous estimates | Probe assumptions |
| Same story returns every refinement | Split or spike |
| Always high points | Research spike first |
| PO cannot answer questions | Defer; PO prepares |
| Session overruns | Fewer items; better prep |

---

## Health Check After Refinement

- ~2 Sprints of **Ready** items at top?
- Top items estimated with clear acceptance criteria?
- Stale items removed?
- Team aligned on priority?

Smooth [[Scrum — Sprint Planning & User Stories]] depends on this runway.

---

## Related Notes

- [[Scrum]]
- [[Scrum — Product Backlog]]
- [[Scrum — Sprint Planning & User Stories]]
- [[Scrum — Metrics]]

---

## Tags

#scrum #refinement #grooming #planning-poker #spikes #splitting-stories
