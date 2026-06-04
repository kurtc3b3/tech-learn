## What & When

**sed**, **awk**, **sort/uniq**, **tmux/screen**, and **log** commands. Overview: [[Linux]]. Concepts: [[Linux — Shell Scripting]].

---

## Install (macOS)

```bash
brew install tmux
# GNU sed on macOS (optional):
brew install gnu-sed
# watch:
brew install watch
```

---

## sed

| Command | Description |
| --- | --- |
| `sed 's/old/new/' file` | First match per line |
| `sed 's/old/new/g' file` | All matches |
| `sed -i 's/old/new/g' file` | In-place (GNU) |
| `sed -n '5p' file` | Print line 5 |
| `sed -n '5,10p' file` | Lines 5–10 |
| `sed '/pattern/d' file` | Delete matching lines |

```bash
sed -i 's/ERROR/WARN/g' app.log
```

macOS in-place: `sed -i '' 's/a/b/' file`

---

## awk

| Command | Description |
| --- | --- |
| `awk '{print $1}' file` | First column |
| `awk -F: '{print $1}' /etc/passwd` | Custom delimiter |
| `awk '$3 > 1000' file` | Filter |
| `awk '{s+=$1} END {print s}' file` | Sum column |

```bash
df -h | awk '$5+0 > 80 {print $1, $5}'
```

---

## cut, sort, uniq, tr

```bash
cut -d: -f1 /etc/passwd
sort file
sort -n file
sort -nr file
uniq -c file
tr 'a-z' 'A-Z' < file
```

---

## Logs

| Command | Description |
| --- | --- |
| `tail -f /var/log/syslog` | Follow file |
| `grep -i error app.log` | Search |
| `journalctl` | systemd logs |
| `journalctl -u nginx` | Unit logs |
| `journalctl -f` | Follow |
| `journalctl --since "1 hour ago"` | Window |

---

## watch

```bash
watch -n 1 'df -h'
watch -n 2 'ss -tulpn | head'
```

---

## screen

| Command | Description |
| --- | --- |
| `screen` | New session |
| `screen -S name` | Named |
| `screen -ls` | List |
| `screen -r name` | Attach |
| `Ctrl+A D` | Detach |

---

## tmux (preferred)

| Command | Description |
| --- | --- |
| `tmux` | New session |
| `tmux new -s dev` | Named |
| `tmux ls` | List |
| `tmux attach -t dev` | Attach |
| `Ctrl+B D` | Detach |
| `Ctrl+B %` | Split vertical |
| `Ctrl+B "` | Split horizontal |

**Headless server habit:** start `tmux` before long deploys so SSH drops do not kill the job.

---

## Combos

```bash
df -h | sed '1d' | awk '$5+0 > 90 {print $0}'
```

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Essentials]]
- [[Commands/Linux — Bash Scripting]]
- [[Commands/Linux — Processes & Services]]

---

## Tags

#linux #sed #awk #tmux #journalctl #logs
