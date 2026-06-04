**Key Points:**

- **Threat actors** range from opportunistic criminals to APTs and insiders — motive and patience differ.
- **Malware families** — viruses, worms, ransomware, spyware — delivery often starts with phishing.
- **Password attacks** — brute force and rainbow tables; defeated by MFA, salting, and lockout.
- **Supply chain and crypto attacks** — trust in dependencies and algorithms can fail at scale.
- **Adversarial AI** — emerging risk for [[AI]] systems (prompt injection, data poisoning).

# Cybersecurity — Threats & Attacks

Part of [[Cybersecurity]]. Concept-only.

---

## Threat Actors

| Actor | Characteristics |
| --- | --- |
| **Advanced Persistent Threat (APT)** | Nation-state or well-funded; long dwell time, targeted |
| **Insider (internal threat)** | Employee, contractor; malicious or negligent |
| **Hacktivists** | Ideological goals; defacement, DDoS, leaks |
| **Organized crime** | Ransomware, fraud, credential markets |
| **Opportunistic attackers** | Mass scanning, default passwords |

**Insider threat** is especially dangerous because **trust and access already exist** — controls: segregation, logging, least privilege.

---

## Malware

| Type | Behavior |
| --- | --- |
| **Virus** | Needs host file; spreads when executed |
| **Worm** | Self-propagates across network |
| **Ransomware** | Encrypts or exfiltrates; demands payment |
| **Spyware** | Covert collection of activity/credentials |

Defence: patching, EDR/antivirus, backups, network segmentation, user training — [[Cybersecurity — Social Engineering]].

---

## Password Attacks

| Method | Idea | Mitigation |
| --- | --- | --- |
| **Brute force** | Try many passwords | Rate limits — [[API - FastAPI — Rate Limiting (SlowAPI)]], lockout |
| **Rainbow table** | Precomputed hash lookups | Salted hashes, strong algorithms |
| **Credential stuffing** | Reuse leaked pairs | MFA, breach detection |

---

## Attack Type Catalog (from reference)

Beyond social engineering ([[Cybersecurity — Social Engineering]]):

| Category | Examples |
| --- | --- |
| **Physical** | Malicious USB cable, device theft |
| **Supply chain** | Compromised library, vendor, or update channel |
| **Cryptographic** | Birthday, collision, downgrade attacks on weak crypto |
| **Adversarial AI** | Manipulated inputs to ML/LLM systems — [[AI]] |
| **Network** | Packet sniffing, spoofing — [[Cybersecurity — Network Security]] |

### Cryptographic attacks (concept)

| Attack | Idea |
| --- | --- |
| **Birthday** | Collision finding faster than brute force for hashes |
| **Collision** | Two inputs, same hash — breaks integrity assumptions |
| **Downgrade** | Force weaker cipher/protocol (e.g. legacy TLS) |

Use **current algorithms and protocols**; disable obsolete options.

### Supply chain attack

Trust is placed in **vendors, packages, and CI artifacts**. One compromised dependency affects many customers. Mitigate with pinning, signing, SBOM review, and least privilege in build pipelines.

### Adversarial artificial intelligence

Risks include **prompt injection**, **training data poisoning**, and **model extraction**. Treat LLM apps as untrusted input sources — [[AI]], [[Cybersecurity — Fundamentals & Controls]].

---

## Common Attacks (Summary Table)

| Attack | Entry |
| --- | --- |
| Phishing / BEC / vishing / smishing | Human |
| Malware | Email, drive-by, USB |
| Social engineering | Human + sometimes web |
| Password attack | Exposed login |
| Physical | On-site, hardware |
| Supply chain | Build/deploy |
| Crypto weakness | Protocol/config |

---

## Backdoor Attacks

A **backdoor** is persistent **unauthorized access** — planted malware, stolen keys, or hidden admin accounts. Detection: anomaly monitoring, integrity checks, code review — [[Cybersecurity — Security Operations]].

---

## Related Notes

- [[Cybersecurity]]
- [[Cybersecurity — Social Engineering]]
- [[Cybersecurity — Network Security]]
- [[Cybersecurity — Security Operations]]
- [[AI]]
- [[API - FastAPI]]

---

## Tags

#cybersecurity #threats #malware #ransomware #apt #insider #supply-chain #passwords
