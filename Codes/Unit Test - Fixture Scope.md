#unit-test #fixtures #pytest 
- Pre-requisites

Install dependencies: [[pytest]]

```python
import pytest

@pytest.fixture(scope="session")
def db_connection():
    print("🔌 Connect to DB")   # runs ONCE
    yield
    print("❌ Disconnect DB")   # cleanup runs ONCE at the very end

@pytest.fixture(scope="function")
def sample_user():
    return {"name": "Alice"}    # runs fresh for EVERY test
```
