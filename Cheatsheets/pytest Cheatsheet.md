  
## CLI — Running Tests

| Command                          | Description                          |
| -------------------------------- | ------------------------------------ |
| `pytest`                         | Run all tests from current directory |
| `pytest test_file.py`            | Run a specific file                  |
| `pytest test_file.py::test_name` | Run a single test                    |
| `pytest -k "auth or login"`      | Run tests matching expression        |
| `pytest -m slow`                 | Run tests with a marker              |
| `pytest -x`                      | Stop on first failure                |
| `pytest --lf`                    | Re-run last failed tests             |
| `pytest --lf --nf`               | Re-run failed + new tests            |
| `pytest -v` / `-q`               | Verbose / quiet output               |
| `pytest -s`                      | Show stdout (no capture)             |
| `pytest --tb=short\|long\|no`    | Traceback style                      |
| `pytest -n 4`                    | Parallel execution (pytest-xdist)    |
| `pytest --co -q`                 | Collect only, don't run              |

---
## Assertions

```python

def test_basics():
	assert x == 42
	assert x != 0x
	assert x > 10
	assert x is None
	assert x is not None
	assert x in [1, 2, 3]
	assert not x

# float comparison
from pytest import approx
assert 0.1 + 0.2 == approx(0.3)
assert val == approx(1.0, rel=1e-3)

# exceptions
with pytest.raises(ValueError):
	risky_fn()

with pytest.raises(ValueError, match=r"invalid"):
	risky_fn()

# capture exception info
with pytest.raises(TypeError) as exc:
	bad()
	assert "expected" in str(exc.value)

```

---
## Fixtures

```python
import pytest

@pytest.fixture
def db():
	conn = create_db()
	yield conn # setup
	conn.close() # teardown

# use in test
def test_query(db):
	assert db.count() == 0

# scope (function is default)
@pytest.fixture(scope="module")
# scope: function | class | module | package | session

# autouse — runs for every test automatically
@pytest.fixture(autouse=True)
def reset_state(): ...


# parametrize fixture
@pytest.fixture(params=["pg", "sqlite"])
def driver(request):
	return request.param

```

> **`conftest.py`** — place shared fixtures here; auto-discovered, no import needed.

---
## Parametrize

```python

@pytest.mark.parametrize("x,expected", [
	(2, 4),
	(3, 9),
	(-1, 1),
])
def test_square(x, expected):
	assert x**2 == expected

# single param
@pytest.mark.parametrize("val", [0, -1, 999])
def test_valid(val): ...

# custom ids for readability
@pytest.mark.parametrize("n", [10, 100], ids=["small", "large"])
def test_n(n): ...

# stacked decorators = cartesian product (4 tests)
@pytest.mark.parametrize("x", [1, 2])
@pytest.mark.parametrize("y", [3, 4])
def test_xy(x, y): ...

```

---
## Markers

```python
# built-in
@pytest.mark.skip(reason="not ready")
@pytest.mark.skipif(sys.platform == "win32", reason="unix only")
@pytest.mark.xfail # expected failure
@pytest.mark.xfail(strict=True) # must fail

# custom markers — register in pytest.ini
@pytest.mark.slow
@pytest.mark.integration
def test_heavy(): ...

```


```ini
# pytest.ini
[pytest]
markers =
	slow: marks tests as slow
	integration: integration tests
```


```bash
pytest -m slow
pytest -m "not slow"
pytest -m "slow and not db"
```
  
---
## Mocking (pytest-mock)

```python
# pip install pytest-mock
def test_call(mocker):
	m = mocker.patch("app.send_email")
	register_user("alice")
	m.assert_called_once()

# patch return value
mocker.patch("app.get_data", return_value={"key": "val"})

# patch side effect
mocker.patch("app.fetch", side_effect=ConnectionError)

# spy — calls real method and records calls
spy = mocker.spy(obj, "method")
obj.method()
spy.assert_called_once_with(42)

# stdlib unittest.mock
from unittest.mock import MagicMock, patch
with patch("module.Class") as mock:
	mock.return_value.method.return_value = 1
```

---
## Configuration

```ini
# pytest.ini
[pytest]
testpaths = tests
addopts = -v --tb=short
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
log_cli = true
log_level = INFO
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
```

---
## Built-in Fixtures

```python
# tmp_path — temporary directory
def test_file(tmp_path):
	f = tmp_path / "data.txt"
	f.write_text("hello")
	assert f.read_text() == "hello"

# capsys — capture stdout/stderr
def test_print(capsys):
	print("hello")
	out, err = capsys.readouterr()
	assert out == "hello\n"

# monkeypatch — patch objects, env vars, etc.
def test_env(monkeypatch):
	monkeypatch.setenv("API_KEY", "test")
	monkeypatch.setattr(obj, "attr", val)
	monkeypatch.delattr(module, "fn")

# async tests (requires pytest-asyncio)
@pytest.mark.asyncio
async def test_async():
	result = await fetch_data()
	assert result["ok"]
```

---
## Fixture Scope Reference

| Scope                  | Lifetime                             |
| ---------------------- | ------------------------------------ |
| `function` *(default)* | New instance per test function       |
| `class`                | Shared across tests in a class       |
| `module`               | Shared across tests in a file        |
| `package`              | Shared across tests in a directory   |
| `session`              | One instance for the entire test run |

---
## Useful Plugins

| Plugin             | Purpose                                            |
| ------------------ | -------------------------------------------------- |
| `pytest-cov`       | Coverage — `--cov=src --cov-report=html`           |
| `pytest-mock`      | `mocker` fixture wrapping `unittest.mock`          |
| `pytest-xdist`     | Parallel execution with `-n auto`                  |
| `pytest-asyncio`   | Test async functions                               |
| `pytest-httpx`     | Mock httpx requests                                |
| `pytest-freezegun` | Freeze time with `@freeze_time("2024-01-01")`      |
| `pytest-randomly`  | Randomize test order to catch order-dependent bugs |
| `pytest-timeout`   | Fail slow tests with `--timeout=5                  |
