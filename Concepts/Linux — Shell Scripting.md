**Key Points:**

- **Bash is the default automation language** on Linux servers — deploy scripts, cron wrappers, log pipelines.
- **Safe defaults** — `set -euo pipefail`, quote variables, prefer `[[ ]]`, use `#!/usr/bin/env bash`.
- **Don't parse `ls`** — use globs, `find`, or `read` loops.
- **Syntax reference** — [[Commands/Linux — Bash Scripting]]; this note covers *why* and *patterns*.
- **Lint scripts** — `shellcheck` before production cron.

# Linux — Shell Scripting

Part of [[Linux]]. Syntax tables: [[Commands/Linux — Bash Scripting]].

---

## When to Script vs Other Tools

| Use Bash when | Use something else when |
| --- | --- |
| Glue on Linux servers | Cross-platform app logic → Python ([[Python Development]]) |
| Cron / systemd oneshot | Complex API → [[Python — Typer]] |
| Log one-liners + small tools | Heavy data → Python/pandas |
| Deploy hook | Infra as code → Terraform (external) |

---

## Script Skeleton (Production)

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

main() {
  echo "Starting..."
}

main "$@"
```

| Option | Effect |
| --- | --- |
| `-e` | Exit on first failing command |
| `-u` | Error on unset variables |
| `-o pipefail` | Pipeline fails if any stage fails |

---

## Core Patterns

### Check file before action

```bash
for f in /var/log/app/*.log; do
  [[ -f "$f" ]] || continue
  gzip "$f"
done
```

### Read file line by line

```bash
while IFS= read -r line; do
  printf '%s\n' "$line"
done < "$file"
```

### Wait for dependency

```bash
until curl -sf http://localhost:8000/health >/dev/null; do
  sleep 2
done
```

Pairs with [[API - FastAPI]] health endpoints.

### Case for subcommands

```bash
case "${1:-}" in
  start) systemctl start myapp ;;
  stop)  systemctl stop myapp ;;
  *)     echo "Usage: $0 {start|stop}" >&2; exit 1 ;;
esac
```

---

## Variables and Quoting

- Always double-quote expansions: `"$var"`
- Prefer `$(command)` over backticks
- Use `${var:-default}` for defaults
- Export only what child processes need

---

## Error Handling

- Check exit codes: `if ! cmd; then ... fi`
- Provide context: `echo "Failed to backup $dir" >&2`
- Use `trap` for cleanup on exit/signal

```bash
trap 'rm -f "$tmpfile"' EXIT
```

---

## Functions

- Use `local` for function-scoped variables
- `return` sets exit status (0–255); use sparingly
- Pass args as `$1`, `$2` — same as script args

---

## Debugging Workflow

1. `bash -n script.sh` — syntax only
2. `bash -x script.sh` — trace execution
3. `shellcheck script.sh` — static lint

---

## sed vs awk (When to Use)

| Tool | Use for |
| --- | --- |
| **sed** | Line substitutions, delete/print ranges |
| **awk** | Columnar reports, sums, conditions on fields |
| **grep** | Find lines matching pattern |
| **cut** | Fixed delimiter columns |

Examples: [[Commands/Linux — Text & Sessions]].

---

## Cron Integration

Scripts invoked from cron should:

- Use absolute paths or `cd` explicitly
- Log stdout/stderr to files
- Set `PATH` if needed
- Avoid relying on interactive shell

Cron format: [[Commands/Linux — Processes & Services]].

---

## Anti-Patterns

| Avoid | Prefer |
| --- | --- |
| Unquoted `$var` | `"$var"` |
| Parsing `ls` output | `find`, globs |
| `rm -rf $dir` without quotes | `rm -rf -- "$dir"` |
| Secrets in script | Env vars, secret managers |
| Silent failures | `set -e` + explicit checks |

---

## Related Notes

- [[Linux]]
- [[Commands/Linux — Bash Scripting]]
- [[Commands/Linux — Text & Sessions]]
- [[Commands/Linux — Processes & Services]]
- [[CLI]]

---

## Tags

#linux #bash #shell-scripting #automation #cron #shellcheck
