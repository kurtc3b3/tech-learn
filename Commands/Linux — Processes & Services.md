## What & When

**Processes**, signals, **systemd**, and **cron**. Concept: [[Linux — Architecture]]. Overview: [[Linux]].

---

## Install (macOS)

```bash
brew install htop
```

---

## Process listing

| Command | Description |
| --- | --- |
| `ps` | Snapshot |
| `ps aux` | All processes |
| `ps aux \| grep uvicorn` | Filter |
| `top` | Live view |
| `htop` | Interactive top |
| `pstree` | Tree |

---

## Signals & kill

| Command | Description |
| --- | --- |
| `kill PID` | SIGTERM (default) |
| `kill -9 PID` | SIGKILL (force) |
| `pkill -f pattern` | By command line |
| `killall name` | By name |

| Signal | Typical use |
| --- | --- |
| SIGTERM (15) | Graceful stop |
| SIGKILL (9) | Force |
| SIGHUP (1) | Reload some daemons |

---

## Jobs

| Command | Description |
| --- | --- |
| `cmd &` | Background |
| `jobs` | List jobs |
| `fg %1` | Foreground |
| `bg %1` | Resume background |

---

## Priority

```bash
nice -n 10 cmd
renice -n 5 -p PID
```

---

## systemd

| Command | Description |
| --- | --- |
| `systemctl status nginx` | Status |
| `systemctl start nginx` | Start |
| `systemctl stop nginx` | Stop |
| `systemctl restart nginx` | Restart |
| `systemctl enable nginx` | Boot start |
| `systemctl disable nginx` | No boot |
| `systemctl list-units` | Units |
| `journalctl -u nginx -f` | Follow logs |

```bash
sudo systemctl daemon-reload   # after unit file change
```

---

## Cron

```text
* * * * * command
│ │ │ │ └── day of week (0-7)
│ │ │ └──── month
│ │ └────── day of month
│ └──────── hour
└────────── minute
```

| Command | Description |
| --- | --- |
| `crontab -l` | List |
| `crontab -e` | Edit |
| `crontab -r` | Remove |

```bash
# Every 5 minutes
*/5 * * * * /opt/scripts/health-check.sh
```

---

## at (one-shot)

```bash
echo "/opt/backup.sh" | at 02:00 tomorrow
atq
atrm 1
```

---

## System info

```bash
uname -a
hostname
uptime
free -h
vmstat 1
```

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Text & Sessions]]
- [[API - FastAPI]]
- [[K8S]]

---

## Tags

#linux #processes #systemd #cron #journalctl #htop
