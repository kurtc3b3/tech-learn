**Key Points:**

- **Technology follows strategy** — every major stack choice should trace to a business goal, not a conference keynote.
- **Short vs long term** — ship now with explicit debt register; invest in platforms when repetition proves pain.
- **Technology radar** — adopt, trial, assess, hold — keeps hype out of production.
- **Vendor evaluation** — fit, viability, lock-in, and operability matter more than feature checklists.
- **Standards accelerate teams** — patterns for [[API - FastAPI]], [[ORM - SQLAlchemy]], and observability reduce rework when adoption is voluntary and measured.

# System Design — Strategy & Technology

Part of [[System Design]]. Concept-only.

---

## Strategic Thinking

### Aligning tech with business goals

| Business goal | Architectural lever |
| --- | --- |
| **Time to market** | Buy/SaaS, modular monolith, managed [[GCP]] services |
| **Cost leadership** | FinOps, shared platforms, batch over realtime |
| **Risk & compliance** | Strong boundaries, audit logs, regional data stores |
| **Innovation** | Sandboxed pilots, feature flags, measurable experiments |

Ask in every review: **“If this ships, which OKR moves?”** If none, defer or reframe.

### Long-term vs short-term trade-offs

| Horizon | Typical choice | Record in |
| --- | --- | --- |
| **Now (0–3 months)** | Tactical integration, manual ops acceptable | Sprint goals |
| **Next (3–12 months)** | Extract services, harden SLOs | Roadmap + ADR |
| **Later (12+ months)** | Platform consolidation, multi-region | Strategy doc + radar |

**Explicit technical debt:** what you borrowed, interest (incidents, velocity), and paydown plan — pairs with [[System Design — Delivery & Planning]].

### Technology radar

Inspired by ThoughtWorks-style radars — four rings:

| Ring | Meaning |
| --- | --- |
| **Adopt** | Default for new work (e.g., [[Linting — Ruff]], [[ORM - SQLAlchemy]] in this vault) |
| **Trial** | Pilot on non-critical path with success criteria |
| **Assess** | Learning only; not production |
| **Hold** | Do not expand; migrate off |

Refresh **quarterly** with engineering + product + security input.

### Vendor evaluation and selection

| Criterion | Questions |
| --- | --- |
| **Functional fit** | Covers 80% without custom glue? |
| **Operability** | APIs, exports, status page, support tier |
| **Security** | Certifications, data residency, subprocessors |
| **Commercial** | Pricing model, renewal uplift, exit cost |
| **Ecosystem** | Hiring pool, community, partner integrations |
| **Strategic** | Vendor roadmap vs your three-year bets |

Score vendors; **weight** criteria by scenario (startup vs regulated enterprise).

---

## Driving Standards and Patterns

Architects **influence** more than **mandate**:

- **Golden paths** — blessed way to build a service (auth, logging, deploy)
- **Reference implementations** — thin exemplar on [[API - FastAPI]] + [[K8S]]
- **Guardrails** — CI policies via [[Linting]]; optional escape hatch with ADR
- **Communities of practice** — office hours, not only review boards

Adoption metrics: % new services on golden path, mean time to first deploy, incident rate.

---

## Technology Choices in This Vault (Architect View)

| Domain | Hub | Strategic question |
| --- | --- | --- |
| Compute | [[GCP]], [[K8S]] | Managed vs control? Multi-cloud lock-in? |
| Data | [[DB]], [[ORM - SQLAlchemy]] | Source of truth, eventing, observability |
| APIs | [[API - FastAPI]], [[Web]] | Public vs internal, versioning, BFF |
| ML | [[Machine Learning]], [[AI]] | Model risk, cost of inference, data rights |
| Async | [[Processing]] | Sync vs queue vs stream — operational model |
| Quality | [[Linting]], [[Unit Testing - pytest]] | Definition of done for teams |

Deep implementation stays in **Codes** and **Commands** notes; strategy picks **which hub** to invest in.

---

## Trend Awareness (Without Hype)

| Signal | Healthy response |
| --- | --- |
| New LLM capability | Trial in [[AI]]; production only with eval + cost model |
| “Everyone is microservices” | Check team topology — modular monolith may fit |
| Zero-trust buzzword | Map to identity, network, and secret patterns on [[GCP]] |
| “We need Kafka” | Event volume and consumer count? See [[DB — Kafka]] decision in [[DB]] |

---

## Related Notes

- [[System Design]]
- [[System Design — Economics & Performance]]
- [[System Design — Leadership & Culture]]
- [[System Design — Governance & Documentation]]
- [[GCP]]
- [[K8S]]
- [[DB]]
- [[AI]]
- [[Machine Learning]]

---

## Tags

#system-design #strategy #technology-radar #vendor #standards #technical-debt #roadmap
