**Key Points:**

- **Stakeholder map before design** — identify sponsors, blockers, and users; conflicting priorities are normal, not exceptional.
- **Same diagram, different story** — executives need outcomes; engineers need boundaries; operators need failure modes.
- **ADRs and diagrams are communication** — C4 and UML reduce ambiguity; see [[System Design — Governance & Documentation]] for records.
- **Facilitation beats broadcasting** — listen first, then propose; whiteboarding is a shared thinking tool.
- **Negotiation is part of communication** — scope, vendors, and timelines need explicit trade-off language.

# System Design — Stakeholders & Communication

Part of [[System Design]]. Concept-only.

---

## Stakeholder Management

### Identify and map stakeholders

| Dimension | Ask |
| --- | --- |
| **Interest** | Who wins or loses if this ships? |
| **Influence** | Who can fund, block, or redirect? |
| **Expertise** | Who holds domain or operational truth? |
| **Accountability** | Who owns runtime incidents and cost? |

Use a simple **power/interest grid**: high power + high interest = manage closely; high power + low interest = keep satisfied; low power + high interest = keep informed.

### Managing conflicting priorities

- **Name the conflict** — “Team A needs speed; Team B needs stability” is clearer than debating tools.
- **Anchor on outcomes** — revenue, risk reduction, regulatory deadline, customer SLA.
- **Offer options, not ultimatums** — two or three viable paths with cost, time, and risk labeled.
- **Escalate with data** — when deadlock persists, bring decision rights to the right sponsor with a recommendation.

### Executive communication

- **Lead with the decision or ask** — one sentence: what you need and by when.
- **Three layers** — outcome → approach → one proof point (metric, pilot, or reference customer).
- **Visuals over jargon** — one context diagram (C4 Level 1) often beats ten bullet acronyms.
- **Pre-wire** — informal 1:1s before the big room reduces surprises.

### Building consensus

- **Shared glossary** — agree what “done,” “platform,” and “MVP” mean.
- **Decision log** — link to ADRs in [[System Design — Governance & Documentation]].
- **Timebox disagreement** — diverge in workshop, converge with a dated decision and owner.

---

## Communication Craft

### Technical → non-technical

| Technique | Purpose |
| --- | --- |
| **Analogy** | Relate to money, time, or risk they already track |
| **Before/after** | Current pain vs future state in business terms |
| **Boundaries** | What the system will *not* do (prevents scope creep) |
| **Assumptions** | What must stay true for the design to hold |

Avoid implementation detail unless they ask; offer a **“deeper dive” appendix** for the curious.

### Writing Architecture Decision Records (ADRs)

- **Context** — forces and constraints
- **Decision** — what we chose
- **Consequences** — positive, negative, and follow-ups
- **Status** — proposed, accepted, superseded

ADRs are short, durable, and searchable — the antidote to “why did we pick Kafka?” six months later. Pairs with [[DB — Kafka]] vs [[DB — RabbitMQ]] style choices in the technical vault.

### Diagramming (C4, UML)

| Level | Audience | Shows |
| --- | --- | --- |
| **C4 Context** | Executives, PMs | System in the world |
| **C4 Container** | Tech leads | Major deployables (API, DB, queue) |
| **C4 Component** | Implementers | Modules inside a container |
| **UML (selective)** | Engineers | Sequences, states when behavior is subtle |

**Rule:** one diagram per question — “deployment,” “request flow,” “data ownership” are different drawings.

### Active listening and facilitation

- **Playback** — “What I heard is … did I miss anything?”
- **Parking lot** — capture tangents without losing the agenda
- **Round-robin** — quieter voices in cross-functional rooms
- **Breakouts** — deep technical disputes after the main session (same pattern as Daily Scrum “after-party” in Scrum Inbox notes)

---

## Negotiation (Communication Extension)

| Topic | Architect move |
| --- | --- |
| **Vendor contracts** | Tie SLAs and exit clauses to [[System Design — Economics & Performance]] |
| **Scope vs timeline** | Fixed date → flex scope; fixed scope → flex date; never hide both as fixed |
| **Technical debt vs speed** | Quantify interest: incident rate, lead time, migration cost |
| **Standards adoption** | Pilot + metrics + executive sponsor — see [[System Design — Leadership & Culture]] |

---

## Common Pitfalls

| Pitfall | Better habit |
| --- | --- |
| Designing in a vacuum | Stakeholder map in week one |
| Slides full of logos | One outcome metric per slide |
| Winning the argument | Winning alignment on the next step |
| No written decision | ADR within 48 hours of agreement |

---

## Related Notes

- [[System Design]]
- [[System Design — Governance & Documentation]]
- [[System Design — Leadership & Culture]]
- [[System Design — Delivery & Planning]]
- [[GCP]] — cloud narrative for executives

---

## Tags

#system-design #stakeholders #communication #adr #c4 #facilitation #negotiation
