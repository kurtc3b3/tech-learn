## What & When

Daily **file, search, and shell** commands for Linux servers and SSH sessions. Overview: [[Linux]].

---

## Install (macOS — optional local practice)

```bash
brew install ripgrep fd   # optional modern find/grep alternatives
```

On Ubuntu/Debian servers, `grep`, `find`, and `coreutils` are preinstalled.

---

## Directory & Files

| Command | Description |
| --- | --- |
| `pwd` | Current directory |
| `ls` | List |
| `ls -la` | Long + hidden |
| `cd /path` | Change dir |
| `cd ..` | Parent |
| `mkdir -p a/b/c` | Nested dirs |
| `rm file` | Delete file |
| `rm -r dir` | Recursive delete |
| `cp -r src dest` | Copy tree |
| `mv src dest` | Move/rename |
| `touch file` | Create / touch mtime |
| `stat file` | Metadata |

```bash
# Safe-ish recursive delete
rm -rf -- /tmp/myapp-cache/
```

---

## View Files

| Command | Description |
| --- | --- |
| `cat file` | Print file |
| `less file` | Paged view |
| `head -n 20 file` | First lines |
| `tail -n 50 file` | Last lines |
| `tail -f file` | Follow (logs) |
| `wc -l file` | Line count |

---

## Find & Search

| Command | Description |
| --- | --- |
| `find /path -name '*.log'` | Find by name |
| `find . -type f -size +100M` | Large files |
| `grep pattern file` | Search |
| `grep -ri pattern dir` | Recursive |
| `grep -i pattern file` | Case-insensitive |
| `which cmd` | Binary path |
| `whereis cmd` | Binary + man |

```bash
# Large files under /var
find /var -type f -size +100M -exec ls -lh {} \;
```

---

## Pipes & Redirection

| Syntax | Meaning |
| --- | --- |
| `cmd1 \| cmd2` | Pipe stdout |
| `cmd > file` | Redirect stdout |
| `cmd >> file` | Append |
| `cmd 2> err.log` | Stderr only |
| `cmd &> all.log` | stdout + stderr |
| `cmd &` | Background |

```bash
ps aux | grep uvicorn
history | tail -20
```

---

## Shortcuts

| Key / cmd | Action |
| --- | --- |
| `Ctrl+C` | Interrupt |
| `Ctrl+Z` | Suspend |
| `Ctrl+D` | EOF / logout |
| `!!` | Repeat last command |
| `history` | Command history |
| `clear` | Clear screen |

---

## Power Combos

```bash
cat access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

```bash
ps aux | grep -E 'nginx|uvicorn' | grep -v grep
```

Text tools: [[Commands/Linux — Text & Sessions]].

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Files Advanced]]
- [[Commands/Linux — Text & Sessions]]
- [[API - FastAPI]]

---

## Tags

#linux #commands #files #grep #find #pipes
