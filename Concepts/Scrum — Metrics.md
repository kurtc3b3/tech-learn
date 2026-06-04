**Key Points:**

- **Metrics serve the team** — forecast and improve; never compare teams or weaponize velocity.
- **Velocity** — story points completed per Sprint (Done only); rolling average for planning.
- **Burndown** — remaining work in a Sprint; flat lines and scope creep are warning patterns.
- **Burnup** — completed work vs total scope; scope changes stay visible.
- **Flow metrics** — cycle time and lead time complement Scrum’s timeboxed view.

# Scrum — Metrics

Part of [[Scrum]]. Concept-only.

---

## Why Measure in Scrum?

- Forecast releases and Sprint capacity
- Spot blockers and unhealthy patterns early
- Improve planning accuracy over time
- Communicate progress without fiction
- Track team learning — for **this team only**

> A team’s velocity is meaningful **only to that team**. Different teams use different scales.

---

## Velocity

**Velocity** = total story points of work that met **Definition of Done** in a Sprint.

### Rolling average

Use the last **3–5 Sprints**, drop outliers (holidays, major outages).

| Use | How |
| --- | --- |
| **Sprint Planning** | Pull items summing to ~average velocity |
| **Release forecast** | Remaining points ÷ average velocity ≈ Sprints left |
| **Improvement signal** | Stable or rising completion with stable quality |

### Velocity traps

| Trap | Fix |
| --- | --- |
| Velocity as management target | Inflate estimates; focus on value |
| Comparing teams | Meaningless — different scales |
| One bad Sprint panic | Use rolling average |
| Counting incomplete stories | Only **Done** counts |
| Adding people mid-Sprint | Expect dip before rise |

---

## Burndown Chart

Shows **remaining work** in a Sprint (points or tasks) over days. Ideal line trends to zero by Sprint end.

### Sprint patterns (conceptual)

| Pattern | Meaning |
| --- | --- |
| Steady slope to zero | Healthy progress |
| Flat then cliff | Blockers or poor breakdown; rush at end |
| Line rises | Scope added mid-Sprint — Sprint Goal at risk |
| Early zero | Overestimated; pull more Ready work or pay debt |
| Stuck above zero | Overcommit; items roll over |

---

## Release Burndown

Remaining backlog points **across Sprints** toward a release or Product Goal. Stakeholders ask: “Are we on track for the date?”

Pair with honest scope management — hiding new work skews the chart.

---

## Burnup Chart

Plots **work completed** rising toward a **total scope** line.

**Advantage:** when scope grows, the scope line moves up — stakeholders see **why** the date moved. Burndown alone can hide added work.

---

## Other Useful Metrics

| Metric | Definition | Insight |
| --- | --- | --- |
| **Cycle time** | Started → Done | Flow efficiency, bottlenecks |
| **Lead time** | Backlog entry → Done | Includes queue wait; customer wait |
| **Sprint Goal success** | % Sprints meeting goal | ~80–90% healthy; 100% may mean easy goals |
| **Escaped defects** | Bugs found after Sprint | Weak DoD or testing — [[Scrum — Framework]] |

---

## Metrics Dashboard (team-facing)

Combine a small set: average velocity, Sprint Goal hit rate, cycle time trend, escaped defects, burndown health, backlog Ready runway ([[Scrum — Product Backlog]]).

---

## Golden Rule

> **Metrics serve the team — the team does not serve the metrics.**

Use them in Retrospectives ([[Scrum — Daily Scrum & Retrospective]]) for better conversations — not pressure, ranking, or micromanagement.

---

## Related Notes

- [[Scrum]]
- [[Scrum — Sprint Planning & User Stories]]
- [[Scrum — Product Backlog]]
- [[Scrum — Scrum vs Kanban]] — cycle/lead time emphasis
- [[System Design — Economics & Performance]] — SLAs and KPIs at scale

---

## Tags

#scrum #velocity #burndown #burnup #cycle-time #metrics #forecasting
