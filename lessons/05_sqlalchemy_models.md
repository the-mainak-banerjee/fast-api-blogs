# FastAPI — Lesson 5: SQLAlchemy Models & Relationships

**Lesson:** 5
**Video:** https://youtu.be/NvOV3ig2tGY?si=ohAMRQGSZVsB2EGs

## Learnings

### 1. Request → Database → Response

The basic flow is:

```text
Client
  ↓
Request
  ↓
Pydantic validates request
  ↓
SQLAlchemy stores/retrieves data
  ↓
Pydantic validates/formats response
  ↓
Response
  ↓
Client
```

* **Pydantic** → validates API input/output.
* **SQLAlchemy** → communicates with the database.
* **Database** → stores the actual data.

---

## 2. Setting Up SQLAlchemy

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///./blog.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False},
)

SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine,
)
```

### `create_engine()`

Creates a **database engine**, which manages the connection between our application and the database.

### `check_same_thread=False`

SQLite normally restricts a connection to the thread where it was created.

FastAPI can handle requests using multiple threads, so this restriction is disabled for this SQLite setup.

> This setting is mainly relevant to SQLite; other databases have different connection/concurrency behavior.

### `SessionLocal`

`sessionmaker()` creates a **factory for database sessions**.

A session represents a working interaction/transaction with the database.

```python
SessionLocal()
```

creates a new session.

We generally create a separate session for each request and close it when the request is finished.

```python
def get_db():
    with SessionLocal() as db:
        yield db
```

* `SessionLocal()` → creates a database session.
* `with` → ensures the session is closed after use.
* `yield db` → provides the session to the route that needs it.

---

## 3. SQLAlchemy `Base`

```python
class Base(DeclarativeBase):
    pass
```

`Base` is the parent class for our SQLAlchemy models.

For example:

```python
class User(Base):
    ...
```

SQLAlchemy uses `Base` to keep track of the models and their database table definitions.

---

## 4. Forward References

```python
from __future__ import annotations
```

This allows type annotations to be treated more flexibly, including references to classes that are defined later.

For example:

```python
class User(Base):
    posts: Mapped[list[Post]]
```

`Post` hasn't been defined yet when Python reads `User`.

With postponed annotations, Python doesn't immediately need to resolve `Post`.

---

## 5. SQLAlchemy `User` Model

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(
        Integer,
        primary_key=True,
        index=True,
    )

    username: Mapped[str] = mapped_column(
        String(50),
        unique=True,
        nullable=False,
    )

    email: Mapped[str] = mapped_column(
        String(120),
        unique=True,
        nullable=False,
    )

    image_file: Mapped[str | None] = mapped_column(
        String(200),
        nullable=True,
        default=None,
    )

    posts: Mapped[list[Post]] = relationship(
        back_populates="author"
    )
```

* `__tablename__` → database table name.
* `Mapped[int]` → Python/SQLAlchemy type annotation.
* `mapped_column()` → defines a database column.
* `primary_key=True` → primary key.
* `index=True` → creates an index for faster lookups.
* `unique=True` → value must be unique.
* `nullable=False` → value is required.
* `nullable=True` → value can be `NULL`.
* `relationship()` → defines a relationship between models.

### `image_path` Property

```python
@property
def image_path(self) -> str:
    if self.image_file:
        return f"/media/profile_pics/{self.image_file}"

    return "/static/profile_pics/default.jpg"
```

`image_path` is a **computed Python property**. It is not stored as a database column.

It returns a different path depending on whether the user has an `image_file`.

---

## 6. SQLAlchemy `Post` Model

```python
class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(
        Integer,
        primary_key=True,
        index=True,
    )

    title: Mapped[str] = mapped_column(
        String(100),
        nullable=False,
    )

    content: Mapped[str] = mapped_column(
        Text,
        nullable=False,
    )

    user_id: Mapped[int] = mapped_column(
        ForeignKey("users.id"),
        nullable=False,
        index=True,
    )

    date_posted: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(UTC),
    )

    author: Mapped[User] = relationship(
        back_populates="posts"
    )
```

### Foreign Key

```python
ForeignKey("users.id")
```

means `posts.user_id` references `users.id`.

This creates the relationship:

```text
User
  │
  └── posts
       │
       ├── Post
       ├── Post
       └── Post
```

One **User** can have many **Posts**.

### `relationship()`

```python
posts: Mapped[list[Post]]
```

A user has a list of posts.

```python
author: Mapped[User]
```

A post has one author.

`back_populates` connects these two sides of the relationship.

---

## 7. Pydantic User Schemas

```python
class UserBase(BaseModel):
    username: str = Field(min_length=1, max_length=50)
    email: EmailStr = Field(max_length=120)


class UserCreate(UserBase):
    pass


class UserResponse(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    image_file: str | None
    image_path: str
```

### Why `from_attributes=True` here?

This allows Pydantic to create `UserResponse` from a SQLAlchemy `User` object.

For example:

```python
user.username
user.email
user.image_file
user.image_path
```

Pydantic can read these attributes and create the response.

### Important: `image_path`

`image_path` is **not stored in the database**.

