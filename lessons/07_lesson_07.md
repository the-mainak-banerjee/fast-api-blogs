# FastAPI — Lesson 7: Sync vs Async — Converting Your App to Asynchronous

**Lesson:** 7
**Video:** https://youtu.be/2JPDt-Jp6fM?si=Y8ssCe-DZRWj-S-m

---

## 1. What is Async?

**Async** allows a program to work on other tasks while one task is waiting for something to finish.

For example:

```text
Request 1 → Waiting for database
Request 2 → Process
Request 3 → Waiting for API
Request 4 → Process
```

Instead of sitting idle while Request 1 waits, the application can work on Request 2 and 4.

> **Async is mainly useful for handling I/O-bound work efficiently.**

---

## 2. I/O-Bound vs CPU-Bound

### I/O-Bound

The program spends most of its time **waiting**.

Examples:

* Database queries
* API requests
* Reading files
* Network requests

```text
Send request
     ↓
     WAIT...
     ↓
Response arrives
```

Async is useful here because we can do other work while waiting.

### CPU-Bound

The program spends most of its time **calculating**.

Examples:

* Large mathematical calculations
* Image processing
* Video encoding
* Machine learning computation

```text
Start calculation
       ↓
   CPU working
       ↓
Calculation finished
```

Async doesn't automatically make CPU-heavy work faster.

---

## 3. Can Async Become Slower?

**Yes.**

Async has some overhead for managing tasks and switching between them.

For a simple task:

```text
Task → Do work → Finish
```

adding asynchronous scheduling may provide no benefit and can sometimes be slightly slower.

Async becomes valuable when there are many tasks that spend significant time **waiting for I/O**.

Also, if you perform heavy CPU work directly inside the async event loop, it can block other requests and hurt concurrent performance.

### Example

Imagine 1,000 requests:

```text
Request 1 → Database → WAIT
Request 2 → Database → WAIT
Request 3 → Database → WAIT
...
Request 1000 → Database → WAIT
```

Async can keep the application busy handling other requests while those database operations are waiting.

But if every request performs a huge CPU calculation:

```text
Request 1 → CPU calculation
Request 2 → CPU calculation
Request 3 → CPU calculation
```

the event loop can become blocked.

---

# 4. `aiosqlite`

SQLite itself is traditionally accessed synchronously.

`aiosqlite` provides an **async interface/driver for SQLite**.

We change:

```python
SQLALCHEMY_DATABASE_URL = "sqlite:///./blog.db"
```

to:

```python
SQLALCHEMY_DATABASE_URL = "sqlite+aiosqlite:///./blog.db"
```

The important part is:

```text
sqlite + aiosqlite
       ↑
    async driver
```

This tells SQLAlchemy:

> "Use SQLite, but communicate with it through the `aiosqlite` async driver."

---

# 5. Async Session Factory

```python
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)
```

### `async_sessionmaker`

Creates a **factory for asynchronous SQLAlchemy sessions**.

Similar to:

```python
SessionLocal = sessionmaker(...)
```

but for async operations.

### `class_=AsyncSession`

Tells SQLAlchemy:

> "The sessions created by this factory should be `AsyncSession` objects."

### `expire_on_commit=False`

Normally SQLAlchemy can expire object attributes after `commit()`, meaning SQLAlchemy may need to load them again from the database when accessed.

With async code, unexpected lazy database access can cause problems.

Setting:

```python
expire_on_commit=False
```

means:

> "Keep the object's loaded attributes available after commit."

---

# 6. Async Database Dependency

Our database dependency becomes something like:

```python
async def get_db():
    async with AsyncSessionLocal() as db:
        yield db
```

Notice:

```python
async def
```

and:

```python
async with
```

because we're working with an asynchronous session.

---

# 7. Why `await`?

Some database operations involve waiting for the database.

For example:

```python
result = await db.execute(...)
```

The `await` means:

> "Start this operation and let other async work continue while we're waiting for the result."

Once the database responds, execution continues.

---

# 8. Why Isn't `db.add()` Awaited?

We have:

```python
db.add(new_user)
```

but:

```python
await db.commit()
```

Why?

Because `add()` doesn't immediately communicate with the database.

It simply tells the SQLAlchemy Session:

> "Track this object as something I want to insert."

It's an in-memory operation.

```text
db.add()
   ↓
Session remembers the object
```

`commit()`, on the other hand, needs to actually communicate with the database:

```text
await db.commit()
        ↓
Database communication
        ↓
Wait for database
```

Therefore:

```python
db.add(new_user)       # No await
await db.commit()      # await
```

---

# 9. `refresh()` and `await`

```python
await db.refresh(new_user)
```

`refresh()` may need to communicate with the database to reload the object's current state.

Therefore it is asynchronous and needs:

```python
await
```

---

# 10. Lazy Loading

Suppose our models have:

```python
class Post(Base):
    author: Mapped[User] = relationship(
        back_populates="posts"
    )
```

You retrieve a post:

```python
post = ...
```

and later access:

```python
post.author
```

SQLAlchemy may need to execute another database query to get the author.

That's called **lazy loading**.

Conceptually:

```text
Get Post
   ↓
Post loaded
   ↓
post.author
   ↓
"Oh, I need the author"
   ↓
Another DB query
```

---

# 11. Why Lazy Loading Is a Problem in Async?

In synchronous SQLAlchemy, SQLAlchemy can transparently perform that additional query.

With async SQLAlchemy, implicit database I/O when simply accessing an attribute can cause problems because database I/O should happen through an awaited async operation.

Therefore, we generally want to **load relationships explicitly**.

---

# 12. Eager Loading with `selectinload`

We can use:

```python
from sqlalchemy.orm import selectinload
```

and:

```python
select(models.Post).options(
    selectinload(models.Post.author)
)
```

This tells SQLAlchemy:

> "When retrieving the posts, also load their authors."

Conceptually:

```text
Query Posts
    ↓
Load Posts
    ↓
Load related Authors
    ↓
Return everything
```

This is called **eager loading**.

---

# 13. `selectinload()` vs Lazy Loading

### Lazy loading

```text
Get Post
   ↓
Access post.author
   ↓
Another query
```

### Eager loading

```text
Get Post
   ↓
Load Post + Author
   ↓
Everything is ready
```

For async applications, explicit relationship loading is often preferable because it makes database I/O predictable.

---

# 14. Refreshing a Relationship

Another approach is:

```python
await db.refresh(
    new_post,
    attribute_names=["author"]
)
```

This tells SQLAlchemy:

> "Refresh this object and explicitly load its `author` relationship."

So instead of:

```text
post
 ↓
post.author
 ↓
Unexpected database access
```

we explicitly ask for it:

```text
await db.refresh(post, ["author"])
                ↓
        Author gets loaded
```

This is useful when you specifically need that relationship on an already-loaded object.

---

# 15. `selectinload()` vs `refresh(..., attribute_names=...)`

They solve a similar problem but are used in different situations.

### `selectinload()`

Usually used **when querying**:

```python
result = await db.execute(
    select(models.Post).options(
        selectinload(models.Post.author)
    )
)
```

Meaning:

> "When you fetch these posts, also fetch their authors."

### `refresh()`

Useful **after you already have an object**:

```python
await db.refresh(
    new_post,
    attribute_names=["author"]
)
```

Meaning:

> "I already have this post. Now explicitly load its author."

---

# 16. New Imports in `main.py`

### Exception handlers

```python
from fastapi.exception_handlers import (
    http_exception_handler,
    request_validation_exception_handler
)
```

These are FastAPI's **built-in exception handlers**.

Instead of rewriting the default FastAPI behavior ourselves, we can reuse these handlers inside our custom exception handling logic.

For example, we might customize handling for HTML pages while still using FastAPI's standard handling for API requests.

---

# 17. `asynccontextmanager`

```python
from contextlib import asynccontextmanager
```

This is used to create an **async context manager**, commonly for FastAPI's application lifespan.

It allows us to define what should happen when the application:

```text
Starts
  ↓
Runs
  ↓
Shuts down
```

---

# 18. What Is a Lifespan Function?

A lifespan function lets us run code during the **startup and shutdown of the FastAPI application**.

Example:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("Application starting")

    yield

    # Shutdown
    print("Application shutting down")
```

Before `yield`:

```text
Application startup
```

After `yield`:

```text
Application shutdown
```

### What can we use it for?

Common examples:

* Initialize resources
* Create database tables in simple applications
* Load ML/AI models
* Create shared clients
* Initialize caches
* Clean up resources
* Close connections/resources during shutdown

Think:

> **Lifespan = "What should my application do when it starts and when it stops?"**

---

# 19. Don't Use Sync Libraries Inside Async Code

Suppose your route is:

```python
async def get_data():
```

and you use a synchronous HTTP library that blocks while waiting.

Conceptually:

```text
Async event loop
      ↓
Sync request
      ↓
BLOCKED
      ↓
Other async tasks can't run efficiently
```

So when writing async applications, use async-compatible libraries for I/O when appropriate.

For example:

```text
Async FastAPI
      ↓
Async database driver
      ↓
Async HTTP client
```

The goal is to avoid blocking the event loop with synchronous I/O.

---

# 20. Complete Async Request Flow

After converting our application:

```text
Client
  ↓
FastAPI
  ↓
async route
  ↓
Pydantic validation
  ↓
AsyncSession
  ↓
SQLAlchemy Async Engine
  ↓
aiosqlite
  ↓
SQLite
```

When the database is waiting:

```text
Request A
   ↓
await database
   ↓
     WAIT ─────────┐
                   │
Request B          │
   ↓               │
do other work      │
                   │
Request C          │
   ↓               │
do other work      │
                   │
                   ▼
             Database responds
                   ↓
             Request A continues
```

That's the main benefit of async I/O.

---

# 🧠 Key Takeaways

* **Async** allows other tasks to run while an I/O operation is waiting.
* **I/O-bound** → mostly waiting → async is useful.
* **CPU-bound** → mostly calculating → async doesn't automatically make it faster.
* `aiosqlite` → async SQLite driver.
* `sqlite+aiosqlite://` → tells SQLAlchemy to use SQLite through `aiosqlite`.
* `AsyncSession` → SQLAlchemy's async session.
* `await` → wait for an asynchronous operation without blocking the event loop in the same way a synchronous wait would.
* `db.add()` → in-memory session operation, so no `await`.
* `db.commit()` → database I/O, so `await`.
* **Lazy loading** → relationship is loaded when accessed.
* **Eager loading** → relationship is loaded explicitly as part of the query/operation.
* `selectinload()` → useful for eagerly loading relationships during a query.
* `refresh(..., attribute_names=["author"])` → explicitly loads a relationship on an existing object.
* `lifespan` → startup and shutdown logic for the application.
* Avoid blocking synchronous I/O libraries inside async code.

### The mental model

```text
SYNC

Request
   ↓
Database operation
   ↓
WAIT
   ↓
Continue


ASYNC

Request A
   ↓
Database operation
   ↓
WAIT ─────────────┐
                  │
Request B         │
   ↓              │
Do other work     │
                  │
Request C         │
   ↓              │
Do other work     │
                  │
                  ▼
           A's DB operation
              finishes
                  ↓
            Continue A
```

> **Async doesn't make the database itself faster; it helps your application use its waiting time more efficiently.**
