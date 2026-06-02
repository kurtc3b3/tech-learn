#fixtures #pytest #unit-test #python 

- **`tmp_path` — temporary folder for files:**

```python
def test_write_file(tmp_path):
    file = tmp_path / "hello.txt"     # creates a temp folder path
    file.write_text("hello world")    # write to it
    assert file.read_text() == "hello world"
```

Pytest **automatically deletes** the folder after the test. Great for testing file operations without cluttering your project.

- **`tmpdir` — older version of `tmp_path`:**

```python
def test_write_file(tmpdir):
    file = tmpdir.join("hello.txt")   # older py.path style
    file.write("hello world")
    assert file.read() == "hello world"
```

`tmp_path` is preferred nowadays — it uses modern Python `pathlib`.

- **`capsys` — capture print output:**

```python
def greet():
    print("hello world")

def test_greet(capsys):
    greet()
    captured = capsys.readouterr()    # grab stdout/stderr
    assert captured.out == "hello world\n"
```

Useful when your function prints output and you want to assert on it.

- **`monkeypatch` — temporarily replace/mock things:**

```python
def test_env_variable(monkeypatch):
    monkeypatch.setenv("APP_MODE", "testing")   # fake an env variable
    import os
    assert os.getenv("APP_MODE") == "testing"
    # restored automatically after test
```

Great for faking environment variables, replacing functions, or patching values — all **automatically restored** after the test.

- **`caplog` — capture log messages:**

```python
import logging

def do_something():
    logging.warning("something went wrong")

def test_logging(caplog):
    do_something()
    assert "something went wrong" in caplog.text
```

- **`request` — inspect the test context itself:**

```python
@pytest.fixture
def smart_fixture(request):
    print(f"Running test: {request.node.name}")   # prints test name
    return 42
```

Useful inside fixtures to know which test is calling them.

