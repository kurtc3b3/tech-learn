
## What & When

**pathlib** is Python's built-in object-oriented filesystem path library, introduced in Python 3.4 and the recommended way to work with file paths since Python 3.6. It replaces scattered `os.path` string operations with a clean, chainable, cross-platform `Path` object.

Use pathlib when:

- Reading, writing, or navigating the filesystem
- Building file paths in a cross-platform way
- Globbing for files matching a pattern
- Checking file/directory existence, type, or metadata
- Walking directory trees

```python
from pathlib import Path
```

No installation needed — pathlib is part of the Python standard library.

---

## pathlib vs os.path

|Task|os.path|pathlib|
|---|---|---|
|Join paths|`os.path.join(a, b, c)`|`a / b / c`|
|File exists|`os.path.exists(p)`|`p.exists()`|
|Is file|`os.path.isfile(p)`|`p.is_file()`|
|Is dir|`os.path.isdir(p)`|`p.is_dir()`|
|Filename|`os.path.basename(p)`|`p.name`|
|Directory|`os.path.dirname(p)`|`p.parent`|
|Extension|`os.path.splitext(p)[1]`|`p.suffix`|
|Absolute path|`os.path.abspath(p)`|`p.resolve()`|
|Home dir|`os.path.expanduser("~")`|`Path.home()`|

---

## Creating Paths

```python
from pathlib import Path

# From string
p = Path("/home/user/documents/report.pdf")
p = Path("relative/path/to/file.txt")

# Current directory
p = Path(".")
p = Path.cwd()              # absolute current working directory

# Home directory
p = Path.home()             # /home/user  (or C:\Users\user on Windows)

# Joining with /  operator
base  = Path("/home/user")
docs  = base / "documents"
report = docs / "report.pdf"

# From parts
p = Path("/", "home", "user", "documents", "report.pdf")

# Platform-specific
from pathlib import PurePosixPath, PureWindowsPath
p = PurePosixPath("/home/user/file.txt")
p = PureWindowsPath("C:/Users/user/file.txt")
```

---

## Path Properties

```python
p = Path("/home/user/documents/report.pdf")

p.name          # "report.pdf"          — filename with extension
p.stem          # "report"              — filename without extension
p.suffix        # ".pdf"               — last extension
p.suffixes      # [".pdf"]             — all extensions e.g. [".tar", ".gz"]
p.parent        # Path("/home/user/documents")
p.parents       # sequence: [documents, user, home, /]
p.parts         # ("/", "home", "user", "documents", "report.pdf")
p.root          # "/"
p.anchor        # "/"  (root + drive on Windows)
p.drive         # ""   ("C:" on Windows)

str(p)          # "/home/user/documents/report.pdf"
p.as_posix()    # "/home/user/documents/report.pdf"  (always forward slashes)
p.as_uri()      # "file:///home/user/documents/report.pdf"
```

---

## Checking State

```python
p = Path("/home/user/documents/report.pdf")

p.exists()          # True if path exists (file or directory)
p.is_file()         # True if it is a regular file
p.is_dir()          # True if it is a directory
p.is_symlink()      # True if it is a symbolic link
p.is_absolute()     # True if path is absolute
p.is_relative_to("/home/user")  # True (Python 3.9+)

# Stat info
stat = p.stat()
stat.st_size        # file size in bytes
stat.st_mtime       # last modified time (Unix timestamp)
stat.st_ctime       # created / metadata change time
stat.st_mode        # file mode / permissions

import os
from datetime import datetime
modified = datetime.fromtimestamp(p.stat().st_mtime)
```

---

## Reading & Writing Files

```python
p = Path("data/notes.txt")

# Read
text  = p.read_text(encoding="utf-8")          # entire file as string
raw   = p.read_bytes()                          # entire file as bytes

# Write (overwrites)
p.write_text("Hello, pathlib!", encoding="utf-8")
p.write_bytes(b"\x00\x01\x02")

# Open (standard file handle — for streaming or appending)
with p.open("r", encoding="utf-8") as f:
    content = f.read()

with p.open("a", encoding="utf-8") as f:
    f.write("appended line\n")

with p.open("rb") as f:
    data = f.read()
```

