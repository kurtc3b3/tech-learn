## What & When

**Kali tool reference** by category — command names and purpose only. **Authorized labs.** Concept: [[Linux — Kali & Security Labs]]. Setup: [[Commands/Linux — Kali in Docker]].

---

## Information gathering

| Tool | Purpose |
| --- | --- |
| `nmap` | Port / service scan |
| `netdiscover` | LAN discovery |
| `dnsenum` | DNS enumeration |
| `whois` | Domain registration |
| `theHarvester` | OSINT emails/hosts |
| `amass` | Subdomain enumeration |

```bash
nmap -sV -T4 target.example.com
```

---

## Web application analysis

| Tool | Purpose |
| --- | --- |
| `burpsuite` | Proxy / scanner (GUI) |
| `nikto` | Web server scan |
| `dirb` | Directory brute force |
| `gobuster` | Dir/DNS busting |
| `wpscan` | WordPress |

```bash
gobuster dir -u http://target -w /usr/share/wordlists/dirb/common.txt
```

---

## Password / credentials (authorized)

| Tool | Purpose |
| --- | --- |
| `hydra` | Online login testing |
| `john` | Offline hash cracking |
| `hashcat` | GPU cracking |
| `crunch` | Wordlist generation |
| `cewl` | Site-based wordlist |

---

## Wireless (usually VM + adapter)

| Tool | Purpose |
| --- | --- |
| `airmon-ng` | Monitor mode |
| `airodump-ng` | Capture |
| `aircrack-ng` | Crack captures |

---

## Vulnerability analysis

| Tool | Purpose |
| --- | --- |
| `nuclei` | Template scans |
| `nikto` | Web issues |
| `openvas` | Broad scanning (heavy) |

---

## Exploitation frameworks (labs)

| Tool | Purpose |
| --- | --- |
| `msfconsole` | Metasploit console |
| `searchsploit` | Exploit-DB search |

```bash
searchsploit apache 2.4
```

---

## Sniffing & traffic

| Tool | Purpose |
| --- | --- |
| `tcpdump` | CLI capture |
| `wireshark` | GUI analysis |
| `bettercap` | Modern MITM framework |

```bash
sudo tcpdump -i eth0 port 80 -w capture.pcap
```

---

## Post-exploitation (research)

| Tool | Purpose |
| --- | --- |
| `linpeas` | Linux priv esc audit script |
| `pspy` | Process snooping |

---

## Forensics

| Tool | Purpose |
| --- | --- |
| `binwalk` | Firmware / embedded |
| `foremost` | File carving |
| `exiftool` | Metadata |

---

## Kali system commands

```bash
cat /etc/os-release
sudo apt update && sudo apt full-upgrade -y
kali-tweaks        # if installed
gzip -d /usr/share/wordlists/rockyou.txt.gz
```

---

## Wordlists

| Path | Note |
| --- | --- |
| `/usr/share/wordlists/rockyou.txt` | Common passwords |
| `/usr/share/seclists` | SecLists pack |

---

## Related Notes

- [[Linux — Kali & Security Labs]]
- [[Commands/Linux — Kali in Docker]]
- [[Linux]]
- [[Browser Automation]]

---

## Tags

#linux #kali #security #nmap #metasploit #tools
