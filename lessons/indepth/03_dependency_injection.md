# FastAPI Dependency Injection — Complete Notes

> A practical guide to understanding Dependency Injection in FastAPI, especially if you're coming from JavaScript / React / Express.

---

## What is Dependency Injection?

**Dependency Injection (DI)** is a design pattern where a function or component **receives the things it needs from an external system instead of creating them itself**.

### Without Dependency Injection

Imagine an API endpoint needs:

* Authentication token
* Database connection
* Current user

Without DI, the endpoint might manually do everything:

```python
@router.get("/me")
async def get_current_user(request):
    token = extract_token(request)

    user_id = verify_token(token)

    db = create_database_connection()

    user = await get_user(db, user_id)

    return user
```

Now every protected endpoint may have to repeat the same logic.

```text
/me
  ↓
extract token
verify token
connect DB
find user

/profile
  ↓
extract token
verify token
connect DB
find user

/settings
  ↓
extract token
verify token
connect DB
find user
```

This becomes repetitive and difficult to maintain.

---

# What Does Dependency Injection Solve?

With FastAPI's Dependency Injection system, we can tell FastAPI:

> "This function needs these things. You take care of getting them and give them to me."

For example:

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],
):
    ...
```

The function doesn't need to manually create or obtain:

* `token`
* `db`

FastAPI provides them.

Conceptually:

```text
                    get_current_user()
                           ↑
                           │
                    FastAPI provides
                           │
                ┌──────────┴──────────┐
                │                     │
              token                   db
                ↑                     ↑
        oauth2_scheme()            get_db()
```

---

# Dependency Injection in JavaScript

If you come from React, you've already seen a similar idea.

Consider:

```jsx
function UserProfile({ user }) {
    return <h1>{user.name}</h1>;
}
```

`UserProfile` needs a `user`.

But it doesn't create the user itself.

Someone else provides it:

```jsx
<UserProfile user={user} />
```

Conceptually:

```text
UserProfile
     ↑
     │
    user
     │
Parent provides it
```

FastAPI works similarly.

```python
async def profile(
    user: Annotated[User, Depends(get_current_user)]
):
    return user
```

The endpoint says:

> "I need a `user`."

FastAPI says:

> "I'll resolve `get_current_user` and give you the result."

---

# A Simple FastAPI Dependency

Let's start with a very simple example.

```python
from fastapi import Depends, FastAPI

app = FastAPI()


def get_name():
    return "Mainak"


@app.get("/")
def hello(name: str = Depends(get_name)):
    return f"Hello {name}"
```

Here:

```python
Depends(get_name)
```

means:

> FastAPI, get this value by executing `get_name`.

You don't manually call:

```python
get_name()
```

FastAPI does it.

---

# What is `Depends()`?

`Depends()` is FastAPI's way of declaring a dependency.

```python
Depends(get_db)
```

means:

> This value should be provided by the `get_db` dependency.

For example:

```python
def get_db():
    db = create_database_connection()

    try:
        yield db
    finally:
        db.close()
```

Then:

```python
@app.get("/users")
async def get_users(
    db = Depends(get_db)
):
    ...
```

FastAPI handles calling `get_db` and injecting its result into `db`.

---

# Dependency Injection Flow

The general flow looks like this:

```text
                 HTTP Request
                       │
                       ▼
                  FastAPI
                       │
                       ▼
              Inspect endpoint
                       │
                       ▼
             Find dependencies
                       │
              ┌────────┴────────┐
              ▼                 ▼
          get_db()        get_current_user()
              │                 │
              ▼                 ▼
              db              user
              │                 │
              └────────┬────────┘
                       ▼
                  Endpoint()
                       │
                       ▼
                    Response
```

FastAPI automatically resolves the dependencies before executing the endpoint.

---

# What is `Annotated`?

Now let's look at this syntax:

```python
token: Annotated[str, Depends(oauth2_scheme)]
```

This can look strange if you're coming from JavaScript.

Let's break it down.

---

## The Parameter

```python
token
```

This is simply the parameter name.

---

## The Type

```python
token: str
```

This means:

> `token` is expected to be a string.

If you're coming from TypeScript:

```typescript
function getCurrentUser(token: string) {
}
```

is conceptually similar to:

```python
def get_current_user(token: str):
    ...
```

---

# What Does `Annotated` Do?

Python's `Annotated` allows you to attach **additional metadata** to a type.

For example:

```python
Annotated[str, Depends(oauth2_scheme)]
```

contains two pieces of information:

```text
Annotated[
    str,
    Depends(oauth2_scheme)
]
   │
   ├── Type → str
   │
   └── Metadata → Depends(oauth2_scheme)
```

So:

```python
token: Annotated[str, Depends(oauth2_scheme)]
```

can be mentally read as:

> `token` is a string, and FastAPI should obtain it using `oauth2_scheme`.

Other example of Annotated
```python
from typing import Annotated