It is a Python property calculated by the SQLAlchemy model:

```python
@property
def image_path(self):
    ...
```

Because `from_attributes=True` allows Pydantic to read object attributes, it can also read this computed property and include it in the API response.

So the flow is:

```text
Database
   ↓
SQLAlchemy User object
   ↓
image_path property is calculated
   ↓
Pydantic UserResponse
   ↓
JSON response
```

---

## 8. Nested Pydantic Schemas

```python
class PostResponse(PostBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    user_id: int
    date_posted: datetime
    author: UserResponse
```

Here:

```python
author: UserResponse
```

means the response should contain an **author object** matching the `UserResponse` schema.

Conceptually:

```json
{
    "id": 1,
    "title": "Hello",
    "content": "My post",
    "user_id": 5,
    "date_posted": "...",
    "author": {
        "id": 5,
        "username": "mainak",
        "email": "example@email.com",
        "image_file": null,
        "image_path": "/static/profile_pics/default.jpg"
    }
}
```

So yes — **Pydantic nests the `UserResponse` data inside `author`**, provided the corresponding relationship/data is available.

---

## 9. Dependency Injection

**Dependency Injection (DI)** means FastAPI can automatically provide something a route needs instead of the route creating it itself.

```python
db: Annotated[Session, Depends(get_db)]
```

Here FastAPI:

```text
Request
  ↓
Calls get_db()
  ↓
Gets database session
  ↓
Passes it to create_user()
  ↓
Route uses db
  ↓
Session is closed
```

This keeps database-session management separate from the route logic.

---

## 10. Creating a User

```python
@app.post(
    "/api/users",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
)
def create_user(
    user: UserCreate,
    db: Annotated[Session, Depends(get_db)]
):
```

### `response_model=UserResponse`

The returned SQLAlchemy object is validated/serialized according to `UserResponse`.

### `status_code=201`

A successful user creation returns **201 Created**.

### `user: UserCreate`

Pydantic validates the incoming request using `UserCreate`.


---

### Check Existing Username

```python
result = db.execute(
    select(models.User).where(
        models.User.username == user.username
    )
)

existing_user = result.scalars().first()
```

### Understanding the Query

```python
select(models.User)
```

Tells SQLAlchemy that we want to **select records from the `User` model/table**.

Equivalent SQL:

```sql
SELECT * FROM users
```

---

```python
.where(models.User.username == user.username)
```

Adds a **condition** to the query.

* `models.User.username` → the `username` column in the database.
* `user.username` → the username received from the client's request.
* `==` → checks whether both values are equal.

So this means:

> "Find users whose database username matches the username provided by the client."

Equivalent SQL:

```sql
SELECT * FROM users
WHERE username = 'requested_username'
```

---

```python
db.execute(...)
```

Sends the SQLAlchemy query to the database and returns the query result.

---

```python
result.scalars()
```

Extracts the actual `User` objects from the query result instead of returning row objects.

---

```python
.first()
```

Gets the **first matching user**.

If no user is found, it returns `None`.

So:

```python
existing_user = result.scalars().first()
```

means:

> "Get the first user whose username matches the requested username."

Then:

```python
if existing_user:
    raise HTTPException(
        status_code=status.HTTP_400_BAD_REQUEST,
        detail="Username already exists",
    )
```

checks whether a matching user was found. If yes, the request is rejected.

The **same query pattern** is used to check whether the email already exists.

---

### Create the User

```python
new_user = models.User(
    username=user.username,
    email=user.email,
)
```

Creates a new SQLAlchemy `User` object.

### Add to Session

```python
db.add(new_user)
```

Adds the object to the current database session.

### Commit

```python
db.commit()
```

Persists the changes to the database.

### Refresh

```python
db.refresh(new_user)
```

Reloads the object from the database so that newly generated values, such as the `id`, are available.

### Return

```python
return new_user
```

FastAPI then uses `UserResponse` to serialize the SQLAlchemy object into the API response.

---

## 11. `Base.metadata.create_all(engine)`

```python
Base.metadata.create_all(engine)
```

This tells SQLAlchemy to **create all tables defined by the models registered with `Base` if they don't already exist**.

For example:

```text
User model → users table
Post model → posts table
```

The `engine` tells SQLAlchemy **which database** to create the tables in.

It generally does not recreate tables that already exist.

## 🧠 Key Takeaways

* **Pydantic** → validates API input/output.
* **SQLAlchemy** → communicates with the database.
* **Session** → manages a unit of database interaction.
* `get_db()` → provides a session and ensures it closes.
* `Base` → parent class for SQLAlchemy models.
* `ForeignKey` → connects tables.
* `relationship()` → represents relationships between SQLAlchemy objects.
* `from_attributes=True` → lets Pydantic read SQLAlchemy object attributes/properties.
* Nested Pydantic models → create nested API responses.
* **Dependency Injection** → FastAPI provides dependencies automatically.
* `db.add()` → adds an object to the session.
* `db.commit()` → saves changes.
* `db.refresh()` → reloads the object with database-generated values.
* `Base.metadata.create_all(engine)` → creates the defined database tables if they don't exist.
