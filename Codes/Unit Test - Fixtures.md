#unit-test #code-quality #python #pytest  #fixtures #conftest-py

- Pre-requisites

Install dependencies: [[pytest]]

- **`conftest.py`:**

```python
import pytest

@pytest.fixture
def sample_numbers():
    return {"a": 2, "b": 3}
```

- **`test_math_utils.py`:**

```python
from math_utils import add

def test_add(sample_numbers):        # pytest sees this name and
    a = sample_numbers["a"]          # automatically pulls it from
    b = sample_numbers["b"]          # conftest.py
    assert add(a, b) == 5
```

No import needed — pytest finds `conftest.py` automatically.
