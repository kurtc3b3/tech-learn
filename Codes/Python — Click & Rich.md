## What & When

**Click** is a composable CLI framework built on decorators. It is the foundation that Typer is built on, and is the standard choice for complex CLIs when you want more control than Typer provides.

**Rich** is a terminal formatting library that adds colour, tables, progress bars, syntax highlighting, markdown, and pretty-printing to any Python application — CLI or otherwise.

```bash
pip install click
pip install rich
pip install "rich[jupyter]"   # Jupyter notebook support
```

---
## Click vs argparse vs Typer

|                  | Click       | argparse | Typer       |
| ---------------- | ----------- | -------- | ----------- |
| Stdlib           | ❌           | ✅        | ❌           |
| Decorator API    | ✅           | ❌        | ✅           |
| Type hints       | Manual      | Manual   | ✅ Native    |
| Composable       | ✅           | Limited  | ✅           |
| Shell completion | Manual      | ❌        | ✅ Auto      |
| Testing          | ✅ CliRunner | Manual   | ✅ CliRunner |
| Rich output      | Manual      | ❌        | ✅ Built-in  |
| Typer built on   | ✅           | ❌        | —           |

---

## Basic Click App

```python
import click

@click.command()
@click.argument("name")
@click.option("--count", "-c", default=1, help="Number of greetings")
@click.option("--verbose", "-v", is_flag=True, help="Enable verbose output")
def hello(name: str, count: int, verbose: bool):
    """Greet NAME a number of times."""
    for _ in range(count):
        click.echo(f"Hello, {name}!")
    if verbose:
        click.echo(f"(Greeted {count} time(s))")

if __name__ == "__main__":
    hello()
```

```bash
python main.py Alice --count 3 --verbose
python main.py --help
```

---

## Arguments vs Options

```python
import click

@click.command()
# Arguments — positional, required by default
@click.argument("source")
@click.argument("destination", default="./output")

# Options — keyword, prefixed with --
@click.option("--port",    "-p", type=int, default=8000, show_default=True)
@click.option("--env",         default="dev",  show_default=True)
@click.option("--verbose", "-v", is_flag=True)
@click.option("--dry-run",      is_flag=True)
@click.option("--tag",     "-t", multiple=True, help="Tags (repeatable)")
@click.option("--output",  "-o", type=click.Path(), default=None)
def deploy(source, destination, port, env, verbose, dry_run, tag, output):
    """Deploy SOURCE to DESTINATION."""
    click.echo(f"Deploying {source} → {destination}")
    if tag:
        click.echo(f"Tags: {', '.join(tag)}")
```

---

## Types

```python
import click
from pathlib import Path

@click.command()
@click.option("--count",   type=int)
@click.option("--rate",    type=float)
@click.option("--flag",    type=bool)

# Choice — restricted set, shown in help
@click.option("--env",     type=click.Choice(["dev", "staging", "prod"]))

# Path — validates existence, type, permissions
@click.option("--config",  type=click.Path(exists=True, file_okay=True, dir_okay=False, readable=True))
@click.option("--output",  type=click.Path(writable=True, resolve_path=True))

# File — opens the file for you
@click.option("--input",   type=click.File("r"))
@click.option("--log",     type=click.File("a"))

# INT range
@click.option("--workers", type=click.IntRange(1, 16), default=4)
@click.option("--port",    type=click.IntRange(1024, 65535))

# FLOAT range
@click.option("--rate",    type=click.FloatRange(0.0, 1.0))

# UUID
@click.option("--job-id",  type=click.UUID)

# DateTime
@click.option("--since",   type=click.DateTime(formats=["%Y-%m-%d"]))

# Tuple
@click.option("--size",    type=(int, int), default=(800, 600))

def configure(count, rate, flag, env, config, output, input, log,
              workers, port, rate2, job_id, since, size):
    ...
```

---

## Prompts & Confirmation

```python
import click

@click.command()
@click.option("--name",     prompt="Your name",     help="Name to greet")
@click.option("--password", prompt=True, hide_input=True,
              confirmation_prompt=True, help="Your password")
def setup(name, password):
    click.echo(f"Hello {name}!")

# Manual prompt
@click.command()
def interactive():
    name  = click.prompt("Enter name", default="World")
    age   = click.prompt("Enter age",  type=int)
    if click.confirm("Are you sure?"):
        click.echo(f"Hello {name}, age {age}")
    else:
        raise click.Abort()
```

---

## Output

