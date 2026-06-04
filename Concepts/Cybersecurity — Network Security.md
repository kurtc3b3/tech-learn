**Key Points:**

- **TCP/IP model** — four layers from link to application; maps loosely to OSI seven layers.
- **Security protocols** — HTTPS, SSH, SFTP prefer encryption and authentication over cleartext (Telnet, FTP).
- **Perimeter and segmentation** — firewalls, VPNs, IDS/IPS reduce exposure and detect abuse.
- **Wireless needs WPA2/WPA3** — WEP is obsolete.
- **Network attacks** — sniffing, spoofing, on-path, DoS — tie to monitoring in [[Cybersecurity — Security Operations]].

# Cybersecurity — Network Security

Part of [[Cybersecurity]]. Concept-only. Operator commands: [[Linux]], [[Commands/Linux — Networking]].

---

## TCP/IP Model (Four Layers)

| Layer | Examples | OSI mapping (approx.) |
| --- | --- | --- |
| **Network access** | Ethernet, Wi-Fi | Physical + Data link |
| **Internet** | IPv4, IPv6, ICMP | Network |
| **Transport** | TCP, UDP | Transport |
| **Application** | HTTP, DNS, SMTP, SSH | Session + Presentation + Application |

Understanding layers helps **pinpoint where controls apply** (firewall at network/transport, WAF at application).

---

## IPv4 Packet (Header Concepts)

Fields your reference lists — useful for **forensics and study**:

| Field | Role |
| --- | --- |
| **Version (VER)** | IP version |
| **IHL / HLEN** | Header length |
| **ToS** | QoS hints |
| **Total length** | Packet size |
| **Identification, flags, offset** | Fragmentation |
| **TTL** | Hop limit |
| **Protocol** | Upper layer (e.g. TCP=6, UDP=17) |
| **Header checksum** | Header integrity |
| **Source / destination IP** | Endpoints |
| **Options** | Rare extensions |

---

## Protocol Categories

### Communication

| Protocol | Notes |
| --- | --- |
| **TCP** | Reliable, connection-oriented |
| **UDP** | Fast, no guarantee |
| **HTTP** | Web — use **HTTPS** in production |
| **DNS** | Name resolution — DNSSEC where appropriate |

### Management

| Protocol | Notes |
| --- | --- |
| **SNMP** | Device monitoring — secure versions/community strings |
| **ICMP** | Ping, diagnostics — can be abused in floods |

### Security-oriented

| Protocol | Notes |
| --- | --- |
| **HTTPS** | HTTP over TLS |
| **SFTP** | Secure file transfer |
| **SSH** | Encrypted remote shell |

### Other (know risk)

| Protocol | Risk note |
| --- | --- |
| **DHCP** | Rogue server attacks on open LANs |
| **ARP** | Spoofing on local segments |
| **Telnet** | Cleartext — avoid |
| **FTP** | Cleartext — prefer SFTP |
| **SMTP / POP / IMAP** | Use TLS variants |

---

## Wireless Security

| Standard | Status |
| --- | --- |
| **WEP** | Broken — never use |
| **WPA** | Legacy |
| **WPA2** | Widely deployed |
| **WPA3** | Preferred for new deployments |

---

## Network Devices (Roles)

| Device | Role |
| --- | --- |
| **Firewall** | Permit/deny traffic by policy |
| **Router** | Routes between networks |
| **Switch** | Layer 2 forwarding (smart switches add ACLs) |
| **Hub** | Legacy — broadcasts all traffic |
| **Modem / WAP** | Internet edge, wireless access |

---

## Firewalls

| Type | Behavior |
| --- | --- |
| **Stateless** | Each packet judged alone |
| **Stateful** | Tracks connections — more common |

**Proxy servers** — clients connect to proxy; can inspect, filter, and log HTTP.

---

## VPN (Concept)

| Type | Use |
| --- | --- |
| **Remote access VPN** | User to corporate network |
| **Site-to-site VPN** | Network to network |

| Technology | Notes |
| --- | --- |
| **IPsec** | Traditional site-to-site |
| **WireGuard** | Modern, simpler codebase — common on Linux |

Cloud: private connectivity via [[GCP]] VPC; [[K8S]] network policies for pod segmentation.

---

## Network Security Applications

| Tool class | Role |
| --- | --- |
| **Firewall** | Policy enforcement |
| **IDS** | Intrusion **Detection** — alerts |
| **IPS** | Intrusion **Prevention** — block inline |
| **SIEM** | Correlate logs — [[Cybersecurity — Security Operations]] |

---

## Network Attacks

### Interception

| Attack | Idea |
| --- | --- |
| **Packet sniffing** | Capture traffic on shared media — mitigated by encryption |
| **IP spoofing** | Forge source address |
| **On-path (MitM)** | Attacker relays or modifies traffic |
| **Smurf attack** | Amplified ICMP flood (legacy) |
| **DoS / DDoS** | Availability attack — floods or exhausts resources |

### Other

| Attack | Idea |
| --- | --- |
| **Backdoor** | Hidden access channel — [[Cybersecurity — Threats & Attacks]] |

Defence: TLS everywhere, segmentation, rate limiting, DDoS protection at edge, monitoring.

---

## Cloud Network Security (Concept)

- **Security groups / firewall rules** — least open ports
- **Private subnets** — no direct internet on databases
- **IAM** — who can change network rules
- **Service mesh / mTLS** — [[K8S]] east-west traffic

See [[GCP]], [[K8S]].

---

## Related Notes

- [[Cybersecurity]]
- [[Cybersecurity — Threats & Attacks]]
- [[Cybersecurity — Security Operations]]
- [[Linux — Architecture]]
- [[Load Testing]] — availability testing vs malicious DoS

---

## Tags

#cybersecurity #network #tcpip #firewall #vpn #ids #ips #wireless #ddos