---

## Creating Directories & Files

```python
p = Path("output/reports/2026")

# Create directory (and parents)
p.mkdir(parents=True, exist_ok=True)

# Create a single directory
Path("logs").mkdir(exist_ok=True)

# Create an empty file
Path("output/.gitkeep").touch()

# Create file and parent dirs
file = Path("output/reports/2026/report.pdf")
file.parent.mkdir(parents=True, exist_ok=True)
file.write_text("content")
```

---

## Copying, Moving & Deleting

```python
import shutil
from pathlib import Path

src  = Path("data/report.pdf")
dest = Path("backup/report.pdf")

# Copy file
shutil.copy2(src, dest)         # copy with metadata
shutil.copy(src, dest)          # copy without metadata

# Copy directory tree
shutil.copytree("src_dir", "dest_dir")

# Move / rename
src.rename(dest)                # move within same filesystem
src.replace(dest)               # move, overwriting destination if exists
shutil.move(str(src), str(dest))  # cross-filesystem move

# Delete file
src.unlink()                    # raises if not found
src.unlink(missing_ok=True)     # Python 3.8+ — no error if missing

# Delete empty directory
Path("empty_dir").rmdir()

# Delete directory tree
shutil.rmtree("dir_to_delete")
```

---

## Globbing — Finding Files

```python
base = Path("src")

# All Python files (non-recursive)
list(base.glob("*.py"))

# All Python files (recursive)
list(base.rglob("*.py"))

# Specific pattern
list(base.glob("**/*.json"))        # same as rglob("*.json")
list(base.glob("tests/test_*.py"))

# All files in directory
list(base.glob("*"))

# All items recursively
list(base.rglob("*"))

# Only files (filter)
files = [p for p in base.rglob("*") if p.is_file()]

# Only directories
dirs = [p for p in base.rglob("*") if p.is_dir()]

# Sort results
sorted(base.rglob("*.py"))
```

---

## Iterating a Directory

```python
base = Path("src")

# Iterate direct children
for item in base.iterdir():
    print(item.name, "dir" if item.is_dir() else "file")

# Iterate with filter
for f in base.iterdir():
    if f.is_file() and f.suffix == ".py":
        print(f)
```

---

## Path Manipulation

```python
p = Path("/home/user/documents/report.pdf")

# Change name / suffix
p.with_name("summary.pdf")             # /home/user/documents/summary.pdf
p.with_stem("summary")                 # /home/user/documents/summary.pdf (3.9+)
p.with_suffix(".txt")                  # /home/user/documents/report.txt
p.with_suffix("")                      # /home/user/documents/report

# Resolve — make absolute, resolve symlinks
Path("../sibling").resolve()           # absolute path

# Relative to
p.relative_to("/home/user")            # documents/report.pdf

# Make relative path absolute
(Path.cwd() / "relative/path").resolve()
```

---

## Common Path Patterns

```python
from pathlib import Path

# Project root (common in FastAPI / Python packages)
ROOT     = Path(__file__).parent.parent   # two levels up from current file
SRC      = ROOT / "src"
TEMPLATES = ROOT / "templates"
STATIC   = ROOT / "static"
DATA     = ROOT / "data"
LOGS     = ROOT / "logs"

# Config file next to current script
CONFIG = Path(__file__).parent / "config.yaml"

# User data directory
import platformdirs   # pip install platformdirs
data_dir = Path(platformdirs.user_data_dir("MyApp", "MyCompany"))
data_dir.mkdir(parents=True, exist_ok=True)

# Temp directory
import tempfile
with tempfile.TemporaryDirectory() as tmp:
    tmp_path = Path(tmp)
    (tmp_path / "work.txt").write_text("temporary")
```

---

## Working with File Metadata