```python
import click
import sys

# Basic output
click.echo("Hello, World!")
click.echo(f"Value: {42}")

# Styled output
click.echo(click.style("Success!", fg="green", bold=True))
click.echo(click.style("Warning!", fg="yellow"))
click.echo(click.style("Error!",   fg="red", bold=True))

# secho — style + echo combined
click.secho("Done!", fg="green", bold=True)
click.secho("Oops!", fg="red",   err=True)    # write to stderr

# Write to stderr
click.echo("Error message", err=True)
click.echo("Error message", file=sys.stderr)

# Available colours
# black, red, green, yellow, blue, magenta, cyan, white
# bright_black, bright_red, bright_green, etc.

# Pager — show long output in a pager (like less)
with click.open_file("-") as f:
    click.echo_via_pager("Long text...\n" * 100)

# Launch editor
message = click.edit("# Enter your message\n")
```

---

## Groups — Multi-Command Apps

```python
import click

@click.group()
@click.option("--verbose", "-v", is_flag=True)
@click.pass_context
def cli(ctx, verbose):
    """My CLI application."""
    ctx.ensure_object(dict)
    ctx.obj["verbose"] = verbose

@cli.command()
@click.option("--host", default="0.0.0.0", show_default=True)
@click.option("--port", type=int, default=8000, show_default=True)
@click.option("--reload", is_flag=True)
@click.pass_context
def serve(ctx, host, port, reload):
    """Start the development server."""
    if ctx.obj["verbose"]:
        click.echo(f"Starting on {host}:{port}")
    import uvicorn
    uvicorn.run("main:app", host=host, port=port, reload=reload)

@cli.command()
@click.argument("revision", default="head")
@click.option("--dry-run", is_flag=True)
def migrate(revision, dry_run):
    """Run database migrations."""
    click.echo(f"Migrating to: {revision}")

@cli.command("create-user")
@click.argument("email")
@click.option("--admin", is_flag=True)
@click.option("--password", prompt=True, hide_input=True)
def create_user(email, admin, password):
    """Create a new user."""
    click.secho(f"✓ Created {email}", fg="green")

if __name__ == "__main__":
    cli()
```

```bash
myapp --verbose serve --port 9000 --reload
myapp migrate head
myapp create-user alice@example.com --admin
```

---

## Nested Groups

```python
import click

@click.group()
def cli(): ...

@cli.group()
def db():
    """Database management commands."""

@db.command()
def migrate():
    """Run migrations."""
    click.echo("Migrating...")

@db.command()
def reset():
    """Reset the database."""
    if click.confirm("This will delete all data. Continue?"):
        click.echo("Database reset!")

@cli.group()
def users():
    """User management commands."""

@users.command("create")
@click.argument("email")
def user_create(email):
    click.echo(f"Creating {email}")

if __name__ == "__main__":
    cli()
```

```bash
myapp db migrate
myapp db reset
myapp users create alice@example.com
```

---

## Context & Pass Object

```python
import click

@click.group()
@click.option("--config", default="config.yaml", envvar="APP_CONFIG")
@click.pass_context
def cli(ctx, config):
    ctx.ensure_object(dict)
    ctx.obj["config"] = config

@cli.command()
@click.pass_obj               # receives ctx.obj directly
def status(obj):
    click.echo(f"Config: {obj['config']}")

# Or pass_context for full context access
@cli.command()
@click.pass_context
def info(ctx):
    click.echo(f"Config: {ctx.obj['config']}")
    click.echo(f"Command: {ctx.info_name}")
```

---

## Environment Variables

```python
import click

@click.command()
@click.option("--db-url",    envvar="DATABASE_URL",  required=True)
@click.option("--secret-key",envvar="SECRET_KEY",    required=True)
@click.option("--debug",     envvar="DEBUG",         is_flag=True, default=False)
@click.option("--port",      envvar="PORT",          type=int, default=8000)
def run(db_url, secret_key, debug, port):
    """Values come from env vars if not set on CLI."""
    click.echo(f"Connecting to DB: {db_url[:20]}...")
```

```bash
DATABASE_URL=postgresql://... python main.py run
# or
python main.py run --db-url postgresql://...
```

---

## Version Option

```python
import click

@click.group()
@click.version_option(version="1.2.3", prog_name="myapp")
def cli():
    """My application."""

# Custom version message
@click.version_option(
    version="1.2.3",
    message="%(prog)s version %(version)s",
)
def cli(): ...
```

---

## Custom Decorators — Reusable Options

