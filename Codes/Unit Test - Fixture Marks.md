#unit-test #python #pytest #fixtures 

- Pre-requisites

Install dependencies: [[pytest]]

**Step 1 — Register the mark in `pytest.ini`:**

```ini
[pytest]
markers =
    slow: marks tests as slow
```

**Step 2 — Tag your slow tests:**

```python
import pytest
from math_utils import add

def test_add_fast():
    assert add(2, 3) == 5          # quick test, no mark

@pytest.mark.slow
def test_add_slow():
    import time
    time.sleep(3)                  # simulates a slow operation
    assert add(10, 20) == 30
```

**Step 3 — Run only what you need:**

```bash
# Run ALL tests (including slow)
pytest

# Skip slow tests
pytest -m "not slow"

# Run ONLY slow tests
pytest -m slow
```

**Output when skipping slow:**

```
test_math_utils.py::test_add_fast    PASSED

1 passed, 1 deselected in 0.01s
```

**You can combine marks too:**

```python
@pytest.mark.slow
@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (0, 0, 0),
])
def test_add_slow_parametrized(a, b, expected):
    import time
    time.sleep(1)
    assert add(a, b) == expected
```
