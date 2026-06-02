## What & When

**Typer** is a library for building CLI applications using Python type hints. It is built on top of Click and follows the same philosophy as FastAPI — declare your interface with type annotations and let the framework handle validation, help text, and shell completion automatically.

Use Typer when:

- Building CLI tools, scripts, or management commands
- Replacing `argparse` with something more ergonomic
- Creating CLIs that share models/types with a FastAPI service
- Wanting automatic `--help`, shell completion, and rich output
- Building multi-command CLIs (like `git`, `docker`, `kubectl`)

```bash
pip install typer
pip install "typer[all]"    # includes rich (pretty output) + shellingham (shell detection)
```

---

## Typer vs argparse vs Click

|                  | Typer      | Click    | argparse |
| ---------------- | ---------- | -------- | -------- |
| Type hints       | ✅ Native   | ❌ Manual | ❌ Manual |
| Auto help text   | ✅          | ✅        | ✅        |
| Shell completion | ✅ Auto     | Manual   | ❌        |
| Rich output      | ✅ Built-in | ❌        | ❌        |
| FastAPI-like API | ✅          | ❌        | ❌        |
| Multi-command    | ✅          | ✅        | Limited  |
| Stdlib           | ❌          | ❌        | ✅        |
| Based on         | Click      | —        | —        |

---

## Basic App

```python
# main.py
import typer

app = typer.Typer()

@app.command()
def hello(name: str):
    """Say hello to NAME."""
    typer.echo(f"Hello, {name}!")

if __name__ == "__main__":
    app()
```

```bash
python main.py Alice
# Hello, Alice!

python main.py --help
# Usage: main.py [OPTIONS] NAME
#   Say hello to NAME.
# Options:
#   --help  Show this message and exit.
```

> [!tip] Single-command shortcut For a single command app you can skip `Typer()` and call `typer.run(fn)` directly:

```python
import typer

def main(name: str):
    typer.echo(f"Hello, {name}!")

if __name__ == "__main__":
    typer.run(main)
```

---

## Arguments vs Options

```python
import typer
from typing import Optional

app = typer.Typer()

@app.command()
def process(
    # Arguments — positional, required by default
    filename:   str,                    # required positional argument
    output_dir: str = "output",         # optional positional with default

    # Options — keyword, prefixed with --
    verbose:    bool          = False,              # --verbose / --no-verbose
    count:      int           = 1,                  # --count 5
    tag:        Optional[str] = None,               # --tag my-tag (optional)
    format:     str           = typer.Option("json", help="Output format"),
):
    """Process FILENAME and write to OUTPUT_DIR."""
    typer.echo(f"Processing {filename} → {output_dir}")
    if verbose:
        typer.echo(f"Count: {count}, Tag: {tag}, Format: {format}")
```

```bash
python main.py data.csv results/ --verbose --count 3 --tag v1 --format csv
```

---

## `typer.Argument` & `typer.Option`

Full control over argument metadata.

```python
import typer
from typing import Optional

@app.command()
def deploy(
    service: str = typer.Argument(
        ...,                                    # ... = required
        help="Service name to deploy",
        metavar="SERVICE",
    ),
    env: str = typer.Option(
        "staging",
        "--env", "-e",                          # short alias
        help="Target environment",
        show_default=True,
    ),
    replicas: int = typer.Option(
        1,
        "--replicas", "-r",
        min=1, max=20,
        help="Number of replicas",
    ),
    dry_run: bool = typer.Option(
        False,
        "--dry-run",
        help="Preview without applying",
        is_flag=True,
    ),
    tag: Optional[str] = typer.Option(
        None,
        envvar="DEPLOY_TAG",                    # read from env var if not set
        help="Docker image tag",
    ),
):
    """Deploy SERVICE to the target environment."""
    typer.echo(f"Deploying {service} to {env} (replicas={replicas})")
    if dry_run:
        typer.echo("[DRY RUN] No changes applied")
```

---

## Types & Validation

Typer uses Python types for automatic validation and coercion.

