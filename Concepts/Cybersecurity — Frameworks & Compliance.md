**Key Points:**

- **Frameworks organize security work** — NIST CSF for outcomes; NIST RMF for federal-style lifecycle.
- **Compliance is mandatory where applicable** — GDPR, PCI DSS, HIPAA, SOC 2, and sector rules drive controls.
- **Audits prove posture** — scope, risk assessment, evidence, mitigation, stakeholder reporting.
- **Compliance ≠ full security** — meeting checkboxes without risk context leaves gaps.
- **Architects map controls to design** — links to [[System Design — Governance & Documentation]].

# Cybersecurity — Frameworks & Compliance

Part of [[Cybersecurity]]. Concept-only.

---

## Security Frameworks (Why They Exist)

**Frameworks** provide **shared vocabulary and processes** so teams identify, protect, detect, respond, and recover consistently — especially at scale or under regulation.

| Framework | Focus |
| --- | --- |
| **NIST Cybersecurity Framework (CSF)** | Outcomes: Govern, Identify, Protect, Detect, Respond, Recover |
| **NIST Risk Management Framework (RMF)** | Formal risk lifecycle for systems (often US federal) |

---

## NIST CSF — Six Core Functions

| Function | Summary |
| --- | --- |
| **Govern** | Policies, roles, legal, supply chain |
| **Identify** | Asset management, risk assessment, improvement |
| **Protect** | Access control, awareness, data security, maintenance |
| **Detect** | Monitoring, anomaly detection |
| **Respond** | Analysis, containment, communication |
| **Recover** | Recovery planning, improvements |

Use CSF to **structure gap analysis** and roadmaps — not as a single checklist.

---

## NIST RMF — Seven Steps

| Step | Activity |
| --- | --- |
| **Prepare** | Organization-wide risk management context |
| **Categorize** | System impact level (e.g. confidentiality/integrity/availability tier) |
| **Select** | Choose controls baseline |
| **Implement** | Apply controls |
| **Assess** | Verify control effectiveness |
| **Authorize** | Risk acceptance decision by accountable official |
| **Monitor** | Ongoing assessment and change |

RMF is **heavier** than CSF — common in regulated US government systems.

---

## Compliance Regimes (Reference List)

| Regime | Typical scope |
| --- | --- |
| **GDPR** | EU personal data — rights, breach notification, DPA |
| **PCI DSS** | Payment card data handling |
| **HIPAA** | US health information (PHI) |
| **SOC 2 Type I / II** | Service organization controls (trust services criteria) |
| **ISO 27001** | ISMS certification (international) |
| **FedRAMP** | US federal cloud authorization |
| **FERC-NERC** | North American electric reliability / energy sector |
| **CIS Controls** | Prioritized technical safeguards (CIS®) |

**Which apply** depends on industry, geography, and customers — legal counsel confirms obligations.

Overlap with [[System Design — Governance & Documentation]] (GDPR, SOC2 mentioned for architects).

---

## Compliance vs Security Posture

| Compliance | Security |
| --- | --- |
| Demonstrates meeting **defined** requirements | Reduces **actual** risk |
| Audit evidence, contracts | Threat-informed defence |
| Can be minimum viable | Should exceed minimum where justified |

Strong teams **use compliance as a floor**, not a ceiling.

---

## Security Audit Checklist (Process)

| Step | Action |
| --- | --- |
| 1 | **Identify scope** — systems, data, time window, standards |
| 2 | **Risk assessment** — threats, likelihood, impact |
| 3 | **Conduct audit** — interviews, config review, sampling |
| 4 | **Mitigation plan** — prioritized fixes, owners, dates |
| 5 | **Communicate results** — executives, engineering, legal |

Connects to [[System Design — Stakeholders & Communication]] for reporting.

---

## Security and Risk Management (Domain)

- **Policies and standards**
- **Risk assessment methodologies**
- **Business continuity and disaster recovery**
- **Legal and regulatory awareness**
- **Security awareness program**

---

## Related Notes

- [[Cybersecurity]]
- [[Cybersecurity — Fundamentals & Controls]]
- [[Cybersecurity — Security Operations]]
- [[System Design — Governance & Documentation]]
- [[GCP]]
- [[System Design — Economics & Performance]] — cost of compliance

---

## Tags

#cybersecurity #nist #gdpr #pci #hipaa #soc2 #compliance #audit #rmf
