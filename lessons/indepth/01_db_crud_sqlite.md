# FastAPI — Database & CRUD Architecture

This note explains the complete flow we built while connecting **SQLite + SQLAlchemy + Pydantic** to our FastAPI blog API.

The goal is not just to understand the code, but to understand **why each layer exists and how a real backend handles requests**.

---

# 1. The Big Picture

When a client creates or retrieves a blog post:

```text
                Client
                  │
                  │ HTTP Request
                  ▼
             FastAPI Route
                  │
                  ▼
             Pydantic
          Request Validation
                  │
                  ▼
          Dependency Injection
                  │
                  ▼
            DB Session
                  │
                  ▼
             SQLAlchemy
                  │
                  ▼
              Database
                  │
                  │ Data
                  ▼
             SQLAlchemy
              ORM Object
                  │
                  ▼
             Pydantic
          Response Validation
                  │
                  ▼
             JSON Response
                  │
                  ▼
                Client
```

Each layer has a different responsibility.

---

# 2. Step 1 — Choose and Set Up the Database

For our project we use **SQLite**.

```python
SQLALCHEMY_DATABASE_URL = "sqlite:///./blog.db"
```

SQLite stores the database in a file:

```text
blog.db
```

For a small learning project, SQLite is convenient because it doesn't require a separate database server.

In a production application, databases such as **PostgreSQL** are commonly used when we need stronger concurrency, scalability, reliability, and operational capabilities.

---

# 3. Step 2 — Create the SQLAlchemy Engine

```python
from sqlalchemy import create_engine

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False},
)
```

The **engine** is SQLAlchemy's main interface to the database.

Think of it as the component responsible for managing communication between:

```text
Application
     ↓
SQLAlchemy Engine
     ↓
Database
```

The engine also manages database connections and, for databases that support it, connection pooling.

> In simple word engine is basically the thing that knows how to connect your application to the database and manage those connections.

---

# 4. Step 3 — Create the Database Models

We define what our database tables look like using SQLAlchemy models.

For example:

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(
        Integer,
        primary_key=True
    )

    username: Mapped[str] = mapped_column(
        String(50),
        unique=True,
        nullable=False
    )
```

This describes the `users` table.

Conceptually:

```text
users
--------------------------------
id          INTEGER PRIMARY KEY
username    VARCHAR(50) UNIQUE
email       VARCHAR(120)
```

Similarly, we create a `Post` model.

---

# 5. What Is a Database Schema?

A **database schema** describes the structure of the data stored in the database.

For example:

```text
users
 ├── id
 ├── username
 ├── email
 └── image_file

posts
 ├── id
 ├── title
 ├── content
 ├── user_id
 └── date_posted
```

It defines things such as:

* Tables
* Columns
* Data types
* Primary keys
* Foreign keys
* Constraints
* Relationships
* Indexes

### Important distinction

We have two different kinds of schemas in this project.

### Database schema

Defined using SQLAlchemy:

```python
class User(Base):
    ...
```

It defines **how data is stored**.

### API/Pydantic schema

Defined using Pydantic:

```python
class UserCreate(BaseModel):
    ...
```

It defines **what data our API accepts or returns**.

They solve different problems.

---

# 6. Why Do We Need Pydantic?

Suppose the client sends:

```json
{
    "username": "mainak",
    "email": "hello@example.com"
}
```

Our Pydantic model:

```python
class UserCreate(BaseModel):
    username: str
    email: EmailStr
```

validates the incoming data.

Pydantic answers:

> "Is this data valid for my API?"

SQLAlchemy answers:

> "How do I store or retrieve this data from the database?"

So:

```text
Client Data
     ↓
Pydantic
     ↓
Validated Python data
     ↓
SQLAlchemy
     ↓
Database
```

---

# 7. Why Not Use the Database Model for API Validation?

Because the database and API have different responsibilities.

For example, our database `User` model might contain:

```text
id
username
email
image_file
```

But when creating a user, the client shouldn't provide:

```json
{
    "id": 5000
}
```

The database should generate the ID.

Similarly, the database might contain internal fields that we don't want to expose through the API.

That's why we create separate Pydantic models:

```text
UserCreate
UserResponse
UserUpdate
```

This gives us control over what enters and leaves our API.

---

# 8. Step 4 — Create the Session Factory

```python
from sqlalchemy.orm import sessionmaker

SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine,
)
```

`sessionmaker` creates a **factory** for database sessions.

It does not create one database session here.

Instead:

```text
SessionLocal
     │
     ├── creates Session 1
     ├── creates Session 2
     ├── creates Session 3
     └── ...