```python
import click
import functools

def common_options(fn):
    """Reusable decorator for options shared across commands."""
    @click.option("--verbose", "-v", is_flag=True)
    @click.option("--dry-run",       is_flag=True)
    @click.option("--config", default="config.yaml", envvar="APP_CONFIG")
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper

@cli.command()
@common_options
def deploy(verbose, dry_run, config):
    ...

@cli.command()
@common_options
def rollback(verbose, dry_run, config):
    ...
```

---

## Error Handling

```python
import click

@click.command()
@click.argument("path")
def process(path):
    try:
        result = do_work(path)
        click.echo(result)
    except FileNotFoundError:
        raise click.ClickException(f"File not found: {path}")
        # prints "Error: File not found: path" and exits 1

    except PermissionError as e:
        raise click.UsageError(f"Permission denied: {e}")
        # same format but suggests --help

# Custom exit code
@click.command()
def risky():
    click.echo("Something went wrong", err=True)
    raise click.exceptions.Exit(code=2)

# Abort — prints "Aborted!" and exits 1
raise click.Abort()
```

---

## Testing Click Apps

```python
from click.testing import CliRunner
from myapp import cli

runner = CliRunner()

def test_hello():
    result = runner.invoke(cli, ["Alice"])
    assert result.exit_code == 0
    assert "Hello, Alice!" in result.output

def test_missing_arg():
    result = runner.invoke(cli, [])
    assert result.exit_code != 0

def test_with_env():
    result = runner.invoke(cli, env={"DATABASE_URL": "sqlite:///test.db"})
    assert result.exit_code == 0

def test_prompt():
    result = runner.invoke(cli, input="Alice\n")   # simulate stdin
    assert "Hello, Alice!" in result.output

def test_file_input(tmp_path):
    f = tmp_path / "data.csv"
    f.write_text("a,b,c")
    result = runner.invoke(cli, ["--input", str(f)])
    assert result.exit_code == 0
```

---

# Rich

## What Rich Provides

```python
from rich import print as rprint              # drop-in print with markup
from rich.console import Console             # main output interface
from rich.table import Table                 # formatted tables
from rich.progress import Progress, track    # progress bars
from rich.syntax import Syntax               # syntax highlighting
from rich.markdown import Markdown           # render markdown
from rich.panel import Panel                 # boxed content
from rich.tree import Tree                   # tree structures
from rich.columns import Columns             # multi-column layout
from rich.prompt import Prompt, Confirm      # styled prompts
from rich.logging import RichHandler         # rich log handler
from rich.traceback import install           # pretty tracebacks
from rich.pretty import pprint               # pretty-print objects
from rich.inspect import inspect             # inspect objects
```

---

## Console — The Core Object

```python
from rich.console import Console

console = Console()

# Basic output
console.print("Hello, [bold]World[/bold]!")
console.print("[red]Error:[/red] Something went wrong")
console.print("[green]✓[/green] Task completed")

# stderr
err_console = Console(stderr=True)
err_console.print("[red]Error![/red]")

# Write to file
file_console = Console(file=open("output.txt", "w"))
file_console.print("Written to file")

# Force colour even when piped (e.g. in CI)
console = Console(force_terminal=True)

# Disable colour (plain text)
console = Console(no_color=True)

# Width control
console = Console(width=120)
```

---

## Markup & Styles

```python
from rich.console import Console

console = Console()

# Markup tags
console.print("[bold]bold[/bold]")
console.print("[italic]italic[/italic]")
console.print("[underline]underline[/underline]")
console.print("[strike]strikethrough[/strike]")
console.print("[bold red]bold red[/bold red]")
console.print("[on blue]blue background[/on blue]")
console.print("[bold white on red]white on red[/bold white on red]")

# 256-colour
console.print("[color(208)]orange text[/color(208)]")

# Hex colour
console.print("[#ff6600]hex colour[/#ff6600]")

# Escape markup
console.print(r"\[not markup\]")
console.print("[bold]real bold[/bold] [\\[escaped\\]]")

# Emoji
console.print(":white_check_mark: Done!")
console.print(":warning: Warning!")
console.print(":rocket: Deploying...")
```

---

## Tables