```python
import typer
from pathlib import Path
from datetime import datetime
from enum import Enum

class Environment(str, Enum):
    dev     = "dev"
    staging = "staging"
    prod    = "prod"

@app.command()
def run(
    # Primitives
    name:    str,
    port:    int   = 8000,
    rate:    float = 0.5,
    debug:   bool  = False,

    # Path — validated to exist
    config:  Path  = typer.Option(
        Path("config.yaml"),
        exists=True,                    # must exist
        file_okay=True,
        dir_okay=False,
        readable=True,
    ),

    # Output path — parent must exist
    output:  Path  = typer.Option(
        Path("output/"),
        writable=True,
        resolve_path=True,
    ),

    # Enum — choices shown in help and validated
    env:     Environment = Environment.dev,

    # Date
    since:   datetime = typer.Option(
        None,
        formats=["%Y-%m-%d", "%Y-%m-%dT%H:%M:%S"],
    ),

    # UUID
    job_id:  str = typer.Argument(...),
):
    typer.echo(f"Running {name} on port {port} in {env}")
```

---

## Multiple Values

```python
import typer
from typing import List, Optional

@app.command()
def process(
    # List argument — accept multiple values
    files: List[str] = typer.Argument(..., help="Files to process"),

    # List option — --tag a --tag b --tag c
    tags:  Optional[List[str]] = typer.Option(None, "--tag", "-t"),
):
    for f in files:
        typer.echo(f"Processing: {f}")
    if tags:
        typer.echo(f"Tags: {', '.join(tags)}")
```

```bash
python main.py file1.csv file2.csv --tag v1 --tag latest
```

---

## Multi-Command App

```python
import typer

app = typer.Typer(
    name="myapp",
    help="My CLI application",
    no_args_is_help=True,           # show help when called with no args
)

@app.command()
def serve(
    host: str = "0.0.0.0",
    port: int = 8000,
    reload: bool = False,
):
    """Start the development server."""
    import uvicorn
    uvicorn.run("main:app", host=host, port=port, reload=reload)

@app.command()
def migrate(
    revision: str = "head",
    dry_run:  bool = False,
):
    """Run database migrations."""
    typer.echo(f"Migrating to: {revision}")

@app.command()
def shell():
    """Open an interactive Python shell."""
    import code
    code.interact(local={})

@app.command("create-user")          # kebab-case command name
def create_user(
    email:    str,
    password: str = typer.Option(..., prompt=True, hide_input=True),
    admin:    bool = False,
):
    """Create a new user account."""
    typer.echo(f"Creating user: {email} (admin={admin})")

if __name__ == "__main__":
    app()
```

```bash
myapp --help
myapp serve --port 9000 --reload
myapp migrate --revision abc123
myapp create-user alice@example.com --admin
```

---

## Sub-Applications — Nested Commands

Like `docker compose`, `kubectl get`, etc.

```python
import typer

app     = typer.Typer(no_args_is_help=True)
db_app  = typer.Typer(no_args_is_help=True, help="Database management commands")
user_app = typer.Typer(no_args_is_help=True, help="User management commands")

# Register sub-apps
app.add_typer(db_app,   name="db")
app.add_typer(user_app, name="users")

@db_app.command("migrate")
def db_migrate(revision: str = "head"):
    """Run database migrations."""
    typer.echo(f"Migrating: {revision}")

@db_app.command("reset")
def db_reset(confirm: bool = typer.Option(..., prompt="Are you sure?")):
    """Reset the database."""
    if confirm:
        typer.echo("Database reset!")

@user_app.command("create")
def user_create(email: str, admin: bool = False):
    """Create a new user."""
    typer.echo(f"Creating {email}")

@user_app.command("list")
def user_list(limit: int = 20):
    """List all users."""
    typer.echo(f"Listing {limit} users")

if __name__ == "__main__":
    app()
```

```bash
myapp db migrate --revision head
myapp db reset
myapp users create alice@example.com --admin
myapp users list --limit 50
```

---

## Prompts & Confirmation

```python
import typer

@app.command()
def delete_user(user_id: int):
    """Delete a user account."""
    # Confirmation prompt
    confirm = typer.confirm(f"Delete user {user_id}?", abort=True)
    # abort=True raises typer.Abort if user says no

    typer.echo(f"Deleted user {user_id}")

@app.command()
def set_password(
    # Auto-prompt if not provided on CLI
    password: str = typer.Option(
        ...,
        prompt="New password",
        confirmation_prompt="Confirm password",
        hide_input=True,
    ),
):
    typer.echo("Password updated")

@app.command()
def interactive():
    # Manual prompts
    name  = typer.prompt("What is your name?")
    age   = typer.prompt("Your age", type=int, default=25)
    color = typer.prompt("Favourite colour", default="blue")
    typer.echo(f"Hello {name}, age {age}, loves {color}")
```