```

---

# 9. What Is a Database Session?

A SQLAlchemy **Session** is the object through which our application performs database operations.

For example:

```python
db.execute(...)
db.add(...)
db.commit()
db.refresh(...)
```

The session manages the interaction between our Python objects and the database.

It also tracks changes to SQLAlchemy objects.

Think of it as a **unit of work**.

For example:

```text
Session starts
     ↓
Read data
     ↓
Modify data
     ↓
Commit
     ↓
Session ends
```

---

# 10. Session ≠ Database Connection

These concepts are related but not identical.

```text
Application
     ↓
SQLAlchemy Session
     ↓
SQLAlchemy Engine
     ↓
Database Connection
     ↓
Database
```

The **Session** manages ORM-level work.

The **Engine** manages the lower-level database connectivity and, where applicable, the connection pool.

This distinction becomes important when we discuss scalability.

---

# 11. Step 5 — Create `get_db()`

```python
def get_db():
    with SessionLocal() as db:
        yield db
```

This function creates a session and gives it to the API route.

The `with` statement ensures the session is closed when the request is finished.

Conceptually:

```text
Request starts
     ↓
Create Session
     ↓
Give Session to route
     ↓
Route performs DB operations
     ↓
Request finishes
     ↓
Close Session
```

---

# 12. Why Do We Need a Session for Each Request?

Imagine 100 users sending requests at approximately the same time.

We don't want:

```text
100 requests
      ↓
One shared Session
      ↓
Database
```

Instead, we want each request to have its own unit of database work:

```text
Request 1 → Session 1 ─┐
Request 2 → Session 2 ─┤
Request 3 → Session 3 ─┤
Request 4 → Session 4 ─┤
                       ▼
                 Database
```

This prevents different requests from interfering with each other's ORM state and transaction boundaries.

---

# 13. What Would Happen If We Used One Global Session?

Imagine:

```python
db = SessionLocal()
```

and every request used that same session.

Now:

```text
User A ──┐
User B ──┤
User C ──┼──> SAME SESSION
User D ──┤
User E ──┘
```

This creates serious problems.

### Problem 1 — Shared State

The session tracks ORM objects and their changes.

User A could modify an object while User B is also interacting with the same session.

This creates unwanted shared state.

### Problem 2 — Transaction Conflicts

Suppose:

```text
User A → begins transaction
User B → uses same session
User C → commits
```

Now different requests are effectively sharing the same transaction boundary.

That's not what we want.

### Problem 3 — Errors Affect Other Requests

If a database operation fails and the session enters a failed transaction state, other requests using the same session can be affected.

### Problem 4 — Concurrency

Requests can execute concurrently.

A single mutable Session is not designed to be shared across concurrent request handling.

### Problem 5 — Security / Data Isolation

One request should not accidentally see uncommitted or tracked state belonging to another request.

---

# 14. What About 10,000 Users?

This is where an important distinction appears.

**10,000 users does NOT necessarily mean 10,000 database connections.**

For example:

```text
10,000 users
      ↓
Many HTTP requests
      ↓
FastAPI workers
      ↓
SQLAlchemy
      ↓
Connection Pool
      ↓
Limited number of DB connections
      ↓
Database
```

The application can have many concurrent requests while the database connection pool limits how many database connections are actively used.

The exact numbers depend on:

* Number of application workers
* Database configuration
* Connection pool size
* Query duration
* Traffic pattern
* Database capacity
* Infrastructure

---

# 15. Connection Pooling

A production database usually uses **connection pooling**.

Instead of opening a completely new database connection for every query:

```text
Request
   ↓
Open connection
   ↓
Query
   ↓
Close connection
```

we can maintain a pool:

```text
Connection Pool

┌─────────────┐
│ Connection 1│
│ Connection 2│
│ Connection 3│
│ Connection 4│
│ Connection 5│
└─────────────┘
```

Requests borrow connections when needed and return them to the pool afterward.

This avoids the overhead of constantly creating new connections.

---

# 16. Session + Connection Pool

The architecture is roughly:

```text
HTTP Request
     ↓
FastAPI
     ↓
Dependency Injection
     ↓
SQLAlchemy Session
     ↓
SQLAlchemy Engine
     ↓
Connection Pool
     ↓
Database Connection
     ↓
PostgreSQL / SQLite / etc.
```

The important idea is:

> **One request gets its own Session, but Sessions do not imply one permanent database connection per user.**

---

# 17. Step 6 — Dependency Injection

We use:

```python
db: Annotated[Session, Depends(get_db)]
```

FastAPI sees:

```python
Depends(get_db)
```

and automatically calls `get_db()` for us.

This is **Dependency Injection**.

Instead of writing:

```python
def create_user():
    db = SessionLocal()

    # database work

    db.close()
