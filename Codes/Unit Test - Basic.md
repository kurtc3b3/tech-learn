#unit-test #code-quality #python #pytest 

- Pre-requisites

Install dependencies: [[pytest]]

- The function (`math_utils.py`):

```python
def add(a, b):
    return a + b
```

- The test (`test_math_utils.py`):

```python
from math_utils import add

def test_add():
    assert add(2, 3) == 5

def test_add_negative():
    assert add(-1, 4) == 3
```

- Run it with:

```shell
pytest test_math_utils.py
```

- Note:
Each `test_` function is a unit test. `assert` checks that the result matches what you expect — if it doesn't, pytest flags it as a failure.