---

## Rich Output

```python
import typer
from rich.console import Console
from rich.table import Table
from rich.progress import track
from rich import print as rprint
import time

console = Console()

@app.command()
def list_users():
    """List all users in a table."""
    table = Table("ID", "Name", "Email", "Role", title="Users")
    table.add_row("1", "Alice", "alice@example.com", "[green]admin[/green]")
    table.add_row("2", "Bob",   "bob@example.com",   "user")
    table.add_row("3", "Carol", "carol@example.com",  "[red]banned[/red]")
    console.print(table)

@app.command()
def process_files(count: int = 10):
    """Process files with a progress bar."""
    for item in track(range(count), description="Processing..."):
        time.sleep(0.1)
    console.print("[bold green]✓ Done![/bold green]")

@app.command()
def status():
    """Show system status."""
    rprint("[bold]System Status[/bold]")
    rprint(f"  Database: [green]✓ Connected[/green]")
    rprint(f"  Redis:    [yellow]⚠ Degraded[/yellow]")
    rprint(f"  API:      [red]✗ Down[/red]")
```

---

## Callbacks — App-Level Options

Add global options (like `--version`, `--config`) that run before any command.

```python
import typer
from typing import Optional

app = typer.Typer()

def version_callback(value: bool):
    if value:
        typer.echo("myapp v1.0.0")
        raise typer.Exit()

@app.callback()
def main(
    ctx: typer.Context,
    version: Optional[bool] = typer.Option(
        None,
        "--version", "-v",
        callback=version_callback,
        is_eager=True,              # process before other options
        help="Show version and exit",
    ),
    verbose: bool = typer.Option(False, "--verbose", "-V"),
    config:  str  = typer.Option("config.yaml", envvar="APP_CONFIG"),
):
    """My CLI application."""
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose
    ctx.obj["config"]  = config
```

---

## Context — Passing State Between Commands

```python
import typer
from typing import Optional

app = typer.Typer()

@app.callback()
def setup(ctx: typer.Context, debug: bool = False):
    ctx.ensure_object(dict)
    ctx.obj["debug"] = debug

@app.command()
def run(ctx: typer.Context, name: str):
    debug = ctx.obj["debug"]
    if debug:
        typer.echo(f"[DEBUG] Running: {name}")
    typer.echo(f"Running: {name}")
```

---

## Exit Codes & Errors

```python
import typer
import sys

@app.command()
def risky_operation(path: str):
    try:
        result = do_work(path)
        typer.echo(result)
    except FileNotFoundError:
        typer.echo(f"Error: {path} not found", err=True)  # stderr
        raise typer.Exit(code=1)

    except PermissionError:
        typer.secho("Permission denied", fg=typer.colors.RED, err=True)
        raise typer.Exit(code=2)

# Abort — user-friendly exit with "Aborted!" message
raise typer.Abort()

# Clean exit
raise typer.Exit(code=0)
```

---

## Colours & Styling

```python
import typer

# typer.secho — styled echo
typer.secho("Success!", fg=typer.colors.GREEN, bold=True)
typer.secho("Warning!", fg=typer.colors.YELLOW)
typer.secho("Error!",   fg=typer.colors.RED, err=True)   # stderr

# typer.style — return styled string
msg = typer.style("important", fg=typer.colors.CYAN, bold=True, underline=True)
typer.echo(f"This is {msg} text")

# Available colours
typer.colors.BLACK
typer.colors.RED
typer.colors.GREEN
typer.colors.YELLOW
typer.colors.BLUE
typer.colors.MAGENTA
typer.colors.CYAN
typer.colors.WHITE
typer.colors.BRIGHT_RED
typer.colors.BRIGHT_GREEN
```

---

## Shell Completion

```bash
# Install completion for your shell
myapp --install-completion

# Show completion script (without installing)
myapp --show-completion

# Supported shells: bash, zsh, fish, PowerShell
```

