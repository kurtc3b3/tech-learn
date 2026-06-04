**Key Points:**

- **Cybersecurity protects confidentiality, integrity, and availability** — the CIA triad guides every control choice.
- **Risk is managed, not eliminated** — frameworks (NIST), compliance (GDPR, PCI), and audits align effort with impact.
- **People are a primary attack surface** — social engineering and insider threats matter as much as malware.
- **Defence in depth** — layers across network, cloud, application, and operations; see [[API - FastAPI]] and [[GCP]] for implementation context.
- **Concept-only** — no tool commands here; labs and operator skills link to [[Linux — Kali & Security Labs]] where appropriate.

# Cybersecurity — Overview & Security Knowledge Map

> **From scratch checklist:** [[Build a Cybersecurity Practice Path from Scratch]] · All roadmaps: [[README]]

## What is Cybersecurity (in this vault)?

**Cybersecurity** here means **protecting systems and data** from unauthorized access, disruption, and misuse — through controls, governance, detection, and response. These notes summarize terms and models from your Inbox reference for study, architecture reviews, and compliance conversations.

Typical outcomes:

- **Speak the language** — threat actors, controls, frameworks, compliance regimes
- **Classify incidents** — phishing vs supply chain vs network interception
- **Align product work** — OWASP principles, PII handling, secure defaults in [[API - FastAPI]]
- **Plan audits** — scope, risk assessment, mitigation, stakeholder communication
- **Connect to ops** — SIEM, playbooks, and [[System Design — Governance & Documentation]]

---

## Concept Map

| Theme | Note | Core question |
| --- | --- | --- |
| Foundations | [[Cybersecurity — Fundamentals & Controls]] | What are we protecting, and how? |
| Human threats | [[Cybersecurity — Social Engineering]] | How do attackers manipulate people? |
| Technical threats | [[Cybersecurity — Threats & Attacks]] | What can go wrong technically? |
| Rules & standards | [[Cybersecurity — Frameworks & Compliance]] | What must we follow, and how do we prove it? |
| Networks | [[Cybersecurity — Network Security]] | How does data move, and where is it exposed? |
| Detect & respond | [[Cybersecurity — Security Operations]] | How do we see attacks and react? |

---

## NIST Cybersecurity Framework (CSF) — Six Functions

| Function | Purpose |
| --- | --- |
| **Govern** | Strategy, roles, supply chain risk |
| **Identify** | Assets, risks, improvement |
| **Protect** | Safeguards and access control |
| **Detect** | Anomalies and events |
| **Respond** | Containment, analysis, communication |
| **Recover** | Restore capabilities and lessons learned |

Deep dive: [[Cybersecurity — Frameworks & Compliance]].

---

## Vault Connections (Build & Run Securely)

| Layer | Security lens | Vault link |
| --- | --- | --- |
| HTTP APIs | Auth, rate limits, validation | [[API - FastAPI]], [[API - FastAPI — Rate Limiting (SlowAPI)]] |
| Cloud | IAM, VPC, secrets | [[GCP]], [[System Design — Governance & Documentation]] |
| Containers | Namespaces, image supply chain | [[K8S]], [[Linux — Architecture]] |
| Data | PII, retention, encryption | [[Cybersecurity — Fundamentals & Controls]], [[ORM - SQLAlchemy]] |
| Monitoring | Metrics vs logs vs SIEM | [[DB — ELK]], [[DB — Prometheus & Grafana]] |
| Authorized testing | Labs only | [[Linux — Kali & Security Labs]] |
| AI systems | Prompt injection, model abuse | [[AI]] |

---

## Security Domains (Study Map)

Eight domains often used in professional certification study paths:

1. Security and risk management
2. Asset security
3. Security architecture and engineering
4. Communication and network security → [[Cybersecurity — Network Security]]
5. Identity and access management
6. Security assessment and testing
7. Security operations → [[Cybersecurity — Security Operations]]
8. Software development security → [[Cybersecurity — Fundamentals & Controls]]

---

## When to Use Which Note

| Situation | Open |
| --- | --- |
| Explain CIA to a team | [[Cybersecurity — Fundamentals & Controls]] |
| Phishing incident briefing | [[Cybersecurity — Social Engineering]] |
| Ransomware / supply chain headline | [[Cybersecurity — Threats & Attacks]] |
| Customer asks for SOC 2 / GDPR | [[Cybersecurity — Frameworks & Compliance]] |
| Design VPC / firewall rules | [[Cybersecurity — Network Security]] |
| SOC analyst workflow / SIEM | [[Cybersecurity — Security Operations]] |

---

## Recommended Learning Path

1. **Fundamentals & Controls** — CIA, OWASP principles, PII
2. **Social Engineering** — recognize manipulation
3. **Threats & Attacks** — malware, passwords, supply chain
4. **Network Security** — TCP/IP, firewalls, common network attacks
5. **Frameworks & Compliance** — NIST, GDPR, PCI, audit flow
6. **Security Operations** — SIEM, playbooks, incident lifecycle

---

## Related Notes

- [[Cybersecurity — Fundamentals & Controls]]
- [[Cybersecurity — Social Engineering]]
- [[Cybersecurity — Threats & Attacks]]
- [[Cybersecurity — Frameworks & Compliance]]
- [[Cybersecurity — Network Security]]
- [[Cybersecurity — Security Operations]]
- [[System Design — Governance & Documentation]]
- [[Linux — Kali & Security Labs]]
- [[GCP]]
- [[API - FastAPI]]
- [[AI]]

---

## Tags

#cybersecurity #cia #nist #compliance #threats #owasp #risk #governance
