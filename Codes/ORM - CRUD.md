
**Create sync:**

```python
with Session() as s:
    user = User(
        name="Alice",
        email="alice@example.com",
    )
    s.add(user)
    s.commit()
    s.refresh(user)
    print(user.id)  # populated

# Bulk insert
with Session() as s:
    s.add_all([
        User(name="Bob", email="b@b.com"),
        User(name="Eve", email="e@e.com"),
    ])
    s.commit()
```

**Create async:**

```python
async with AsyncSessionLocal() as s:
    user = User(
        name="Alice",
        email="alice@example.com",
    )
    s.add(user)
    await s.commit()
    await s.refresh(user)
    print(user.id)

# Bulk insert (async)
async with AsyncSessionLocal() as s:
    s.add_all([
        User(name="Bob", email="b@b.com"),
        User(name="Eve", email="e@e.com"),
    ])
    await s.commit()
```

**Read sync:**

```python
from sqlalchemy import select

with Session() as s:
    # by PK
    user = s.get(User, 1)

    # first match
    user = s.execute(
        select(User).where(User.email == "a@a.com")
    ).scalar_one_or_none()

    # all
    users = s.execute(
        select(User).order_by(User.name)
    ).scalars().all()
```

**Read async:**

```python
from sqlalchemy import select

async with AsyncSessionLocal() as s:
    # by PK
    user = await s.get(User, 1)

    # first match
    result = await s.execute(
        select(User).where(User.email == "a@a.com")
    )
    user = result.scalar_one_or_none()

    # all
    result = await s.execute(select(User))
    users = result.scalars().all()
```

**Update & Delete sync:**

```python
from sqlalchemy import update, delete

with Session() as s:
    # ORM-style update
    user = s.get(User, 1)
    user.name = "Updated"
    s.commit()

    # bulk update
    s.execute(
        update(User)
        .where(User.age < 18)
        .values(age=18)
    )
    s.commit()

    # delete
    s.delete(user)
    s.commit()
```

**Update & Delete async:**

```python
from sqlalchemy import update, delete

async with AsyncSessionLocal() as s:
    # ORM-style update
    user = await s.get(User, 1)
    user.name = "Updated"
    await s.commit()

    # bulk update
    await s.execute(
        update(User)
        .where(User.age < 18)
        .values(age=18)
    )
    await s.commit()

    # delete
    await s.delete(user)
    await s.commit()
```