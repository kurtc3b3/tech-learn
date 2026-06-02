
## What & When

**re** is Python's built-in regular expression module. It provides pattern matching, searching, splitting, and substitution on strings using a concise pattern language.

Use `re` when:

- Validating formats (email, phone, postcode, SKU)
- Extracting structured data from unstructured text
- Parsing logs, config files, or markup
- Find-and-replace with pattern awareness
- Tokenising text

```python
import re
```

No installation needed — `re` is part of the standard library.

---

## Core Functions

```python
import re

pattern = r"\d{3}-\d{4}"
text    = "Call us on 020-1234 or 020-5678"

re.match(pattern, text)         # match at START of string only → Match | None
re.search(pattern, text)        # first match anywhere → Match | None
re.findall(pattern, text)       # all non-overlapping matches → list[str]
re.finditer(pattern, text)      # all matches as iterator → Iterator[Match]
re.fullmatch(pattern, text)     # entire string must match → Match | None
re.sub(pattern, repl, text)     # replace matches → str
re.subn(pattern, repl, text)    # replace + count → (str, int)
re.split(pattern, text)         # split on matches → list[str]
re.compile(pattern)             # compile to reusable Pattern object
re.escape(text)                 # escape special chars in literal string
```

---

## `match` vs `search` vs `fullmatch`

```python
import re

text = "hello world 123"

re.match(r"\d+", text)          # None — digits not at start
re.search(r"\d+", text)         # Match("123") — found anywhere
re.fullmatch(r"\d+", text)      # None — whole string isn't digits
re.fullmatch(r"[\w\s]+", text)  # Match — whole string matches
```

> [!tip] Prefer `re.search` over `re.match` `re.match` only checks the start of the string — a common source of bugs when the intent is to find a pattern anywhere. Use `re.fullmatch` for strict whole-string validation.

---

## Flags

```python
import re

re.search(r"hello", "Hello World", re.IGNORECASE)   # case-insensitive
re.search(r"hello", "Hello World", re.I)             # shorthand

# Multiple flags
re.search(r"^hello", text, re.IGNORECASE | re.MULTILINE)

# Inline flags (inside the pattern)
re.search(r"(?i)hello", "Hello World")               # case-insensitive
re.search(r"(?im)^hello", text)                      # inline multiple
```

|Flag|Short|Description|
|---|---|---|
|`re.IGNORECASE`|`re.I`|Case-insensitive matching|
|`re.MULTILINE`|`re.M`|`^` and `$` match line boundaries|
|`re.DOTALL`|`re.S`|`.` matches newlines too|
|`re.VERBOSE`|`re.X`|Allow whitespace and comments in pattern|
|`re.ASCII`|`re.A`|`\w`, `\d`, etc. match ASCII only|
|`re.UNICODE`|`re.U`|`\w`, `\d` match Unicode (default)|
|`re.LOCALE`|`re.L`|Use locale for `\w` etc.|

---

## Pattern Syntax

### Character Classes

```
.           any character except newline (use re.DOTALL for newline)
\d          digit [0-9]
\D          non-digit
\w          word character [a-zA-Z0-9_]
\W          non-word character
\s          whitespace [ \t\n\r\f\v]
\S          non-whitespace
\b          word boundary
\B          non-word boundary
[abc]       character class — a, b, or c
[^abc]      negated class — anything except a, b, c
[a-z]       range — lowercase a to z
[a-zA-Z0-9] alphanumeric
```

### Anchors

```
^           start of string (or line with re.MULTILINE)
$           end of string (or line with re.MULTILINE)
\A          absolute start of string (ignores re.MULTILINE)
\Z          absolute end of string
\b          word boundary
```

### Quantifiers

```
*           0 or more (greedy)
+           1 or more (greedy)
?           0 or 1 (greedy)
{n}         exactly n
{n,}        n or more
{n,m}       between n and m (inclusive)
*?          0 or more (lazy — as few as possible)
+?          1 or more (lazy)
??          0 or 1 (lazy)
{n,m}?      between n and m (lazy)
```