```python
# Completion works automatically for:
# - Enum choices
# - Path arguments (file/directory completion)
# - bool options (--flag / --no-flag)

# Custom completion
import typer
from click.shell_completion import CompletionItem

def complete_environment(ctx, param, incomplete):
    envs = ["dev", "staging", "prod"]
    return [e for e in envs if e.startswith(incomplete)]

@app.command()
def deploy(
    env: str = typer.Argument(
        ...,
        autocompletion=complete_environment,
    ),
):
    ...
```

---

## Testing Typer Apps

```python
from typer.testing import CliRunner
from myapp import app

runner = CliRunner()

def test_hello():
    result = runner.invoke(app, ["Alice"])
    assert result.exit_code == 0
    assert "Hello, Alice!" in result.output

def test_hello_missing_arg():
    result = runner.invoke(app, [])
    assert result.exit_code != 0

def test_deploy_dry_run():
    result = runner.invoke(app, ["deploy", "api", "--env", "prod", "--dry-run"])
    assert result.exit_code == 0
    assert "DRY RUN" in result.output

def test_version():
    result = runner.invoke(app, ["--version"])
    assert "v1.0.0" in result.output
```

---

## FastAPI + Typer — Shared in One Project

A common pattern — one codebase with both a REST API and a CLI.

```python
# cli.py
import typer
from myapp.config import get_settings
from myapp.db import create_tables, SessionLocal
from myapp.models import User

cli = typer.Typer(name="myapp", no_args_is_help=True)

@cli.command()
def serve(
    host:   str  = "0.0.0.0",
    port:   int  = 8000,
    reload: bool = False,
    workers: int = 1,
):
    """Start the FastAPI server."""
    import uvicorn
    uvicorn.run(
        "myapp.main:app",
        host=host,
        port=port,
        reload=reload,
        workers=workers,
    )

@cli.command("init-db")
def init_db():
    """Initialise database tables."""
    typer.echo("Creating tables...")
    create_tables()
    typer.echo("[green]✓ Done[/green]")

@cli.command("create-superuser")
def create_superuser(
    email:    str = typer.Argument(...),
    password: str = typer.Option(..., prompt=True, hide_input=True, confirmation_prompt=True),
):
    """Create a superuser account."""
    with SessionLocal() as db:
        user = User(email=email, is_superuser=True)
        user.set_password(password)
        db.add(user)
        db.commit()
    typer.secho(f"✓ Superuser {email} created", fg=typer.colors.GREEN)

@cli.command()
def shell():
    """Open an interactive Python shell with app context."""
    import code
    from myapp.main import app as fastapi_app
    code.interact(local={"app": fastapi_app, "settings": get_settings()})

if __name__ == "__main__":
    cli()
```

```toml
# pyproject.toml
[project.scripts]
myapp = "myapp.cli:cli"
```

```bash
myapp serve --port 9000 --reload
myapp init-db
myapp create-superuser admin@example.com
myapp shell
```

---

## Quick Reference

|Task|Code|
|---|---|
|Basic app|`app = typer.Typer(); @app.command()`|
|Single command|`typer.run(fn)`|
|Argument|`name: str` (positional)|
|Required option|`typer.Option(...)`|
|Optional option|`typer.Option(None)`|
|Short alias|`typer.Option(..., "--name", "-n")`|
|Env var fallback|`typer.Option(None, envvar="MY_VAR")`|
|Prompt if missing|`typer.Option(..., prompt=True)`|
|Hidden input|`typer.Option(..., hide_input=True)`|
|Path validation|`typer.Option(..., exists=True)`|
|Enum choices|`env: MyEnum = MyEnum.dev`|
|List argument|`files: List[str]`|
|Flag|`debug: bool = False`|
|Confirmation|`typer.confirm("Sure?", abort=True)`|
|Styled output|`typer.secho("msg", fg=typer.colors.GREEN)`|
|Rich table|`from rich.table import Table`|
|Progress bar|`from rich.progress import track`|
|Exit with code|`raise typer.Exit(code=1)`|
|Write to stderr|`typer.echo("error", err=True)`|
|Sub-commands|`app.add_typer(sub, name="sub")`|
|Global options|`@app.callback()`|
|Version flag|`callback=version_callback, is_eager=True`|
|Shell completion|`myapp --install-completion`|
|Testing|`CliRunner().invoke(app, ["args"])`|
|Script entry|`[project.scripts] myapp = "pkg.cli:app"`|

---

## Tags

#python #typer #cli #terminal #argparse #click #backend #devtools