```python
from rich.console import Console
from rich.table import Table

console = Console()

table = Table(
    title="Users",
    show_header=True,
    header_style="bold magenta",
    border_style="blue",
    row_styles=["", "dim"],             # alternate row styles
)

table.add_column("ID",    style="cyan",  no_wrap=True, width=5)
table.add_column("Name",  style="green")
table.add_column("Email", style="white")
table.add_column("Role",  style="yellow", justify="center")
table.add_column("Status")

table.add_row("1", "Alice", "alice@example.com", "admin",  "[green]Active[/green]")
table.add_row("2", "Bob",   "bob@example.com",   "user",   "[yellow]Pending[/yellow]")
table.add_row("3", "Carol", "carol@example.com", "user",   "[red]Banned[/red]")

console.print(table)
```

---

## Progress Bars

```python
from rich.progress import track
from rich.progress import (
    Progress,
    SpinnerColumn,
    BarColumn,
    TextColumn,
    TimeElapsedColumn,
    TimeRemainingColumn,
    MofNCompleteColumn,
    TaskProgressColumn,
)
import time

# Simple — wrap any iterable
for item in track(items, description="Processing..."):
    process(item)

# Multiple tasks with full control
with Progress(
    SpinnerColumn(),
    TextColumn("[progress.description]{task.description}"),
    BarColumn(),
    TaskProgressColumn(),
    MofNCompleteColumn(),
    TimeElapsedColumn(),
    TimeRemainingColumn(),
) as progress:

    task1 = progress.add_task("[red]Downloading...",  total=100)
    task2 = progress.add_task("[green]Processing...", total=200)
    task3 = progress.add_task("[blue]Uploading...",   total=50)

    while not progress.finished:
        progress.update(task1, advance=1)
        progress.update(task2, advance=2)
        progress.update(task3, advance=0.5)
        time.sleep(0.02)
```

---

## Panels & Layout

```python
from rich.console import Console
from rich.panel import Panel
from rich.columns import Columns
from rich.align import Align

console = Console()

# Boxed panel
console.print(Panel("Hello, World!", title="Greeting", border_style="green"))
console.print(Panel.fit("[bold]Compact panel[/bold]", border_style="blue"))

# Columns layout
items = [Panel(f"Item {i}", width=20) for i in range(1, 7)]
console.print(Columns(items))

# Centered
console.print(Align.center("[bold]Centered text[/bold]"))
```

---

## Syntax Highlighting

```python
from rich.syntax import Syntax
from rich.console import Console

console = Console()

code = '''
def greet(name: str) -> str:
    return f"Hello, {name}!"
'''

syntax = Syntax(code, "python", theme="monokai", line_numbers=True)
console.print(syntax)

# From file
syntax = Syntax.from_path("main.py", line_numbers=True, theme="github-dark")
console.print(syntax)
```

---

## Markdown

```python
from rich.markdown import Markdown
from rich.console import Console

console = Console()

md = Markdown("""
# Title

This is **bold** and *italic* text.

## Code


print("Hello!")


- Item 1
- Item 2
- Item 3 """)

console.print(md)
```

---

## Tree

```python
from rich.tree import Tree
from rich.console import Console

console = Console()

tree = Tree("[bold]myapp/[/bold]")

src = tree.add("[bold]src/[/bold]")
src.add("main.py")
src.add("config.py")
routers = src.add("[bold]routers/[/bold]")
routers.add("users.py")
routers.add("items.py")

tree.add("[bold]tests/[/bold]").add("test_main.py")
tree.add("pyproject.toml")
tree.add(".env.example")

console.print(tree)
````

---

## Pretty Printing & Inspection

```python
from rich.pretty import pprint
from rich.inspect import inspect
from rich.console import Console

console = Console()

# Pretty print any Python object
data = {"users": [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]}
pprint(data)

# Inspect an object — show all attributes and methods
inspect(data, methods=True, docs=False)
inspect(str, methods=True)

# Print with syntax highlighting
console.print(data)     # auto-pretty-prints dicts/lists
console.print_json('{"key": "value", "number": 42}')
```

---

## Rich Logging

```python
import logging
from rich.logging import RichHandler
from rich.console import Console

logging.basicConfig(
    level=logging.DEBUG,
    format="%(message)s",
    datefmt="[%X]",
    handlers=[RichHandler(
        rich_tracebacks=True,
        tracebacks_show_locals=True,
        show_path=True,
    )],
)

logger = logging.getLogger("myapp")
logger.debug("Debug message")
logger.info("Info message")
logger.warning("Warning message")
logger.error("Error message")

try:
    1 / 0
except ZeroDivisionError:
    logger.exception("Division error")
```

---

## Pretty Tracebacks

```python
from rich.traceback import install

# Install globally — all uncaught exceptions use rich formatting
install(
    show_locals=True,           # show local variables in each frame
    width=120,
    extra_lines=3,
    theme="monokai",
    word_wrap=True,
)