### Groups

```
(abc)       capturing group
(?:abc)     non-capturing group
(?P<name>)  named capturing group
(?P=name)   backreference to named group
\1          backreference to group 1
|           alternation — a|b means a or b
```

### Lookahead & Lookbehind

```
(?=abc)     positive lookahead  — followed by abc
(?!abc)     negative lookahead  — NOT followed by abc
(?<=abc)    positive lookbehind — preceded by abc
(?<!abc)    negative lookbehind — NOT preceded by abc
```

---

## Match Object

```python
import re

m = re.search(r"(\d{4})-(\d{2})-(\d{2})", "Date: 2026-05-31")

if m:
    m.group()       # "2026-05-31"  — entire match
    m.group(0)      # "2026-05-31"  — same as group()
    m.group(1)      # "2026"        — first capture group
    m.group(2)      # "05"
    m.group(3)      # "31"
    m.groups()      # ("2026", "05", "31")

    m.start()       # 6  — start index of match
    m.end()         # 16 — end index of match
    m.span()        # (6, 16)

    m.string        # "Date: 2026-05-31" — original string
```

---

## Named Groups

```python
import re

pattern = r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})"
m = re.search(pattern, "Date: 2026-05-31")

if m:
    m.group("year")     # "2026"
    m.group("month")    # "05"
    m.group("day")      # "31"
    m.groupdict()       # {"year": "2026", "month": "05", "day": "31"}
```

---

## `findall` & `finditer`

```python
import re

text = "Prices: £9.99, £14.50, £3.00"

# findall — returns list of strings (or tuples with groups)
prices = re.findall(r"£(\d+\.\d{2})", text)
print(prices)   # ["9.99", "14.50", "3.00"]

# findall with multiple groups — returns list of tuples
dates = re.findall(r"(\d{4})-(\d{2})-(\d{2})", "2026-01-01 and 2026-12-31")
print(dates)    # [("2026", "01", "01"), ("2026", "12", "31")]

# finditer — returns iterator of Match objects (memory efficient)
for m in re.finditer(r"£(\d+\.\d{2})", text):
    print(m.group(1), "at position", m.start())
```

---

## `sub` & `subn` — Replace

```python
import re

# Simple replacement
result = re.sub(r"\s+", " ", "hello    world\t\there")
# "hello world there"

# Replace with backreference
result = re.sub(r"(\w+)\s(\w+)", r"\2 \1", "hello world")
# "world hello"

# Named group backreference
result = re.sub(r"(?P<year>\d{4})-(?P<month>\d{2})", r"\g<month>/\g<year>", "2026-05")
# "05/2026"

# Replace with a function
def redact(m):
    return "*" * len(m.group())

result = re.sub(r"\b\d{4}\b", redact, "Card: 1234 5678 9012 3456")
# "Card: **** **** **** ****"

# Limit replacements
result = re.sub(r"\d+", "X", "1 2 3 4 5", count=3)
# "X X X 4 5"

# subn — returns (result, count)
result, n = re.subn(r"\d+", "X", "1 2 3")
# ("X X X", 3)
```

---

## `split`

```python
import re

# Split on pattern
re.split(r"\s+", "hello   world\tthere")
# ["hello", "world", "there"]

# Split with capturing group — delimiters included in result
re.split(r"(\s+)", "hello   world")
# ["hello", "   ", "world"]

# Limit splits
re.split(r"\s+", "a b c d e", maxsplit=2)
# ["a", "b", "c d e"]

# Split on multiple delimiters
re.split(r"[,;:\s]+", "a, b; c: d e")
# ["a", "b", "c", "d", "e"]
```

---

## Compiled Patterns — `re.compile`

Compile once, use many times. Faster in loops and provides the same methods.