```

inside every route, we define the dependency once:

```python
def get_db():
    with SessionLocal() as db:
        yield db
```

and FastAPI injects it where needed.

This improves:

* Reusability
* Separation of concerns
* Testing
* Maintainability

---

# 18. Step 7 — Request Enters the API

Suppose we have:

```python
@app.post("/api/users")
def create_user(
    user: UserCreate,
    db: Annotated[Session, Depends(get_db)]
):
    ...
```

The request might be:

```json
{
    "username": "mainak",
    "email": "hello@example.com"
}
```

FastAPI first processes the request.

---

# 19. Step 8 — Pydantic Validates the Request

Because we wrote:

```python
user: UserCreate
```

FastAPI uses Pydantic to validate the request.

If the request is valid:

```text
JSON
 ↓
Pydantic
 ↓
UserCreate object
```

If invalid:

```text
JSON
 ↓
Pydantic
 ↓
Validation Error
 ↓
422 Response
```

The route doesn't need to manually validate every field.

---

# 20. Step 9 — Check the Database

We can now use SQLAlchemy:

```python
result = db.execute(
    select(models.User).where(
        models.User.username == user.username
    )
)

existing_user = result.scalars().first()
```

The query means:

```text
Find a User
where
database username == username from request
```

If the user exists:

```python
raise HTTPException(
    status_code=status.HTTP_400_BAD_REQUEST,
    detail="Username already exists",
)
```

---

# 21. Step 10 — Create the SQLAlchemy Object

If the user doesn't already exist:

```python
new_user = models.User(
    username=user.username,
    email=user.email,
)
```

Now we have a SQLAlchemy object.

```text
Pydantic object
      ↓
SQLAlchemy object
      ↓
Database
```

---

# 22. Step 11 — Add and Commit

```python
db.add(new_user)
```

Adds the object to the current session.

Then:

```python
db.commit()
```

commits the transaction and persists the changes to the database.

---

# 23. Step 12 — Refresh

```python
db.refresh(new_user)
```

Reloads the object using the database's current state.

This is useful when the database generated values such as:

```text
id
timestamps
defaults
```

during the insert.

---

# 24. Step 13 — Return the SQLAlchemy Object

```python
return new_user
```

We're returning a SQLAlchemy object, not manually constructing JSON.

But our route has:

```python
response_model=UserResponse
```

FastAPI uses the Pydantic response model to validate/serialize the returned data.

---

# 25. Response Flow

The final flow becomes:

```text
SQLAlchemy User object
        ↓
UserResponse
        ↓
JSON
        ↓
Client
```

For example:

```json
{
    "id": 1,
    "username": "mainak",
    "email": "hello@example.com",
    "image_file": null,
    "image_path": "/static/profile_pics/default.jpg"
}
```

---

# 26. Why `from_attributes=True`?

Our SQLAlchemy object has attributes:

```python
user.id
user.username
user.email
user.image_path
```

But Pydantic needs to know that it is allowed to read data from object attributes.

That's why we use:

```python
model_config = ConfigDict(
    from_attributes=True
)
```

The flow is:

```text
SQLAlchemy object
       ↓
Read object attributes
       ↓
Pydantic UserResponse
       ↓
JSON
```

This is especially useful when working with ORM objects.

---

# 27. Relationships

Our models define:

```python
class User(Base):
    posts: Mapped[list[Post]] = relationship(
        back_populates="author"
    )
```

and:

```python
class Post(Base):
    author: Mapped[User] = relationship(
        back_populates="posts"
    )
```

Together they represent:

```text
User
 │
 ├── Post
 ├── Post
 └── Post
```

The database uses:

```python
user_id = mapped_column(
    ForeignKey("users.id")
)
```

to establish the actual foreign-key relationship.

---

# 28. Nested Pydantic Response

We can define:

```python
class PostResponse(PostBase):
    author: UserResponse
```

This means the API response contains a nested author object.

Conceptually:

```json
{
    "id": 1,
    "title": "My Post",
    "content": "Hello",
    "author": {
        "id": 5,
        "username": "mainak",
        "email": "hello@example.com"
    }
}
```

Pydantic validates the structure of the nested object as well.

---

# 29. CRUD Architecture

We now have all four CRUD operations.

### Create

```text
POST
 ↓
Validate
 ↓
SQLAlchemy INSERT
 ↓
Commit
 ↓
Return created object
```

### Read

```text
GET
 ↓
Validate path/query parameters
 ↓
SQLAlchemy SELECT
 ↓
Return data
```

### Update

```text
PUT/PATCH
 ↓
Validate request
 ↓
Find existing record
 ↓
Modify SQLAlchemy object
 ↓
Commit
 ↓
Return updated object
```

### Delete

```text
DELETE
 ↓
