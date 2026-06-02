## What & When

**argparse** is Python's built-in library for parsing command-line arguments. It automatically generates help text, validates inputs, and converts argument values to the correct types — with zero dependencies.

Use argparse when:

- Building CLI tools with no external dependencies
- Writing scripts for environments where pip installs aren't available
- Needing fine-grained control over argument parsing
- Working in a stdlib-only codebase

For modern projects with dependencies allowed, consider **Typer** instead. See [[Python - Typer]].

```python
import argparse
```

No installation needed — argparse is part of the standard library.

---
## argparse vs Typer vs Click

|                  | argparse | Typer    | Click  |
| ---------------- | -------- | -------- | ------ |
| Stdlib           | ✅        | ❌        | ❌      |
| Type hints       | ❌ Manual | ✅ Native | ❌      |
| Auto help        | ✅        | ✅        | ✅      |
| Shell completion | ❌        | ✅        | Manual |
| Rich output      | ❌        | ✅        | ❌      |
| Sub-commands     | ✅        | ✅        | ✅      |
| Learning curve   | Medium   | Low      | Medium |

---

## Basic Parser

```python
import argparse

parser = argparse.ArgumentParser(
    prog="myapp",
    description="My CLI application",
    epilog="For more info visit https://example.com",
)

# Positional argument — required, no --flag
parser.add_argument("filename", help="Input file to process")

# Optional argument — prefixed with --
parser.add_argument("--output", "-o", help="Output directory", default="./output")
parser.add_argument("--verbose", "-v", action="store_true", help="Enable verbose output")
parser.add_argument("--count", "-n", type=int, default=10, help="Number of items")

args = parser.parse_args()

print(args.filename)    # positional
print(args.output)      # --output value or "./output"
print(args.verbose)     # True / False
print(args.count)       # int value
```

```bash
python main.py data.csv --output results/ --verbose --count 5
python main.py --help
```

---

## Positional Arguments

```python
parser = argparse.ArgumentParser()

# Required positional
parser.add_argument("source", help="Source file")

# With type conversion
parser.add_argument("port", type=int, help="Port number")

# With choices
parser.add_argument("env", choices=["dev", "staging", "prod"], help="Environment")

# With metavar — changes display name in help
parser.add_argument("url", metavar="URL", help="Target URL")

args = parser.parse_args()
```

---

## Optional Arguments (Options)

```python
parser = argparse.ArgumentParser()

# String option
parser.add_argument("--name", "-n", help="Your name")

# With default
parser.add_argument("--port", type=int, default=8000, help="Port (default: 8000)")

# Required option (unusual but valid)
parser.add_argument("--config", required=True, help="Config file path")

# Boolean flag — store_true / store_false
parser.add_argument("--verbose", action="store_true", help="Enable verbose mode")
parser.add_argument("--no-cache", action="store_false", dest="cache", help="Disable cache")

# Count — -v for verbose, -vv for more verbose
parser.add_argument("--verbose", "-v", action="count", default=0, help="Verbosity level")

# Const — store a fixed value when flag is present
parser.add_argument("--format", action="store_const", const="json", help="Use JSON output")

# Append — accumulate multiple values
parser.add_argument("--tag", action="append", dest="tags", help="Add a tag (repeatable)")

args = parser.parse_args(["--tag", "v1", "--tag", "latest"])
print(args.tags)    # ["v1", "latest"]
```

---

## Types & Validation

```python
import argparse
from pathlib import Path

parser = argparse.ArgumentParser()

# Built-in type conversion
parser.add_argument("--port",    type=int,   default=8000)
parser.add_argument("--rate",    type=float, default=0.5)
parser.add_argument("--workers", type=int,   default=4)

# Custom type function — raises argparse.ArgumentTypeError on failure
def positive_int(value):
    ivalue = int(value)
    if ivalue <= 0:
        raise argparse.ArgumentTypeError(f"{value} must be a positive integer")
    return ivalue

def valid_path(value):
    p = Path(value)
    if not p.exists():
        raise argparse.ArgumentTypeError(f"Path does not exist: {value}")
    return p

parser.add_argument("--count",  type=positive_int, help="Positive integer only")
parser.add_argument("--config", type=valid_path,   help="Must be an existing path")

# Choices — restricted set of values
parser.add_argument("--env",    choices=["dev", "staging", "prod"])
parser.add_argument("--level",  choices=range(1, 11), type=int, help="Level 1-10")

# File type
parser.add_argument("--input",  type=argparse.FileType("r"), help="Input file")
parser.add_argument("--output", type=argparse.FileType("w"), help="Output file")
```

