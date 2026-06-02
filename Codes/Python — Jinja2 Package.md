
## What is Jinja2?

**Jinja2** is a fast, expressive, and sandboxable templating engine for Python. It renders text-based output — HTML, XML, email, Markdown, config files — by combining a **template** (markup + logic) with a **context** (Python data).

```bash
pip install jinja2
```

Used by: FastAPI, Flask, Django (optional), Ansible, Cookiecutter, MkDocs.

---

## Core Concepts

|Concept|Description|
|---|---|
|**Environment**|Central config object — loaders, extensions, globals|
|**Template**|Text file with Jinja2 syntax|
|**Context**|Dict of variables passed to the template|
|**Loader**|Tells Jinja2 where to find templates|
|**Filter**|Transform a value — `{{ name \| upper }}`|
|**Test**|Boolean check — `{% if x is defined %}`|
|**Global**|Variable/function available in all templates|
|**Extension**|Plugin that adds new tags or behaviour|

---

## Basic Usage

```python
from jinja2 import Environment, FileSystemLoader

env = Environment(loader=FileSystemLoader("templates"))
template = env.get_template("hello.html")

output = template.render(name="Alice", items=["a", "b", "c"])
print(output)
```

### Render from a String

```python
from jinja2 import Environment

env = Environment()
template = env.from_string("Hello, {{ name }}!")
print(template.render(name="Alice"))   # Hello, Alice!
```

---

## Syntax Overview

### Delimiters

```jinja2
{{ expression }}      {# output a value #}
{% statement %}       {# control flow — if, for, block, extends #}
{# comment #}         {# not rendered in output #}
```

### Variables & Attribute Access

```jinja2
{{ user }}
{{ user.name }}           {# attribute — dot notation #}
{{ user['email'] }}       {# dict key — bracket notation #}
{{ users[0].name }}       {# list index then attribute #}
{{ config.get('debug') }} {# method call #}
```

---

## Control Flow

### Conditionals

```jinja2
{% if user.is_admin %}
  <a href="/admin">Admin Panel</a>
{% elif user.is_verified %}
  <span class="badge">Verified</span>
{% else %}
  <span>Guest</span>
{% endif %}
```

### Loops

```jinja2
<ul>
{% for item in items %}
  <li>{{ loop.index }}. {{ item.name }} — £{{ item.price }}</li>
{% else %}
  <li>No items found.</li>
{% endfor %}
</ul>
```

### Loop Variables

|Variable|Description|
|---|---|
|`loop.index`|Current iteration (1-based)|
|`loop.index0`|Current iteration (0-based)|
|`loop.first`|`True` on first iteration|
|`loop.last`|`True` on last iteration|
|`loop.length`|Total number of items|
|`loop.revindex`|Iterations from the end (1-based)|
|`loop.cycle(...)`|Cycles through provided values|

```jinja2
{% for item in items %}
  <tr class="{{ loop.cycle('odd', 'even') }}">
    {% if loop.first %}<strong>{% endif %}
    {{ item.name }}
    {% if loop.first %}</strong>{% endif %}
  </tr>
{% endfor %}
```

### Loop with `recursive`

```jinja2
{# Render nested category trees #}
<ul>
{% for category in categories recursive %}
  <li>
    {{ category.name }}
    {% if category.children %}
      <ul>{{ loop(category.children) }}</ul>
    {% endif %}
  </li>
{% endfor %}
</ul>
```

---

## Template Inheritance

The most powerful Jinja2 feature — define a base layout once, extend it in child templates.

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>{% block title %}My App{% endblock %}</title>
  {% block head %}{% endblock %}
</head>
<body>
  {% block navbar %}
  <nav>
    <a href="/">Home</a>
    <a href="/about">About</a>
  </nav>
  {% endblock %}

  <main>
    {% block content %}{% endblock %}
  </main>

  {% block footer %}
  <footer>© 2026 My App</footer>
  {% endblock %}
</body>
</html>
```

```html
<!-- templates/items/list.html -->
{% extends "base.html" %}

{% block title %}Items — My App{% endblock %}

{% block content %}
<h1>All Items</h1>
<ul>
  {% for item in items %}
    <li>{{ item.name }}</li>
  {% endfor %}
</ul>
{% endblock %}
```

> [!tip] `{{ super() }}` Call `{{ super() }}` inside a block to render the parent block's content before adding your own.

```jinja2
{% block head %}
  {{ super() }}
  <link rel="stylesheet" href="/static/page.css">
{% endblock %}
```

---

## Includes & Macros

### Includes — Reusable Partials

```jinja2
{% include "partials/navbar.html" %}
{% include "partials/footer.html" %}

