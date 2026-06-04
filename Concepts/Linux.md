**Key Points:**

- **Linux is the default server OS** for [[API - FastAPI]], [[K8S]] nodes, and [[Commands/CLI — Docker & Compose]] hosts — know files, processes, and networking at the shell.
- **Concept vs Commands** — this hub explains *how Linux is structured*; copy-paste command tables live under **Commands/Linux —**.
- **Bash scripting** automates deploys, log parsing, and cron jobs — patterns in [[Linux — Shell Scripting]], syntax in [[Commands/Linux — Bash Scripting]].
- **Containers share the kernel** — namespaces and cgroups ([[Linux — Architecture]]) underpin [[K8S]] and Docker.
- **Kali is a lab distro** — authorized testing only; see [[Linux — Kali & Security Labs]] and [[Commands/Linux — Kali in Docker]].

# Linux — Overview & Operator Stack

## What is Linux (in this vault)?

**Linux** here means **day-to-day operator skills** on Debian/Ubuntu-style servers and dev machines: navigate the filesystem, manage permissions, inspect processes, tune networking, schedule jobs, and write small Bash scripts. Security-lab content (Kali) is optional and scoped to **ethical, authorized** use.

Typical outcomes:

- **SSH into a VM** and debug a stuck [[API - FastAPI]] service
- **Read logs** with `journalctl` and `tail -f`
- **Check disk and memory** before an incident escalates
- **Script** repeatable admin tasks with `set -euo pipefail`
- **Understand** why Docker/Kubernetes isolation works

---

## Concept Map

| Theme | Concept note | Commands reference |
| --- | --- | --- |
| Layers & boot | [[Linux — Architecture]] | [[Commands/Linux — Processes & Services]] |
| Shell habits | [[Linux — Shell Scripting]] | [[Commands/Linux — Bash Scripting]] |
| Security labs | [[Linux — Kali & Security Labs]] | [[Commands/Linux — Kali Tools]], [[Commands/Linux — Kali in Docker]] |
| Daily operations | *(this hub)* | See command index below |

---

## Commands Index

| Area | Note |
| --- | --- |
| Files, search, pipes | [[Commands/Linux — Essentials]] |
| Links, rsync, ACLs | [[Commands/Linux — Files Advanced]] |
| sed, awk, tmux, logs | [[Commands/Linux — Text & Sessions]] |
| chmod, users, sudo | [[Commands/Linux — Permissions & Users]] |
| ps, systemd, cron | [[Commands/Linux — Processes & Services]] |
| ip, curl, DNS | [[Commands/Linux — Networking]] |
| apt, tar, compression | [[Commands/Linux — Packages & Archives]] |
| mount, df, lsblk | [[Commands/Linux — Disk & Storage]] |
| Bash syntax | [[Commands/Linux — Bash Scripting]] |

---

## macOS Developers (Homebrew)

Many vault developers use **macOS** locally. Install common operator tools via Homebrew; on the server use `apt`.

```bash
brew install tmux htop watch gnu-sed
```

GNU variants help when tutorials assume Linux flags. Remote work still happens on **Linux VMs** or [[K8S]] nodes — practice commands there for prod parity.

---

## Linux in the Broader Vault

| Concern | Linux skill | Vault link |
| --- | --- | --- |
| Run API locally | Files, processes, ports | [[API - FastAPI]], [[Commands/CLI — Docker & Compose]] |
| Container host | cgroups, namespaces | [[Linux — Architecture]], [[K8S]] |
| Load test target | `ss`, logs | [[Load Testing]] |
| CI runner | apt, bash scripts | [[Commands/CLI — Git & GitHub]] |
| Metrics node | disk, memory | [[DB — Prometheus & Grafana]] |

---

## When to Use Which Note

| Situation | Open |
| --- | --- |
| "How is Linux structured?" | [[Linux — Architecture]] |
| Write a deploy script | [[Linux — Shell Scripting]] → [[Commands/Linux — Bash Scripting]] |
| Find a file / grep logs | [[Commands/Linux — Essentials]], [[Commands/Linux — Text & Sessions]] |
| Permission denied | [[Commands/Linux — Permissions & Users]] |
| Service won't start | [[Commands/Linux — Processes & Services]] |
| Port already in use | [[Commands/Linux — Networking]] |
| Disk full | [[Commands/Linux — Disk & Storage]] |
| Pentest lab (authorized) | [[Linux — Kali & Security Labs]] |

---

## Recommended Learning Path

1. **Essentials** — navigation, viewing files, grep, pipes
2. **Permissions & Users** — `chmod`, `sudo`, ownership
3. **Processes & Services** — `ps`, `systemctl`, `journalctl`
4. **Networking** — `ip`, `ss`, `curl`
5. **Architecture** — FHS, boot, containers foundation
6. **Bash Scripting** — safe scripts for automation
7. **Kali** (optional) — only with legal scope and isolated labs

---

## Related Notes

### Concepts

- [[Linux — Architecture]]
- [[Linux — Shell Scripting]]
- [[Linux — Kali & Security Labs]]

### Commands

- [[Commands/Linux — Essentials]]
- [[Commands/Linux — Files Advanced]]
- [[Commands/Linux — Text & Sessions]]
- [[Commands/Linux — Permissions & Users]]
- [[Commands/Linux — Processes & Services]]
- [[Commands/Linux — Networking]]
- [[Commands/Linux — Packages & Archives]]
- [[Commands/Linux — Disk & Storage]]
- [[Commands/Linux — Bash Scripting]]
- [[Commands/Linux — Kali in Docker]]
- [[Commands/Linux — Kali Tools]]

### Connected

- [[CLI]]
- [[K8S]]
- [[Commands/CLI — Docker & Compose]]
- [[Commands/CLI — Git & GitHub]]
- [[API - FastAPI]]
- [[Python Development]]

---

## Tags

#linux #bash #sysadmin #devops #shell #systemd #docker #k8s