---

## Nargs — Multiple Values

```python
parser = argparse.ArgumentParser()

# Exact number
parser.add_argument("coords", nargs=2, type=float, metavar=("X", "Y"))

# One or more
parser.add_argument("files", nargs="+", help="One or more files")

# Zero or more
parser.add_argument("--tags", nargs="*", help="Zero or more tags")

# Optional single value
parser.add_argument("--output", nargs="?", const="stdout", default=None)

# Remainder — capture everything after
parser.add_argument("command", nargs=argparse.REMAINDER, help="Command to run")

args = parser.parse_args(["file1.txt", "file2.txt"])
print(args.files)   # ["file1.txt", "file2.txt"]
```

---

## Argument Groups — Organising Help Text

```python
import argparse

parser = argparse.ArgumentParser(description="Deploy tool")

# Group positional/required args
required = parser.add_argument_group("required arguments")
required.add_argument("--env",    required=True, help="Target environment")
required.add_argument("--service",required=True, help="Service to deploy")

# Group optional args
deploy_opts = parser.add_argument_group("deploy options")
deploy_opts.add_argument("--replicas", type=int, default=1, help="Number of replicas")
deploy_opts.add_argument("--timeout",  type=int, default=300, help="Timeout in seconds")
deploy_opts.add_argument("--dry-run",  action="store_true", help="Preview only")

# Group output args
output_opts = parser.add_argument_group("output options")
output_opts.add_argument("--verbose", action="store_true")
output_opts.add_argument("--quiet",   action="store_true")

args = parser.parse_args()
```

---

## Mutually Exclusive Groups

```python
import argparse

parser = argparse.ArgumentParser()

# Only one of --verbose or --quiet can be used
group = parser.add_mutually_exclusive_group()
group.add_argument("--verbose", action="store_true")
group.add_argument("--quiet",   action="store_true")

# Required — at least one must be present
group = parser.add_mutually_exclusive_group(required=True)
group.add_argument("--file",   help="Input file")
group.add_argument("--stdin",  action="store_true", help="Read from stdin")

args = parser.parse_args()
```

---

## Sub-Commands — `add_subparsers`

```python
import argparse
import sys

parser = argparse.ArgumentParser(
    prog="myapp",
    description="My CLI tool",
)
parser.add_argument("--verbose", action="store_true")

subparsers = parser.add_subparsers(
    dest="command",
    title="commands",
    description="Available commands",
    metavar="COMMAND",
)

# serve command
serve_parser = subparsers.add_parser("serve", help="Start the server")
serve_parser.add_argument("--host",   default="0.0.0.0")
serve_parser.add_argument("--port",   type=int, default=8000)
serve_parser.add_argument("--reload", action="store_true")

# migrate command
migrate_parser = subparsers.add_parser("migrate", help="Run database migrations")
migrate_parser.add_argument("revision", nargs="?", default="head")
migrate_parser.add_argument("--dry-run", action="store_true")

# create-user command
user_parser = subparsers.add_parser("create-user", help="Create a new user")
user_parser.add_argument("email")
user_parser.add_argument("--admin", action="store_true")

# Dispatch
args = parser.parse_args()

if args.command == "serve":
    import uvicorn
    uvicorn.run("main:app", host=args.host, port=args.port, reload=args.reload)

elif args.command == "migrate":
    print(f"Migrating to: {args.revision}")

elif args.command == "create-user":
    print(f"Creating user: {args.email} (admin={args.admin})")

else:
    parser.print_help()
    sys.exit(1)
```

```bash
myapp serve --port 9000 --reload
myapp migrate head --dry-run
myapp create-user alice@example.com --admin
myapp --help
myapp serve --help
```

---

## Sub-Commands with `set_defaults` — Cleaner Dispatch

```python
import argparse

def cmd_serve(args):
    import uvicorn
    uvicorn.run("main:app", host=args.host, port=args.port, reload=args.reload)

def cmd_migrate(args):
    print(f"Migrating: {args.revision}")

def cmd_create_user(args):
    print(f"Creating: {args.email}")

parser    = argparse.ArgumentParser()
subparsers = parser.add_subparsers(dest="command")

# serve
p = subparsers.add_parser("serve")
p.add_argument("--host",   default="0.0.0.0")
p.add_argument("--port",   type=int, default=8000)
p.add_argument("--reload", action="store_true")
p.set_defaults(func=cmd_serve)         # ← bind function to command

# migrate
p = subparsers.add_parser("migrate")
p.add_argument("revision", nargs="?", default="head")
p.set_defaults(func=cmd_migrate)

# create-user
p = subparsers.add_parser("create-user")
p.add_argument("email")
p.set_defaults(func=cmd_create_user)

# Dispatch — clean one-liner
args = parser.parse_args()
if hasattr(args, "func"):
    args.func(args)
else:
    parser.print_help()
```

