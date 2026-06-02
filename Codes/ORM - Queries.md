
**Filtering, Ordering, Pagination:**

```python
from sqlalchemy import select, or_, and_, not_

stmt = (
    select(User)
    .where(
        and_(
            User.age >= 18,
            or_(User.name.like("%Alice%"), User.email.endswith("@corp.com")),
            not_(User.email == None),
        )
    )
    .order_by(User.name.asc(), User.created_at.desc())
    .limit(20)
    .offset(40)   # page 3 of 20
)
```

**Joins:**

```python
# inner join
stmt = (
    select(User, Post)
    .join(Post, Post.user_id == User.id)
)

# left outer join
stmt = (
    select(User, Post)
    .outerjoin(Post)
)

# join via relationship
stmt = (
    select(User)
    .join(User.posts)
    .where(Post.title.contains("SQLAlchemy"))
)
```

**Aggregates & Grouping:**

```python
from sqlalchemy import func, distinct

# count, avg, sum
stmt = select(
    func.count(User.id),
    func.avg(User.age),
    func.sum(User.age),
)

# group by + having
stmt = (
    select(User.name, func.count(Post.id))
    .join(Post)
    .group_by(User.name)
    .having(func.count(Post.id) > 5)
)

# distinct
stmt = select(distinct(User.email))
```

**Eager Loading:**

```python
from sqlalchemy.orm import (
    selectinload, joinedload, lazyload
)

# selectin — best for collections
stmt = (
    select(User)
    .options(selectinload(User.posts))
)

# joined — best for single obj
stmt = (
    select(Post)
    .options(joinedload(Post.author))
)
```

**Raw SQL & Text:**

```python
from sqlalchemy import text

# sync
with Session() as s:
    rows = s.execute(
        text("SELECT * FROM users WHERE age > :age"),
        {"age": 18}
    ).all()

# async
async with AsyncSessionLocal() as s:
    result = await s.execute(
        text("SELECT id, name FROM users")
    )
    rows = result.all()
```

