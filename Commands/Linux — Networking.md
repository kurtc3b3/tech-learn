## What & When

**Network inspection and HTTP clients** on Linux. Overview: [[Linux]]. Load testing: [[Load Testing]].

---

## Install (macOS)

```bash
brew install curl wget bind   # dig/nslookup via bind
```

---

## Interfaces & routing

| Command | Description |
| --- | --- |
| `ip a` | Addresses |
| `ip link` | Interfaces |
| `ip r` | Routing table |
| `hostname -I` | IPs (legacy script) |

```bash
ip addr show eth0
```

---

## Connectivity

```bash
ping -c 4 host
traceroute host
# or: tracepath host
```

---

## DNS

```bash
nslookup example.com
dig example.com +short
host example.com
```

---

## Sockets & ports

| Command | Description |
| --- | --- |
| `ss -tulpn` | Listening TCP/UDP + process |
| `ss -tan` | All TCP |
| `netstat -tulpn` | Legacy (if installed) |

```bash
ss -tulpn | grep 8000
```

---

## HTTP clients

```bash
curl -i http://localhost:8000/health
curl -X POST -H "Content-Type: application/json" \
  -d '{"email":"a@b.com"}' http://localhost:8000/login
curl -o file.zip https://example.com/file.zip
wget https://example.com/file.tar.gz
```

Pair with [[API - FastAPI]] and [[Commands/CLI — Newman & Postman]].

---

## Firewall (concept)

```bash
# Ubuntu ufw (if enabled)
sudo ufw status
sudo ufw allow 22/tcp
```

Production firewalls vary — [[GCP]], cloud security groups.

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Processes & Services]]
- [[Load Testing]]
- [[K8S]]

---

## Tags

#linux #networking #curl #ss #dns #ip