{# Ignore if file doesn't exist #}
{% include "partials/banner.html" ignore missing %}
```

### Macros — Reusable Template Functions

```jinja2
{# Define #}
{% macro input(name, type="text", value="", required=False) %}
  <div class="field">
    <input
      type="{{ type }}"
      name="{{ name }}"
      value="{{ value }}"
      {% if required %}required{% endif %}
    >
  </div>
{% endmacro %}

{# Use #}
{{ input("username", required=True) }}
{{ input("email", type="email", required=True) }}
{{ input("age",   type="number") }}
```

### Importing Macros from Another File

```jinja2
{% from "macros/forms.html" import input, button %}

{{ input("username") }}
{{ button("Submit", type="submit") }}
```

---

## Filters

Filters transform a value using the pipe `|` operator.

### Built-in Filters

```jinja2
{{ "hello world" | upper }}              {# HELLO WORLD #}
{{ "  hello  "   | trim }}               {# hello #}
{{ "hello world" | title }}              {# Hello World #}
{{ "hello world" | replace("world", "jinja2") }} {# hello jinja2 #}
{{ name          | default("Anonymous") }}  {# fallback if None/undefined #}
{{ name          | default("Anon", true) }} {# fallback if falsy #}

{{ 3.14159 | round(2) }}                 {# 3.14 #}
{{ 1000000 | int }}                      {# 1000000 #}

{{ items | length }}                     {# list length #}
{{ items | first }}                      {# first element #}
{{ items | last }}                       {# last element #}
{{ items | reverse | list }}             {# reversed list #}
{{ items | sort(attribute="name") }}     {# sorted by attribute #}
{{ items | groupby("category") }}        {# group by attribute #}
{{ items | selectattr("active") | list }}{# filter truthy attribute #}
{{ items | map(attribute="name") | list }}{# pluck attribute #}
{{ items | join(", ", attribute="name") }}{# join to string #}

{{ content | safe }}                     {# disable auto-escaping #}
{{ content | escape }}                   {# force HTML escape #}
{{ content | truncate(100, True, "…") }} {# truncate to 100 chars #}
{{ content | wordcount }}                {# count words #}
{{ content | nl2br }}                    {# newlines → <br> #}

{{ now   | strftime("%d %b %Y") }}       {# format datetime (custom filter) #}
```

### Chaining Filters

```jinja2
{{ items | selectattr("active") | map(attribute="name") | join(", ") }}
```

### Custom Filters

```python
from jinja2 import Environment

def currency(value, symbol="£"):
    return f"{symbol}{value:,.2f}"

def timeago(dt):
    from datetime import datetime
    delta = datetime.utcnow() - dt
    if delta.days > 0:
        return f"{delta.days}d ago"
    hours = delta.seconds // 3600
    if hours > 0:
        return f"{hours}h ago"
    return f"{delta.seconds // 60}m ago"

env = Environment(loader=FileSystemLoader("templates"))
env.filters["currency"] = currency
env.filters["timeago"]  = timeago
```

```jinja2
{{ product.price | currency }}           {# £9.99 #}
{{ post.created_at | timeago }}          {# 2h ago #}
```

---

## Tests

Tests are boolean checks used in `{% if %}` with `is`.

```jinja2
{% if value is defined %}...{% endif %}
{% if value is undefined %}...{% endif %}
{% if value is none %}...{% endif %}
{% if value is string %}...{% endif %}
{% if value is number %}...{% endif %}
{% if value is iterable %}...{% endif %}
{% if value is mapping %}...{% endif %}   {# dict-like #}
{% if value is sequence %}...{% endif %}
{% if loop.index is odd %}...{% endif %}
{% if loop.index is even %}...{% endif %}
{% if value is divisibleby(3) %}...{% endif %}
{% if value is sameas true %}...{% endif %}
```

### Custom Tests

```python
def is_premium(user):
    return user.plan == "premium"

env.tests["premium"] = is_premium
```

```jinja2
{% if user is premium %}
  <span class="badge">Premium</span>
{% endif %}
```

---

## Global Variables & Functions

Available in every template without being passed in context.

```python
from datetime import datetime

env.globals["app_name"]   = "My App"
env.globals["now"]        = datetime.utcnow
env.globals["url_for"]    = url_for          # e.g. FastAPI's url_for
```

```jinja2
<title>{{ app_name }}</title>
<footer>© {{ now().year }}</footer>
```

---

## Whitespace Control

Jinja2 preserves whitespace by default. Use `-` to strip it.

```jinja2
{%- if user -%}          {# strip whitespace before and after tag #}
  Hello {{ user.name }}
{%- endif -%}

{{- " hello " -}}        {# strip around output tag → "hello" #}
```

Or configure globally:

```python
env = Environment(
    trim_blocks=True,      # remove newline after block tags
    lstrip_blocks=True,    # strip leading spaces/tabs before block tags
)
```

---

## Auto-escaping

Auto-escaping prevents XSS by escaping `<`, `>`, `&`, `"` in output.

```python
# Auto-escape HTML files only
from jinja2 import Environment, FileSystemLoader, select_autoescape

env = Environment(
    loader=FileSystemLoader("templates"),
    autoescape=select_autoescape(["html", "xml"]),  # escape .html and .xml
)
```

```jinja2
{# Escaped (safe by default in HTML templates) #}
{{ user_input }}             {# <script> → &lt;script&gt; #}

{# Mark trusted content as safe — use carefully #}
{{ trusted_html | safe }}
{% autoescape false %}
  {{ trusted_html }}
{% endautoescape %}
```

> [!warning] Never use `| safe` or `autoescape false` on user-supplied input. Only mark content as safe when you control it or have sanitised it.

---

## Environment Options

```python
from jinja2 import Environment, FileSystemLoader, select_autoescape

env = Environment(
    loader=FileSystemLoader("templates"),  # where to find templates
    autoescape=select_autoescape(["html"]),# auto-escape HTML files
    trim_blocks=True,                     # strip newline after {% %}
    lstrip_blocks=True,                   # strip indent before {% %}
    keep_trailing_newline=True,           # preserve final newline
    undefined=StrictUndefined,            # raise on undefined variables
    extensions=["jinja2.ext.debug"],      # enable {% debug %} tag
)
```

### Undefined Behaviour

```python
from jinja2 import Undefined, DebugUndefined, StrictUndefined

# Undefined (default) — renders empty string silently
# DebugUndefined       — renders {{ variable }} literally (useful in dev)
# StrictUndefined      — raises UndefinedError immediately (recommended)

env = Environment(undefined=StrictUndefined)
```

---

## Loaders

```python
from jinja2 import (
    FileSystemLoader,      # load from directory
    PackageLoader,         # load from Python package
    DictLoader,            # load from dict (testing)
    ChoiceLoader,          # try multiple loaders in order
    PrefixLoader,          # namespace templates by prefix
)

# From a directory
FileSystemLoader("templates")
FileSystemLoader(["templates", "shared/templates"])   # multiple dirs

# From a Python package (templates inside the package)
PackageLoader("myapp", "templates")

# From a dict (great for tests)
DictLoader({"index.html": "<h1>Hello {{ name }}</h1>"})

# Try multiple loaders
ChoiceLoader([FileSystemLoader("templates"), PackageLoader("myapp")])

# Namespaced
PrefixLoader({
    "admin":  FileSystemLoader("admin/templates"),
    "public": FileSystemLoader("public/templates"),
})
# Usage: env.get_template("admin/dashboard.html")
```

---

## Template Caching

Jinja2 compiles templates to Python bytecode and caches them in memory. For production, also cache to disk:

```python
from jinja2 import Environment, FileSystemLoader, BytecodeCache
from jinja2.bccache import FileSystemBytecodeCache

cache = FileSystemBytecodeCache(directory="/tmp/jinja2_cache")

env = Environment(
    loader=FileSystemLoader("templates"),
    bytecode_cache=cache,
    auto_reload=False,   # don't check for file changes (prod)
)
```

---

## Useful Extensions

```python
env = Environment(extensions=[
    "jinja2.ext.debug",     # {% debug %} — dumps context variables
    "jinja2.ext.loopcontrols",  # {% break %} and {% continue %} in loops
    "jinja2.ext.do",        # {% do list.append(item) %} — execute expressions
    "jinja2.ext.i18n",      # {% trans %} — internationalisation
])
```

```jinja2
{# Debug — dumps all context variables #}
{% debug %}

{# Loop controls #}
{% for item in items %}
  {% if item.hidden %}{% continue %}{% endif %}
  {{ item.name }}
{% endfor %}

{# Do — execute a statement without output #}
{% set ns = namespace(total=0) %}
{% for item in items %}
  {% do ns.__setattr__('total', ns.total + item.price) %}
{% endfor %}
Total: {{ ns.total }}
```

---

## Quick Reference

|Task|Syntax|
|---|---|
|Output variable|`{{ variable }}`|
|Conditional|`{% if %} … {% elif %} … {% else %} … {% endif %}`|
|Loop|`{% for x in items %} … {% endfor %}`|
|Inherit layout|`{% extends "base.html" %}`|
|Define block|`{% block name %} … {% endblock %}`|
|Include partial|`{% include "partial.html" %}`|
|Define macro|`{% macro name(args) %} … {% endmacro %}`|
|Import macro|`{% from "file.html" import macro %}`|
|Apply filter|`{{ value \| filter }}`|
|Chain filters|`{{ value \| f1 \| f2 \| f3 }}`|
|Run test|`{% if value is test %}`|
|Mark safe|`{{ value \| safe }}`|
|Comment|`{# comment #}`|
|Strip whitespace|`{%- tag -%}`|
|Parent block|`{{ super() }}`|

---

## Tags

#python #jinja2 #templating #html #backend #frontend