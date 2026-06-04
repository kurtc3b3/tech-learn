**Key Points:**

- **CIA triad** — Confidentiality, Integrity, Availability; every control maps to at least one leg.
- **Security posture** — overall strength of defences at a point in time; improves with measurement and remediation.
- **Controls** — administrative, technical, and physical safeguards that reduce risk.
- **PII / SPII** — personal and sensitive personal data drive privacy law and breach obligations.
- **OWASP principles** — design habits for software; apply when building [[API - FastAPI]] services.

# Cybersecurity — Fundamentals & Controls

Part of [[Cybersecurity]]. Concept-only.

---

## CIA Triad (Security Controls Foundation)

| Pillar | Protects against | Examples |
| --- | --- | --- |
| **Confidentiality** | Unauthorized disclosure | Encryption, access control, need-to-know |
| **Integrity** | Unauthorized modification | Hashing, signing, change logs, validation |
| **Availability** | Disruption of access | Redundancy, backups, DDoS mitigation, capacity |

Trade-offs exist: stricter confidentiality can complicate availability (e.g. locked-down recovery).

---

## Security Posture

**Security posture** is the **current state** of an organization’s defences — policies, tooling, patching, monitoring, and culture — relative to its threats and compliance obligations.

| Signal of strong posture | Signal of weak posture |
| --- | --- |
| Inventory of assets and owners | Unknown shadow IT |
| Patch and vuln SLAs | End-of-life systems exposed |
| MFA on critical access | Shared admin passwords |
| Regular backups tested | Backups never restored in drills |
| SIEM or centralized logging | Logs only on local disks |

Posture is **not static** — it decays without maintenance.

---

## Types of Security Controls

| Type | Examples |
| --- | --- |
| **Administrative** | Policies, training, background checks, incident response plans |
| **Technical** | Firewalls, encryption, IAM, WAF, antivirus, IDS/IPS |
| **Physical** | Badges, locks, camera, secure disposal |

Controls are selected from **risk assessment** — [[Cybersecurity — Frameworks & Compliance]].

---

## PII and SPII

| Term | Meaning |
| --- | --- |
| **PII** | Information identifying an individual (name, email, IP in some contexts) |
| **SPII** | Sensitive subset — health, financial, biometric, government IDs, etc. |

Implications:

- **Minimize collection** — only store what the product needs
- **Purpose limitation** — document why data is held
- **Retention** — delete when no longer required
- **Breach notification** — legal timelines under GDPR and similar — [[Cybersecurity — Frameworks & Compliance]]

Connects to [[System Design — Governance & Documentation]] and database design [[ORM - SQLAlchemy]].

---

## OWASP Security Design Principles

Apply when designing APIs, services, and UIs:

| Principle | Practice |
| --- | --- |
| **Minimize attack surface** | Fewer exposed endpoints, disable unused features |
| **Least privilege** | Roles and service accounts scoped narrowly |
| **Defence in depth** | Multiple layers — network + app + data |
| **Separation of duties** | No single person can approve and execute critical actions |
| **Keep security simple** | Complex policy → misconfiguration |
| **Fix issues correctly** | Root cause, not checkbox patches |
| **Secure defaults** | Deny by default; opt-in to permissive settings |
| **Fail securely** | Errors must not leak secrets or bypass auth |
| **Don’t trust services** | Validate third-party callbacks and tokens |
| **Avoid security by obscurity** | Hiding URLs is not authentication |

Maps to [[API - FastAPI]] dependency injection, validation, and auth patterns.

---

## Programming & Secure Development (Concept)

Security in **programming** means:

- Input validation and output encoding (injection, XSS)
- Parameterized queries (SQL injection) — [[ORM - SQLAlchemy]]
- Secrets outside source control — [[GCP]] Secret Manager concept
- Dependency scanning (supply chain) — [[Cybersecurity — Threats & Attacks]]
- Security in CI/CD and code review

“Programming” as a **security tool category** means developers are part of the control set, not only operators.

---

## Asset Security (Domain Snapshot)

- **Inventory** — hardware, software, data, cloud resources
- **Classification** — public, internal, confidential, restricted
- **Ownership** — who patches, who approves access
- **Disposal** — secure wipe, key rotation on decommission

---

## Identity and Access Management (Concept)

- **Authentication** — who you are (MFA strengthens this)
- **Authorization** — what you may do (RBAC, ABAC)
- **Federation** — SSO, OAuth2/OIDC for [[API - FastAPI]]
- **Privileged access** — break-glass, session recording for admins

---

## Related Notes

- [[Cybersecurity]]
- [[Cybersecurity — Frameworks & Compliance]]
- [[Cybersecurity — Threats & Attacks]]
- [[API - FastAPI]]
- [[System Design — Governance & Documentation]]

---

## Tags

#cybersecurity #cia #owasp #pii #controls #secure-development #iam