Find record
 ↓
Delete SQLAlchemy object
 ↓
Commit
 ↓
Return response
```

---

# 30. `PUT` vs `PATCH`

### PUT

Generally represents a **full replacement/update**.

```json
{
    "title": "New title",
    "content": "New content"
}
```

### PATCH

Generally represents a **partial update**.

```json
{
    "title": "New title"
}
```

That's why we created:

```python
class PostUpdate(BaseModel):
    title: str | None = None
    content: str | None = None
```

and:

```python
post_data.model_dump(exclude_unset=True)
```

Only fields actually provided by the client are updated.

---

# 31. What Happens During a Request?

Let's put everything together.

Suppose:

```text
POST /api/users
```

with:

```json
{
    "username": "mainak",
    "email": "hello@example.com"
}
```

### Complete flow

```text
1. Client sends HTTP request
            ↓
2. FastAPI receives request
            ↓
3. Pydantic validates request
            ↓
4. FastAPI resolves Depends(get_db)
            ↓
5. get_db() creates a Session
            ↓
6. Route receives Session
            ↓
7. SQLAlchemy queries database
            ↓
8. SQLAlchemy creates User object
            ↓
9. db.add()
            ↓
10. db.commit()
            ↓
11. db.refresh()
            ↓
12. Route returns SQLAlchemy object
            ↓
13. Pydantic UserResponse validates/serializes it
            ↓
14. FastAPI creates JSON response
            ↓
15. Session is closed
            ↓
16. Response goes to client
```

---

# 32. What Happens With 10,000 Users?

Imagine 10,000 users are using the application.

The architecture should look more like:

```text
                  10,000 Users
                       │
                       ▼
                  Load Balancer
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      FastAPI       FastAPI       FastAPI
      Worker 1      Worker 2      Worker 3
          │            │            │
          ▼            ▼            ▼
       Sessions     Sessions     Sessions
          │            │            │
          └────────────┼────────────┘
                       ▼
                Connection Pool
                       │
                       ▼
                  PostgreSQL
```

The important scaling techniques include:

* Multiple application workers/instances
* Database connection pooling
* Efficient SQL queries
* Proper indexes
* Caching
* Pagination
* Rate limiting
* Background jobs
* Load balancing
* Database replication when needed
* Monitoring and observability

You don't solve 10,000 users simply by creating 10,000 sessions or 10,000 connections.

---

# 33. Why SQLite Isn't Usually the Final Production Choice

SQLite is excellent for:

* Learning
* Small applications
* Local development
* Prototypes
* Simple workloads

But production applications with significant concurrent traffic often use a server-based relational database such as PostgreSQL.

The architecture remains similar:

```text
FastAPI
   ↓
SQLAlchemy
   ↓
PostgreSQL
```

We can learn FastAPI and SQLAlchemy using SQLite and later switch the database configuration to PostgreSQL.

---

# 34. Important System Design Mental Model

When building a backend, think in **layers**:

```text
┌──────────────────────────────┐
│           Client             │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        API / FastAPI         │
│     Routing + HTTP layer     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│          Pydantic            │
│     Input / Output Schema    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Business Logic         │
│    Application / Services    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│         SQLAlchemy           │
│       ORM / Data Access      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│          Database            │
│       PostgreSQL / SQLite    │
└──────────────────────────────┘
```

Our current tutorial is still relatively simple. In a larger application, we would normally separate **routes, schemas, services, repositories/data-access code, models, and database configuration** instead of putting everything into one file.

---

# 🧠 Final Mental Model

Remember these responsibilities:

| Component                | Main Responsibility                      |
| ------------------------ | ---------------------------------------- |
| **FastAPI**              | HTTP/API layer                           |
| **Pydantic**             | Validate and serialize API data          |
| **SQLAlchemy**           | Communicate with the database            |
| **Session**              | Manage a unit of database work           |
| **Engine**               | Manage database connectivity             |
| **Connection Pool**      | Reuse/manage database connections        |
| **Database Model**       | Define how data is stored                |
| **Database**             | Persist the actual data                  |
| **Dependency Injection** | Provide dependencies such as DB sessions |
| **Foreign Key**          | Connect records between tables           |
| **Relationship**         | Represent related objects in SQLAlchemy  |
| **Response Model**       | Control/validate API output              |

### The most important distinction

```text
Pydantic Schema
     ↓
"What can enter/leave my API?"

SQLAlchemy Model
     ↓
"How is my data represented/stored?"

Session
     ↓
"How does this request interact with the database?"

Engine / Pool
     ↓
"How does my application connect to the database?"

Database
     ↓
"Where is the data actually stored?"
```

That separation is the foundation of the architecture we've been building.
