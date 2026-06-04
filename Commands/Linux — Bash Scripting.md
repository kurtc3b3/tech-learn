## What & When

**Bash syntax reference** for scripts and cron. Patterns: [[Linux — Shell Scripting]]. Overview: [[Linux]].

---

## Install (macOS)

```bash
brew install bash shellcheck
# macOS default bash is old; scripts target 5.x on Linux
```

---

## Run scripts

```bash
chmod +x script.sh
./script.sh
bash script.sh
bash -n script.sh    # syntax check
bash -x script.sh    # trace
shellcheck script.sh
```

---

## Shebang & safety

```bash
#!/usr/bin/env bash
set -euo pipefail
```

| Flag | Meaning |
| --- | --- |
| `set -x` | Debug trace |
| `set -e` | Exit on error |
| `set -u` | Unset vars error |

---

## Variables

```bash
name="Linux"
echo "$name"
readonly VAR=1
export VAR=value
```

---

## Positional args

| Var | Meaning |
| --- | --- |
| `$0` | Script name |
| `$1`… | Arguments |
| `$#` | Count |
| `$@` | All args |
| `$?` | Last exit code |

---

## Conditionals

```bash
if [[ -f "$file" ]]; then
  echo "exists"
elif [[ "$n" -gt 10 ]]; then
  echo "big"
else
  echo "other"
fi
```

| Test | Meaning |
| --- | --- |
| `-f file` | File exists |
| `-d dir` | Directory |
| `-z "$s"` | Empty string |
| `-eq -ne -gt -lt` | Numbers |
| `==` `!=` | Strings (in `[[ ]]`) |

---

## Loops

```bash
for i in {1..5}; do echo "$i"; done
for f in *.log; do gzip "$f"; done

while read -r line; do
  echo "$line"
done < file

until curl -sf http://localhost:8000/health; do sleep 1; done
```

---

## Functions

```bash
greet() {
  local name="$1"
  echo "Hello $name"
}
greet World
```

---

## Substitution & pipes

```bash
today=$(date +%F)
files=$(wc -l < file)
cmd > out.txt 2> err.txt
cmd &> all.txt
cmd1 | cmd2
```

---

## case

```bash
case "$1" in
  start) echo start ;;
  stop)  echo stop ;;
  *) echo "usage: $0 start|stop"; exit 1 ;;
esac
```

---

## Arrays

```bash
arr=(one two three)
echo "${arr[0]}"
echo "${arr[@]}"
echo "${#arr[@]}"
```

---

## trap

```bash
trap 'echo cleanup' EXIT
trap 'echo interrupted' INT TERM
```

---

## String ops

```bash
str="linux-shell"
echo "${#str}"
echo "${str:0:5}"
echo "${str/old/new}"
```

---

## Real-world example

```bash
#!/usr/bin/env bash
set -euo pipefail

for f in /var/log/app/*.log; do
  [[ -f "$f" ]] || continue
  gzip "$f"
done
```

---

## Related Notes

- [[Linux]]
- [[Linux — Shell Scripting]]
- [[Commands/Linux — Text & Sessions]]
- [[Commands/Linux — Processes & Services]]

---

## Tags

#linux #bash #shell-scripting #shellcheck