```python
import re

# Compile once at module level
EMAIL_RE  = re.compile(r"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$")
PHONE_RE  = re.compile(r"^\+?[\d\s\-().]{7,20}$")
POSTCODE_RE = re.compile(r"^[A-Z]{1,2}\d[A-Z\d]?\s*\d[A-Z]{2}$", re.IGNORECASE)

# Reuse — same methods as re module
EMAIL_RE.match("alice@example.com")
EMAIL_RE.search(text)
EMAIL_RE.findall(text)
EMAIL_RE.sub("***", text)
EMAIL_RE.split(text)
```

---

## `re.VERBOSE` — Readable Patterns

```python
import re

# Without VERBOSE — hard to read
DATE_RE = re.compile(r"(\d{4})-(\d{2})-(\d{2})")

# With VERBOSE — whitespace and comments ignored
DATE_RE = re.compile(r"""
    (\d{4})     # year
    -           # separator
    (\d{2})     # month
    -           # separator
    (\d{2})     # day
""", re.VERBOSE)

# Combine with other flags
PATTERN = re.compile(r"""
    ^\s*            # optional leading whitespace
    (?P<key>\w+)    # key
    \s*=\s*         # equals sign with optional spaces
    (?P<value>.+)   # value
    \s*$            # optional trailing whitespace
""", re.VERBOSE | re.MULTILINE)
```

---

## Greedy vs Lazy

```python
import re

html = "<b>bold</b> and <i>italic</i>"

# Greedy — matches as much as possible
re.findall(r"<.+>", html)
# ["<b>bold</b> and <i>italic</i>"]   ← one big match

# Lazy — matches as little as possible
re.findall(r"<.+?>", html)
# ["<b>", "</b>", "<i>", "</i>"]      ← each tag separately

# Another example
text = '"first" and "second"'
re.findall(r'".*"',  text)    # ['"first" and "second"']  greedy
re.findall(r'".*?"', text)    # ['"first"', '"second"']   lazy
```

---

## Lookahead & Lookbehind

```python
import re

# Positive lookahead — match word followed by "px"
re.findall(r"\d+(?=px)", "width: 100px height: 200px margin: 5em")
# ["100", "200"]

# Negative lookahead — match digits NOT followed by "px"
re.findall(r"\d+(?!px)\b", "width: 100px count: 42")
# ["42"]

# Positive lookbehind — match digits preceded by "£"
re.findall(r"(?<=£)\d+\.\d{2}", "£9.99 and $4.50 and £14.99")
# ["9.99", "14.99"]

# Negative lookbehind — match digits NOT preceded by "£"
re.findall(r"(?<!£)\d+\.\d{2}", "£9.99 and $4.50")
# ["4.50"]

# Password validation — lookaheads for multiple rules
strong_password = re.compile(r"""
    ^
    (?=.*[a-z])         # at least one lowercase
    (?=.*[A-Z])         # at least one uppercase
    (?=.*\d)            # at least one digit
    (?=.*[!@#$%^&*])    # at least one special char
    .{8,}               # minimum 8 chars
    $
""", re.VERBOSE)

strong_password.match("MyPass1!")     # Match
strong_password.match("weakpass")     # None
```

---

## Common Patterns

```python
import re

# Email
EMAIL = re.compile(r"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$")

# URL
URL = re.compile(
    r"https?://(?:[-\w.]|(?:%[\da-fA-F]{2}))+"
    r"(?:/[-\w._~:/?#\[\]@!$&'()*+,;=%]*)?"
)

# UK postcode
UK_POSTCODE = re.compile(r"^[A-Z]{1,2}\d[A-Z\d]?\s*\d[A-Z]{2}$", re.I)

# IPv4 address
IPV4 = re.compile(
    r"^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}"
    r"(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$"
)

# ISO 8601 date
ISO_DATE = re.compile(r"^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$")

# UK phone
UK_PHONE = re.compile(r"^(?:0|\+44)\s?[\d\s]{9,13}$")

# Slug
SLUG = re.compile(r"^[a-z0-9]+(?:-[a-z0-9]+)*$")

# Hex colour
HEX_COLOUR = re.compile(r"^#(?:[0-9a-fA-F]{3}){1,2}$")

# Credit card (basic)
CREDIT_CARD = re.compile(r"^\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}$")

# JWT
JWT = re.compile(r"^[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+$")
```

