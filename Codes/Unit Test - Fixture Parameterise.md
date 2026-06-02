#unit-test #python #pytest #fixtures 

- Pre-requisites

Install dependencies: [[pytest]]

- **Without parametrize (repetitive 😫):**

```python
def test_add_2_and_3():
    assert add(2, 3) == 5

def test_add_0_and_0():
    assert add(0, 0) == 0

def test_add_negative():
    assert add(-1, 4) == 3
```

- **With parametrize (clean ✅):**

```python
import pytest
from math_utils import add

@pytest.mark.parametrize("a, b, expected", [
    (2,  3,  5),
    (0,  0,  0),
    (-1, 4,  3),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

- pytest runs this **3 separate times**, once per row.

- **You can also combine it with fixtures:**

```python
@pytest.fixture
def multiplier():
    return 2

@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (0, 0, 0),
])
def test_add_with_fixture(a, b, expected, multiplier):
    assert add(a, b) == expected
    assert multiplier == 2          # fixture still available
```
