**Key Points:**

- **Sprint Planning answers two questions** — what can we deliver this Sprint, and how will we do it?
- **Sprint Goal first** — one meaningful objective before selecting backlog items.
- **Team pulls work** — based on capacity and historical throughput, not top-down assignment.
- **User stories express value** — INVEST criteria and acceptance criteria reduce ambiguity.
- **Relative estimation** — story points compare effort; see [[Scrum — Metrics]] for velocity use.

# Scrum — Sprint Planning & User Stories

Part of [[Scrum]]. Concept-only.

---

## Sprint Planning

### Two questions

1. **Why is this Sprint valuable?** → **Sprint Goal** (Product Owner proposes; team aligns)
2. **What can be Done and how?** → Selected Product Backlog items + plan in Sprint Backlog

### Typical flow

| Step | Who | Outcome |
| --- | --- | --- |
| Set Sprint Goal | Product Owner + Developers | Single objective, e.g. “Users can register and log in securely” |
| Select items | Developers pull from top of ordered backlog | Items they believe are completable |
| Break down work | Developers | Tasks usually under one day each |
| Confirm plan | Whole Scrum Team | Sprint Backlog + Sprint Goal committed |

### Role responsibilities

| Role | In Planning |
| --- | --- |
| **Product Owner** | Clarifies items, trade-offs, acceptance |
| **Scrum Master** | Facilitates, timebox, removes confusion |
| **Developers** | Estimate, decompose, commit to the plan |

**Pull, not push:** Managers and Scrum Masters do not assign tasks to Developers.

### Inputs that make Planning fast

- **2 Sprints of ready items** — [[Scrum — Backlog Refinement]], [[Scrum — Product Backlog#Definition of Ready]]
- **Stable velocity range** — [[Scrum — Metrics]]
- **Clear DoD** — [[Scrum — Framework]]

---

## User Stories

A **user story** is a short description of need from a **user’s perspective** — a placeholder for conversation, not a full specification.

### Classic format

> **As a** [type of user],  
> **I want** [action],  
> **so that** [benefit].

Example: *As a shopper, I want to save items to a wishlist, so that I can buy them later.*

### INVEST criteria

| Letter | Meaning |
| --- | --- |
| **I**ndependent | Minimal hard dependency on other stories |
| **N**egotiable | Details emerge in refinement, not frozen upfront |
| **V**aluable | Clear user or business benefit |
| **E**stimable | Team can size with reasonable confidence |
| **S**mall | Completable within one Sprint |
| **T**estable | Verification is explicit |

### Acceptance criteria

Conditions that must be true for the story to count as Done. **Given / When / Then** works well:

- **Given** I am logged in  
- **When** I add a product to the wishlist  
- **Then** it appears on my Wishlist page  

Removes “done means different things to different people.”

### Story points (relative sizing)

Teams estimate **relative effort**, often with a Fibonacci scale (1, 2, 3, 5, 8, 13, 21). Larger gaps reflect growing uncertainty — a 13 is not “slightly bigger” than an 8.

| Illustrative story | Points | Why |
| --- | --- | --- |
| Button label change | 1 | Trivial |
| Search bar UI only | 3 | Simple, some design |
| Login system | 8 | Auth, validation, sessions |
| Payment gateway integration | 13 | Many unknowns |

**Planning Poker** (simultaneous reveal, then discuss divergence) is a common refinement technique — see [[Scrum — Backlog Refinement]].

---

## Hierarchy: Epic → Story → Task

```
Epic (large initiative)
└── Feature / story group
    └── User story
        └── Tasks (Sprint Backlog)
```

Split epics before Sprint Planning — see splitting patterns in [[Scrum — Backlog Refinement]].

---

## Common Mistakes

| Mistake | Better approach |
| --- | --- |
| Technical task disguised as story | Reframe as user/business outcome |
| Epic-sized “story” | Split by workflow, role, or happy path |
| No acceptance criteria | Define before Sprint starts |
| Committing 100% of capacity | Leave room for support, defects, meetings |

---

## Related Notes

- [[Scrum]]
- [[Scrum — Framework]]
- [[Scrum — Product Backlog]]
- [[Scrum — Backlog Refinement]]
- [[Scrum — Metrics]]

---

## Tags

#scrum #sprint-planning #user-stories #invest #acceptance-criteria #story-points