---

## Log Parsing Example

```python
import re
from datetime import datetime

LOG_LINE = re.compile(r"""
    (?P<timestamp>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})
    \s+
    \[(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\]
    \s+
    (?P<logger>[\w.]+)
    :\s+
    (?P<message>.+)
""", re.VERBOSE)

log = "2026-05-31T12:00:01 [ERROR] myapp.users: Failed to send email"

m = LOG_LINE.match(log)
if m:
    entry = m.groupdict()
    entry["timestamp"] = datetime.fromisoformat(entry["timestamp"])
    print(entry)
    # {"timestamp": datetime(...), "level": "ERROR",
    #  "logger": "myapp.users", "message": "Failed to send email"}
```

---

## Performance Tips

```python
import re

# 1. Compile at module level — not inside loops
PATTERN = re.compile(r"\d{4}-\d{2}-\d{2}")   # ✅ once

def process(text):
    return PATTERN.findall(text)               # ✅ reuse

# 2. Use re.search not re.match when you want anywhere match
# 3. Avoid catastrophic backtracking — possessive quantifiers not in stdlib;
#    use atomic groups via workaround or simplify the pattern
# 4. Use non-capturing groups (?:) when you don't need the group value
# 5. Use re.findall only when you need all results at once;
#    re.finditer is memory efficient for large texts
# 6. Anchor patterns when possible — ^ and $ reduce backtracking

# Cache is automatic — re module caches last 512 compiled patterns
# But explicit re.compile is still clearer and faster in tight loops
```

---

## `re` with Pydantic

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    sku:      str = Field(..., pattern=r"^[A-Z]{3}-\d{4}$")
    postcode: str = Field(..., pattern=r"^[A-Z]{1,2}\d[A-Z\d]?\s*\d[A-Z]{2}$")
    slug:     str = Field(..., pattern=r"^[a-z0-9]+(?:-[a-z0-9]+)*$")
```

Pydantic's `Field(pattern=...)` accepts raw regex strings and validates at model instantiation — no manual `re.match` needed.

---

## Quick Reference

|Task|Code|
|---|---|
|Match at start|`re.match(pattern, text)`|
|Match anywhere|`re.search(pattern, text)`|
|Whole string|`re.fullmatch(pattern, text)`|
|All matches|`re.findall(pattern, text)`|
|Iterator of matches|`re.finditer(pattern, text)`|
|Replace|`re.sub(pattern, repl, text)`|
|Replace with fn|`re.sub(pattern, lambda m: ..., text)`|
|Replace n times|`re.sub(pattern, repl, text, count=n)`|
|Split|`re.split(pattern, text)`|
|Compile|`re.compile(pattern, flags)`|
|Case-insensitive|`re.I` or `(?i)` inline|
|Multiline|`re.M`|
|Dot matches all|`re.S`|
|Readable pattern|`re.X` + comments|
|Match group|`m.group(1)`|
|Named group|`m.group("name")` / `m.groupdict()`|
|All groups|`m.groups()`|
|Match position|`m.start()`, `m.end()`, `m.span()`|
|Escape literal|`re.escape(string)`|
|Lazy quantifier|`*?`, `+?`, `??`|
|Lookahead|`(?=...)` / `(?!...)`|
|Lookbehind|`(?<=...)` / `(?<!...)`|
|Non-capturing|`(?:...)`|
|Named group|`(?P<name>...)`|

---

## Tags

#python #regex #re #pattern-matching #parsing #validation #stdlib