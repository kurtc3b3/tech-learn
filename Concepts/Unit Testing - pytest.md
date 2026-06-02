#unit-test #code-quality #fixtures

Official pytest guide: [[pytest How-tos]]
Cheatsheet: [[pytest Cheatsheet]]

---
### Definition

Unit testing is a way of checking that **individual small pieces of your code** (called "units" — usually functions or methods) work correctly on their own, before combining them into a bigger system.

Think of it like testing individual LEGO bricks before building a castle — you make sure each brick is solid and fits properly before snapping everything together.

**Key Points:**

- **Tests one thing at a time** — each test focuses on a single function or piece of logic in isolation
- **Written by developers** — usually the same person who writes the code also writes the tests for it
- **Runs automatically** — tests can be run instantly anytime, catching bugs the moment they're introduced
- **Fast feedback** — since units are small, tests run in milliseconds, giving you quick results
- **Acts as documentation** — tests show exactly how a function is supposed to behave, making code easier to understand
- **Easier debugging** — when a unit test fails, you know _exactly_ which small piece of code is broken
- **Supports refactoring** — you can safely rewrite or improve code knowing the tests will catch any regressions
- **Doesn't test everything** — unit tests don't check how different parts work _together_ (that's what integration testing is for)

**Example:**

See the simple example using [[Unit Test - Basic]]

---
### Fixtures:

`conftest.py` is a **special file pytest automatically finds** — you put shared setup code in it so you don't repeat yourself across multiple test files.

Think of it like a **common kitchen** in an office — instead of every person bringing their own microwave, everyone shares the one in the kitchen.

**The key concept — fixtures:**

A fixture is just a reusable piece of setup (like creating a user, connecting to a database, etc.) that your tests can request by name.

**Key points:**

- **No need to import it** — pytest discovers it on its own
- **Shared across files** — any test file in the same folder can use its fixtures
- **Avoids repetition** — define setup once, reuse everywhere
- **Can do cleanup too** — using `yield` instead of `return`, you can also add teardown logic after the test runs

**Example:**

See the fixture example through [[Unit Test - Fixtures]]

---
### Scope:

Fixture scope controls **how often a fixture is created and destroyed** during your test run.

Think of it like making coffee — do you brew a fresh cup for every sip, every person, or just once for the whole office?

**The 4 scopes:**

```python
@pytest.fixture(scope="function")  # default — runs for every test
@pytest.fixture(scope="class")     # runs once per test class
@pytest.fixture(scope="module")    # runs once per file
@pytest.fixture(scope="session")   # runs once for the entire test run
```

**When to use which:**

| Scope      | Use it for                                        |
| ---------- | ------------------------------------------------- |
| `function` | Simple data, cheap setup                          |
| `class`    | Shared state within a group of related tests      |
| `module`   | Setup that's slow but only needed per file        |
| `session`  | Expensive things like DB connections, API clients |
The golden rule: **use the widest scope you can** without tests interfering with each other — it keeps your test suite fast.

**Example:**

Check the example code snippet: [[Unit Test - Fixture Scope]]

---
### Parameterise Fixtures:

`@pytest.mark.parametrize` lets you **run the same test with multiple inputs** without copy-pasting the test function.

Think of it like a **vending machine** — same machine, different button, different output.

**Key points:**

- Each row of inputs = one separate test case
- Failed cases are reported individually, so you know exactly which input broke
- Keeps your test file short and easy to extend — just add a new row to test a new case

**Example:**

See the example: [[Unit Test - Fixture Parameterise]]

---
### Slow Tests:

`@pytest.mark.slow` is a **custom mark** that lets you label certain tests as slow, so you can choose to skip them when you want a quick test run.

**Key points:**

- `slow` is not built-in — you **define it yourself** in `pytest.ini`
- Great for separating **unit tests** (fast) from **integration tests** (slow)
- You can create **any custom mark** the same way — `@pytest.mark.smoke`, `@pytest.mark.regression`, etc.
- Keeps your **quick feedback loop fast** during development by skipping heavy tests

**Example:**

See the code snippet: [[Unit Test - Fixture Marks]]

---
### Built-in Fixtures

pytest comes with **built-in fixtures** ready to use — no imports, no setup, just add them as arguments and pytest provides them automatically.

**Quick reference:**

|Fixture|What it does|
|---|---|
|`tmp_path`|Temp folder (auto deleted), modern style|
|`tmpdir`|Temp folder, older style|
|`capsys`|Capture stdout/stderr output|
|`monkeypatch`|Temporarily replace env vars, functions, attributes|
|`caplog`|Capture log messages|
|`request`|Access test metadata inside a fixture|

**Key points:**

- All built-in — **no imports or installs needed**
- All **automatically cleaned up** after each test
- `monkeypatch` and `tmp_path` are the most commonly used in real projects
- You can combine them freely — e.g. `def test_something(tmp_path, monkeypatch, capsys)`

**Example:**

See examples: [[Unit Test - Built-in Fixtures]]