```python
from pathlib import Path
from datetime import datetime
import os

p = Path("data/report.pdf")

# Size
size_bytes = p.stat().st_size
size_mb    = size_bytes / (1024 * 1024)

# Timestamps
modified = datetime.fromtimestamp(p.stat().st_mtime)
accessed = datetime.fromtimestamp(p.stat().st_atime)

# Permissions (Unix)
oct(p.stat().st_mode)           # e.g. '0o100644'
p.chmod(0o755)                  # change permissions

# Owner (Unix)
p.owner()                       # username string
p.group()                       # group string

# Symlinks
link = Path("link_to_report")
link.symlink_to(p)              # create symlink
link.resolve()                  # follow to real path
link.readlink()                 # just the link target (Python 3.9+)
```

---

## Sorting & Filtering Files

```python
from pathlib import Path

base = Path("data")

# Sort by name
sorted(base.glob("*.csv"), key=lambda p: p.name)

# Sort by modification time (newest first)
sorted(base.glob("*.log"), key=lambda p: p.stat().st_mtime, reverse=True)

# Sort by file size (largest first)
sorted(base.rglob("*.pdf"), key=lambda p: p.stat().st_size, reverse=True)

# Filter by size > 1MB
large = [p for p in base.rglob("*") if p.is_file() and p.stat().st_size > 1_048_576]

# Filter by extension
csvs = [p for p in base.rglob("*") if p.suffix in {".csv", ".tsv"}]

# Most recently modified file
latest = max(base.glob("*.log"), key=lambda p: p.stat().st_mtime)
```

---

## Paths in FastAPI

```python
from pathlib import Path
from fastapi import FastAPI, UploadFile
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

BASE_DIR     = Path(__file__).parent
STATIC_DIR   = BASE_DIR / "static"
TEMPLATE_DIR = BASE_DIR / "templates"
UPLOAD_DIR   = BASE_DIR / "uploads"

UPLOAD_DIR.mkdir(exist_ok=True)

app = FastAPI()
app.mount("/static", StaticFiles(directory=STATIC_DIR), name="static")
templates = Jinja2Templates(directory=TEMPLATE_DIR)

@app.post("/upload")
async def upload_file(file: UploadFile):
    dest = UPLOAD_DIR / file.filename
    dest.write_bytes(await file.read())
    return {"saved": str(dest)}
```

---

## Reading Config / Data Files

```python
from pathlib import Path
import json, tomllib, yaml

base = Path(__file__).parent

# JSON
config = json.loads((base / "config.json").read_text())

# TOML (Python 3.11+)
with (base / "pyproject.toml").open("rb") as f:
    config = tomllib.load(f)

# YAML — pip install pyyaml
import yaml
config = yaml.safe_load((base / "config.yaml").read_text())

# .env — pip install python-dotenv
from dotenv import dotenv_values
env = dotenv_values(base / ".env")
```

---

## Quick Reference

|Task|Code|
|---|---|
|Create path|`Path("/some/path")`|
|Join|`base / "subdir" / "file.txt"`|
|Current dir|`Path.cwd()`|
|Home dir|`Path.home()`|
|Filename|`p.name`|
|Stem (no ext)|`p.stem`|
|Extension|`p.suffix`|
|Parent dir|`p.parent`|
|Exists|`p.exists()`|
|Is file|`p.is_file()`|
|Is dir|`p.is_dir()`|
|Read text|`p.read_text(encoding="utf-8")`|
|Write text|`p.write_text("content")`|
|Read bytes|`p.read_bytes()`|
|Write bytes|`p.write_bytes(b"...")`|
|Make dir|`p.mkdir(parents=True, exist_ok=True)`|
|Delete file|`p.unlink(missing_ok=True)`|
|Delete dir|`shutil.rmtree(p)`|
|Rename/move|`p.rename(dest)`|
|Glob|`p.glob("*.py")`|
|Recursive glob|`p.rglob("*.py")`|
|List dir|`p.iterdir()`|
|Absolute path|`p.resolve()`|
|Change suffix|`p.with_suffix(".txt")`|
|Change name|`p.with_name("new.txt")`|
|File size|`p.stat().st_size`|
|Modified time|`p.stat().st_mtime`|

---
## Tags

#python #pathlib #filesystem #files #stdlib #backend