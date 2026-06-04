**Key Points:**

- **Social engineering targets people**, not software bugs — urgency, trust, and authority bypass technical controls.
- **Phishing has many channels** — email (BEC, spear, whaling), voice (vishing), SMS (smishing), social media.
- **Physical and USB attacks** — tailgating, malicious cables, baited drives — still common in enterprises.
- **Defence is training + process** — verify out-of-band, report suspicious messages, least privilege limits blast radius.
- **Overlaps [[Cybersecurity — Threats & Attacks]]** — listed here in depth because your reference emphasizes human factors.

# Cybersecurity — Social Engineering

Part of [[Cybersecurity]]. Concept-only.

---

## What is Social Engineering?

**Social engineering** manipulates people into **revealing information**, **running harmful actions**, or **bypassing procedures** — often without exploiting a CVE.

---

## Phishing Variants

| Type | Targeting | Channel |
| --- | --- | --- |
| **Phishing (bulk)** | Many users | Generic email |
| **Spear phishing** | Specific person/role | Tailored email |
| **Whaling** | Executives | High-stakes fraud |
| **Business Email Compromise (BEC)** | Finance, payroll | Impersonated CEO/vendor |
| **Vishing** | Call centers, help desks | Voice |
| **Smishing** | Mobile users | SMS |
| **Social media phishing** | Brand impersonation | LinkedIn, X, etc. |

**Watering hole** — compromise a site the target group visits, then infect visitors.

---

## Other Social Engineering Techniques

| Technique | Description |
| --- | --- |
| **USB baiting** | Leave infected drives where victims will plug them in |
| **Physical social engineering** | Tailgating, impersonating IT, shoulder surfing |
| **Malicious USB cable** | Hardware that acts as attack device when plugged in |
| **Pretexting** | Fabricated scenario to extract data (“audit”, “new vendor”) |

Authorized testing context: [[Linux — Kali & Security Labs]] — **only with permission**.

---

## Principles Attackers Exploit (Influence)

| Principle | How it is used |
| --- | --- |
| **Authority** | “I’m from IT / legal / the CEO” |
| **Intimidation** | Threats of consequences if you don’t comply |
| **Consensus / social proof** | “Everyone else already updated their password” |
| **Scarcity** | “Offer expires in one hour” |
| **Familiarity** | Fake colleague or repeated contact |
| **Trust** | Relationship built before the ask |
| **Urgency** | “Act now or account will be closed” |

Training should teach staff to **pause** when multiple principles appear together.

---

## Defence Strategies

| Control | Detail |
| --- | --- |
| **Security awareness** | Regular, realistic training — not annual checkbox |
| **Phishing simulations** | Measure click rates; coach, don’t shame |
| **Out-of-band verification** | Confirm wire transfers and credential resets by known channel |
| **Email authentication** | SPF, DKIM, DMARC reduce spoofing (with [[Cybersecurity — Network Security]]) |
| **Least privilege** | Limits damage if credentials are stolen |
| **Report button** | Easy path to security team |

---

## Relationship to “Attack Types”

Social engineering appears under:

- Common attacks → Phishing, Social Engineering
- Attack types → Social engineering attack (full list in inbox)
- Password attacks often **follow** successful phishing (credential harvest)

See also [[Cybersecurity — Threats & Attacks]].

---

## Related Notes

- [[Cybersecurity]]
- [[Cybersecurity — Threats & Attacks]]
- [[Cybersecurity — Fundamentals & Controls]]
- [[Cybersecurity — Security Operations]]
- [[System Design — Stakeholders & Communication]]

---

## Tags

#cybersecurity #social-engineering #phishing #bec #vishing #awareness