---

## Namespace — Accessing Parsed Values

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--name",  default="World")
parser.add_argument("--count", type=int, default=1)

args = parser.parse_args()

# Attribute access
args.name           # "World"
args.count          # 1

# Convert to dict
vars(args)          # {"name": "World", "count": 1}

# Check if argument was provided
if args.name != parser.get_default("name"):
    print("Name was explicitly set")

# Custom namespace
namespace = argparse.Namespace(name="Alice", count=5)
args = parser.parse_args(namespace=namespace)

# Parse from a list (instead of sys.argv) — useful for testing
args = parser.parse_args(["--name", "Alice", "--count", "3"])
```

---

## Defaults from Environment Variables

argparse doesn't support env vars natively — read them manually.

```python
import argparse
import os

parser = argparse.ArgumentParser()

parser.add_argument(
    "--db-url",
    default=os.environ.get("DATABASE_URL", "sqlite:///./db.sqlite3"),
    help="Database URL (env: DATABASE_URL)",
)
parser.add_argument(
    "--secret-key",
    default=os.environ.get("SECRET_KEY"),
    required="SECRET_KEY" not in os.environ,
    help="Secret key (env: SECRET_KEY)",
)
parser.add_argument(
    "--debug",
    action="store_true",
    default=os.environ.get("DEBUG", "").lower() in ("1", "true", "yes"),
)

args = parser.parse_args()
```

> [!tip] Use Typer or pydantic-settings for env var support For richer env var integration with type coercion, `typer.Option(envvar=...)` or `pydantic-settings` `BaseSettings` are much cleaner. See [[Python - Typer]] and [[Python - Pydantic]].

---

## Custom Help Formatting

```python
import argparse

# Raw description — preserves whitespace and newlines
parser = argparse.ArgumentParser(
    description="""
My CLI tool.

Examples:
  myapp process data.csv --output results/
  myapp serve --port 9000 --reload
    """,
    formatter_class=argparse.RawDescriptionHelpFormatter,
)

# Argument defaults shown in help automatically
parser = argparse.ArgumentParser(
    formatter_class=argparse.ArgumentDefaultsHelpFormatter
)
parser.add_argument("--port", type=int, default=8000, help="Port number")
# Help shows: --port PORT  Port number (default: 8000)

# Both combined
class HelpFormatter(
    argparse.ArgumentDefaultsHelpFormatter,
    argparse.RawDescriptionHelpFormatter,
):
    pass

parser = argparse.ArgumentParser(formatter_class=HelpFormatter)
```

---

## Error Handling

```python
import argparse
import sys

parser = argparse.ArgumentParser()
parser.add_argument("--port", type=int)

# Default — argparse prints error and exits with code 2
args = parser.parse_args()

# Custom error handling
try:
    args = parser.parse_args()
except SystemExit as e:
    print(f"Argument error (exit code {e.code})")
    sys.exit(e.code)

# Custom error message
def error(message):
    parser.print_usage(sys.stderr)
    print(f"error: {message}", file=sys.stderr)
    sys.exit(2)

parser.error = error

# Validate after parsing
args = parser.parse_args()
if args.port and not (1 <= args.port <= 65535):
    parser.error(f"Port must be 1-65535, got {args.port}")
```

---

## `parse_known_args` — Partial Parsing

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--verbose", action="store_true")

# Returns (namespace, list_of_unknown_args)
args, unknown = parser.parse_known_args(["--verbose", "--extra", "value"])
print(args)     # Namespace(verbose=True)
print(unknown)  # ["--extra", "value"]

# Useful when passing remaining args to another command
args, passthrough = parser.parse_known_args()
subprocess.run(["other-tool"] + passthrough)
```

---

## Config File + CLI — Combined Pattern

```python
import argparse
import json
from pathlib import Path

def load_config(path: str) -> dict:
    p = Path(path)
    if p.exists():
        return json.loads(p.read_text())
    return {}

parser = argparse.ArgumentParser()
parser.add_argument("--config", default="config.json", help="Config file")
parser.add_argument("--host",   default=None)
parser.add_argument("--port",   type=int, default=None)
parser.add_argument("--debug",  action="store_true", default=None)

args = parser.parse_args()

# Load config file
config = load_config(args.config)

# CLI args override config file — only if explicitly provided
final = {
    "host":  args.host  or config.get("host",  "0.0.0.0"),
    "port":  args.port  or config.get("port",  8000),
    "debug": args.debug or config.get("debug", False),
}
```