Age = Annotated[int, "must be positive"]
```
By itself, Python doesn't enforce "must be positive". A library can read that metadata and decide what to do with it.

---

# `Annotated` vs `Depends`

You may see both styles.

### Traditional style

```python
token: str = Depends(oauth2_scheme)
```

### Annotated style

```python
token: Annotated[str, Depends(oauth2_scheme)]
```

Both express the dependency relationship.

The `Annotated` style keeps the type information and dependency metadata separate.

For modern FastAPI code, you'll commonly see:

```python
Annotated[SomeType, Depends(some_dependency)]
```

---

# Dependency Trees

One of FastAPI's powerful features is that dependencies can themselves have dependencies.

For example:

```python
def get_db():
    ...
```

Then:

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)]
):
    ...
```

Then:

```python
async def get_admin_user(
    user: Annotated[
        User,
        Depends(get_current_user)
    ]
):
    ...
```

The dependency tree becomes:

```text
get_admin_user
       │
       ▼
get_current_user
       │
       ├───────────────┐
       ▼               ▼
oauth2_scheme()      get_db()
       │               │
       ▼               ▼
     token             db
```

FastAPI resolves this tree automatically.

---

# What is `OAuth2PasswordBearer`?

Now let's move to authentication.

You might have:

```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="api/users/token"
)
```

`oauth2_scheme` is **not the token**.

It is a **dependency that knows how to extract the Bearer token from an incoming HTTP request**.

---

# Where Does the Token Come From?

Suppose your frontend sends:

```http
GET /api/users/me
Authorization: Bearer eyJhbGciOiJIUzI1Ni...
```

The token is inside the HTTP request.

Specifically:

```text
Authorization: Bearer <TOKEN>
```

`OAuth2PasswordBearer` extracts the token:

```text
Authorization
     ↓
Bearer eyJhbGciOiJIUzI1Ni...
     ↓
oauth2_scheme
     ↓
eyJhbGciOiJIUzI1Ni...
```

So the value injected into:

```python
token
```

is the actual JWT.

---

# How Does This Line Work?

Consider:

```python
token: Annotated[str, Depends(oauth2_scheme)]
```

FastAPI sees:

```python
Depends(oauth2_scheme)
```

and understands:

> "I need to execute `oauth2_scheme` and use its return value as the `token`."

Conceptually:

```python
token = oauth2_scheme(request)

get_current_user(token)
```

You don't manually execute `oauth2_scheme`.

FastAPI's Dependency Injection system does it.

---

# The Frontend Side

Your frontend might send:

```javascript
fetch("/api/users/me", {
    headers: {
        Authorization: `Bearer ${accessToken}`
    }
});
```

The actual HTTP request becomes:

```http
GET /api/users/me
Authorization: Bearer abc123
```

FastAPI receives this request.

Then:

```python
Depends(oauth2_scheme)
```

extracts:

```text
abc123
```

and injects it into:

```python
token
```

---

# What is `tokenUrl`?

You may have:

```python
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="api/users/token"
)
```

The `tokenUrl` is one of the most confusing parts of this API.

### Important:

`tokenUrl` does **NOT** mean:

> "Call this endpoint and get the token."

It does not automatically make a request.

It tells FastAPI/OpenAPI:

> "This API obtains OAuth2 tokens from this endpoint."

---

# Does `tokenUrl` Call the Endpoint?

### No.

This:

```python
OAuth2PasswordBearer(
    tokenUrl="api/users/token"
)
```

does NOT perform:

```text
/api/users/token
       ↓
get token
       ↓
/api/users/me
```

Instead, the client/frontend calls the token endpoint.

---

# Actual Authentication Flow

There are two separate operations.

## Step 1 — Obtain the Token

The frontend sends credentials:

```text
POST /api/users/token
       ↓
username + password
       ↓
Backend verifies credentials
       ↓
Backend creates JWT
       ↓
Returns access_token
```

For example:

```json
{
    "access_token": "eyJhbGciOiJIUzI1Ni...",
    "token_type": "bearer"
}
```

---

## Step 2 — Use the Token

The frontend then makes a protected request:

```text
GET /api/users/me

Authorization: Bearer eyJhbGciOiJIUzI1Ni...
```

FastAPI:

```text
Request
   ↓
oauth2_scheme
   ↓
Extract JWT
   ↓
token
   ↓
verify_access_token(token)
   ↓
user_id
   ↓
Database
   ↓
User
```

---

# Why Does `tokenUrl` Exist?

`tokenUrl` is primarily used as metadata for OAuth2/OpenAPI documentation.

For example:

```python
OAuth2PasswordBearer(
    tokenUrl="api/users/token"
)
```

tells the generated API documentation:

> "The OAuth2 token endpoint is `/api/users/token`."

This allows tools such as Swagger UI to understand how authentication is configured.

It does not execute that endpoint during every authenticated request.

---

# Complete Authentication Example

You might have:

```python
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="api/users/token"
)
```

Then:

```python
@router.get("/me")
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],
):
    user_id = verify_access_token(token)

    result = await db.execute(
        select(models.User).where(
            models.User.id == user_id
        )
    )

    user = result.scalars().first()

    return user
```