# Or handle manually
from rich.console import Console
console = Console()

try:
    1 / 0
except Exception:
    console.print_exception(show_locals=True)
```

---

## Prompts

```python
from rich.prompt import Prompt, Confirm, IntPrompt, FloatPrompt

name    = Prompt.ask("Enter your name", default="World")
age     = IntPrompt.ask("Enter your age", default=25)
rate    = FloatPrompt.ask("Enter rate", default=0.5)
proceed = Confirm.ask("Continue?", default=True)

# With choices
env = Prompt.ask(
    "Select environment",
    choices=["dev", "staging", "prod"],
    default="dev",
)
```

---

## Click + Rich — Together

The most common pattern — Click handles argument parsing, Rich handles output.

```python
import click
from rich.console import Console
from rich.table import Table
from rich.progress import track
from rich.panel import Panel

console = Console()

@click.group()
def cli():
    """My CLI — powered by Click + Rich."""

@cli.command()
@click.option("--limit", "-n", type=int, default=10, show_default=True)
def list_users(limit):
    """List users in a formatted table."""
    users = fetch_users(limit)

    table = Table(title=f"Users (showing {len(users)})", border_style="blue")
    table.add_column("ID",    style="cyan", width=6)
    table.add_column("Name",  style="green")
    table.add_column("Email")
    table.add_column("Role",  justify="center")

    for user in users:
        table.add_row(str(user["id"]), user["name"], user["email"], user["role"])

    console.print(table)

@cli.command()
@click.argument("files", nargs=-1, required=True)
def process(files):
    """Process FILES with a progress bar."""
    for f in track(files, description="[green]Processing..."):
        do_work(f)
    console.print(Panel("[bold green]✓ All done![/bold green]"))

@cli.command()
@click.argument("path")
def show_code(path):
    """Display a file with syntax highlighting."""
    from rich.syntax import Syntax
    syntax = Syntax.from_path(path, line_numbers=True, theme="monokai")
    console.print(syntax)

if __name__ == "__main__":
    cli()
```

---

## Quick Reference — Click

|Task|Code|
|---|---|
|Basic command|`@click.command()`|
|Argument|`@click.argument("name")`|
|Option|`@click.option("--name", "-n", default="val")`|
|Flag|`@click.option("--verbose", is_flag=True)`|
|Multiple values|`@click.option("--tag", multiple=True)`|
|Required option|`@click.option("--key", required=True)`|
|Choices|`type=click.Choice(["a","b","c"])`|
|Path|`type=click.Path(exists=True)`|
|File|`type=click.File("r")`|
|Range|`type=click.IntRange(1, 10)`|
|Env var|`envvar="MY_VAR"`|
|Prompt|`prompt=True`|
|Hide input|`hide_input=True`|
|Confirm|`click.confirm("Sure?")`|
|Group|`@click.group()`|
|Nested group|`@parent_group.group()`|
|Pass context|`@click.pass_context` / `@click.pass_obj`|
|Styled echo|`click.secho("msg", fg="green")`|
|Error|`raise click.ClickException("msg")`|
|Version|`@click.version_option(version="1.0")`|
|Test|`CliRunner().invoke(cli, ["args"])`|

## Quick Reference — Rich

|Task|Code|
|---|---|
|Console|`Console()`|
|Print markup|`console.print("[bold red]text[/bold red]")`|
|Print to stderr|`Console(stderr=True)`|
|Table|`Table(); table.add_column(); table.add_row()`|
|Simple progress|`for x in track(items, description="...")`|
|Multi progress|`with Progress(...) as p: p.add_task(...)`|
|Panel|`Panel("content", title="Title")`|
|Syntax highlight|`Syntax(code, "python", theme="monokai")`|
|Markdown|`Markdown("# Title\n...")`|
|Tree|`Tree("root"); tree.add("branch")`|
|Pretty print|`pprint(obj)`|
|Inspect object|`inspect(obj, methods=True)`|
|JSON output|`console.print_json(json_str)`|
|Rich logging|`RichHandler(rich_tracebacks=True)`|
|Pretty tracebacks|`from rich.traceback import install; install()`|
|Prompts|`Prompt.ask("Question", default="val")`|
|Confirm|`Confirm.ask("Sure?")`|
|Emoji|`console.print(":rocket: Deploying!")`|
|Columns|`Columns([panel1, panel2, panel3])`|

---
## Tags

#python #click #rich #cli #terminal #output #styling #backend #devtools