---

## Testing argparse

```python
import argparse
import pytest

def make_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser()
    parser.add_argument("name")
    parser.add_argument("--count", type=int, default=1)
    parser.add_argument("--verbose", action="store_true")
    return parser

def test_basic():
    parser = make_parser()
    args = parser.parse_args(["Alice"])
    assert args.name == "Alice"
    assert args.count == 1
    assert args.verbose is False

def test_with_options():
    parser = make_parser()
    args = parser.parse_args(["Bob", "--count", "5", "--verbose"])
    assert args.name == "Bob"
    assert args.count == 5
    assert args.verbose is True

def test_invalid_exits():
    parser = make_parser()
    with pytest.raises(SystemExit) as exc:
        parser.parse_args(["--count", "not-an-int"])
    assert exc.value.code == 2
```

---

## Complete CLI Example

```python
#!/usr/bin/env python3
"""
myapp — A complete argparse CLI example.
"""
import argparse
import sys
import os
from pathlib import Path

def cmd_serve(args: argparse.Namespace) -> int:
    print(f"Starting server on {args.host}:{args.port}")
    if args.reload:
        print("Auto-reload enabled")
    return 0

def cmd_process(args: argparse.Namespace) -> int:
    for f in args.files:
        print(f"Processing: {f}")
    if args.verbose:
        print(f"Output: {args.output}")
    return 0

def make_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="myapp",
        description="My CLI tool",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
    )
    parser.add_argument("--version", action="version", version="myapp 1.0.0")

    subparsers = parser.add_subparsers(dest="command", metavar="COMMAND")

    # serve
    serve = subparsers.add_parser("serve", help="Start server",
                                   formatter_class=argparse.ArgumentDefaultsHelpFormatter)
    serve.add_argument("--host",   default="0.0.0.0")
    serve.add_argument("--port",   type=int, default=8000)
    serve.add_argument("--reload", action="store_true")
    serve.set_defaults(func=cmd_serve)

    # process
    process = subparsers.add_parser("process", help="Process files",
                                     formatter_class=argparse.ArgumentDefaultsHelpFormatter)
    process.add_argument("files",  nargs="+", help="Files to process")
    process.add_argument("--output", "-o", default="./output")
    process.add_argument("--verbose", "-v", action="store_true")
    process.set_defaults(func=cmd_process)

    return parser

def main() -> int:
    parser = make_parser()
    args   = parser.parse_args()

    if not args.command:
        parser.print_help()
        return 1

    return args.func(args)

if __name__ == "__main__":
    sys.exit(main())
```

---

## Quick Reference

|Task|Code|
|---|---|
|Create parser|`argparse.ArgumentParser(description="...")`|
|Positional arg|`parser.add_argument("name")`|
|Optional arg|`parser.add_argument("--name", "-n")`|
|With type|`parser.add_argument("--port", type=int)`|
|With default|`parser.add_argument("--port", default=8000)`|
|Required option|`parser.add_argument("--key", required=True)`|
|Boolean flag|`parser.add_argument("--verbose", action="store_true")`|
|Count flag|`parser.add_argument("-v", action="count", default=0)`|
|Append values|`parser.add_argument("--tag", action="append")`|
|Choices|`parser.add_argument("--env", choices=["dev","prod"])`|
|Multiple values|`parser.add_argument("files", nargs="+")`|
|Custom type|`parser.add_argument("--n", type=positive_int)`|
|File argument|`parser.add_argument("--f", type=argparse.FileType("r"))`|
|Arg group|`parser.add_argument_group("group name")`|
|Mutually exclusive|`parser.add_mutually_exclusive_group()`|
|Sub-commands|`parser.add_subparsers(dest="command")`|
|Bind function|`subparser.set_defaults(func=my_fn)`|
|Parse args|`args = parser.parse_args()`|
|Parse from list|`args = parser.parse_args(["--port", "9000"])`|
|To dict|`vars(args)`|
|Partial parse|`args, unknown = parser.parse_known_args()`|
|Custom error|`parser.error("message")`|
|Version flag|`action="version", version="1.0.0"`|
|Show help|`parser.print_help()`|
|Raw help format|`formatter_class=RawDescriptionHelpFormatter`|
|Show defaults|`formatter_class=ArgumentDefaultsHelpFormatter`|

---

## Tags

#python #argparse #cli #terminal #stdlib #scripting #backend