The dependency relationships are:

```text
get_current_user
       │
       ├───────────────┐
       │               │
       ▼               ▼
oauth2_scheme()      get_db()
       │               │
       ▼               ▼
    token              db
```

---

# Complete Request Flow

Suppose the frontend sends:

```http
GET /api/users/me
Authorization: Bearer abc123
```

### Step 1 — Request arrives

```text
HTTP Request
     ↓
FastAPI
```

### Step 2 — FastAPI sees the endpoint

```python
get_current_user(...)
```

### Step 3 — FastAPI finds dependencies

```python
Depends(oauth2_scheme)
Depends(get_db)
```

### Step 4 — Resolve OAuth dependency

```text
oauth2_scheme
      ↓
Authorization header
      ↓
Bearer abc123
      ↓
token = "abc123"
```

### Step 5 — Resolve database dependency

```text
get_db()
    ↓
AsyncSession
    ↓
db
```

### Step 6 — Call your endpoint

Conceptually:

```python
get_current_user(
    token="abc123",
    db=db
)
```

### Step 7 — Verify token

```python
user_id = verify_access_token(token)
```

### Step 8 — Find user

```python
select(models.User).where(
    models.User.id == user_id
)
```

### Step 9 — Return user

```text
User
 ↓
FastAPI
 ↓
JSON Response
```

---

# Dependency Injection vs Middleware

Dependency Injection and middleware are **not the same thing**.

They can sometimes solve similar problems, especially authentication, but their mechanisms are different.

---

# What is Middleware?

If you're coming from Express, you probably know:

```javascript
app.use(authMiddleware);
```

For example:

```javascript
function authMiddleware(req, res, next) {
    const token = req.headers.authorization;

    // Validate token

    req.user = user;

    next();
}
```

Middleware sits inside the request pipeline:

```text
Request
   ↓
Middleware
   ↓
Middleware
   ↓
Route Handler
   ↓
Response
```

Middleware is useful for things such as:

* Logging
* CORS
* Request processing
* Authentication
* Adding/modifying request information
* Running logic before/after requests

---

# What is Dependency Injection?

Dependency Injection is about providing a function with the things it needs.

```python
async def profile(
    user: Annotated[
        User,
        Depends(get_current_user)
    ]
):
    return user
```

The endpoint says:

> "I need a `user`."

FastAPI resolves:

```text
get_current_user()
       ↓
      user
       ↓
profile(user)
```

---

# The Key Difference

### Middleware

Think:

> **"Run this during the request pipeline."**

```text
Request
   ↓
Middleware
   ↓
Route
```

### Dependency Injection

Think:

> **"This function needs something. Provide it."**

```text
Function
   ↑
   │
FastAPI provides dependency
```

---

# The Four Things to Remember

If you're learning FastAPI Dependency Injection, remember these four concepts.

---

## `Depends()`

```python
Depends(get_db)
```

Means:

> **FastAPI, get this value by executing `get_db`.**

---

## `Annotated`

```python
Annotated[AsyncSession, Depends(get_db)]
```

Means:

> **This parameter is an `AsyncSession`, and FastAPI should obtain it through this dependency.**

---

## `OAuth2PasswordBearer`

```python
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="api/users/token"
)
```

Means:

> **Create a dependency that extracts a Bearer token from the incoming request.**

---

## `tokenUrl`

```python
tokenUrl="api/users/token"
```

Means:

> **The OAuth2 token-issuing endpoint is located here.**

It does **not** call the endpoint automatically.

---

# The Most Important Mental Model

Whenever you see:

```python
token: Annotated[
    str,
    Depends(oauth2_scheme)
]
```

translate it in your head to:

> **"FastAPI, I need a string called `token`. Please obtain it using `oauth2_scheme` and inject the result here."**

And when you see:

```python
db: Annotated[
    AsyncSession,
    Depends(get_db)
]
```

translate it to:

> **"FastAPI, I need a database session called `db`. Please obtain it using `get_db` and inject the result here."**

---

# Final Mental Model

Keep this diagram in your head:

```text
                       HTTP REQUEST
                            │
                            ▼
                 GET /api/users/me
                 Authorization: Bearer JWT
                            │
                            ▼
                      ┌──────────┐
                      │ FastAPI  │
                      └────┬─────┘
                           │
                   Resolve dependencies
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       oauth2_scheme()               get_db()
              │                         │
              ▼                         ▼
         Extract JWT              Get DB session
              │                         │
              ▼                         ▼
           token                       db
              │                         │
              └────────────┬────────────┘
                           ▼
                  get_current_user(
                      token,
                      db
                  )
                           │
                           ▼
                 verify_access_token()
                           │
                           ▼
                        user_id
                           │
                           ▼
                    Query database
                           │
                           ▼
                         User
                           │
                           ▼
                       Response
```

### One-line summary

**Dependency Injection = "Tell FastAPI what your function needs, and FastAPI figures out how to obtain and provide it."**
