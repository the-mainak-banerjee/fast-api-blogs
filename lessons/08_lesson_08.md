# Lesson 8: Routers — Organizing Routes into Modules with `APIRouter`

**Lesson:** 8
**Video:** https://youtu.be/NkgIHa6KtHg?si=tdDGAGI8ChSTPlqJ

---

## 1. What is `APIRouter`?

`APIRouter` helps us **organize related API routes into separate modules/files** instead of keeping all routes inside `main.py`.

For example:

```text
fastapi_blog/
├── main.py
└── routers/
    ├── users.py
    └── posts.py
```

`users.py` can contain user-related routes, while `posts.py` contains post-related routes.

---

## 2. Creating an `APIRouter`

```python
from fastapi import APIRouter

router = APIRouter()
```

We can then define routes using `router` instead of `app`:

```python
@router.get("/")
def get_users():
    ...
```

---

## 3. Including a Router

In `main.py`:

```python
app.include_router(
    users.router,
    prefix="/api/users",
    tags=["users"]
)
```

### `users.router`

The router defined inside `users.py`:

```python
router = APIRouter()
```

### `prefix="/api/users"`

Adds this prefix to **every route inside the router**.

For example:

```python
@router.get("/")
def get_users():
    ...
```

becomes:

```text
GET /api/users/
```

And:

```python
@router.get("/{user_id}")
def get_user(user_id: int):
    ...
```

becomes:

```text
GET /api/users/{user_id}
```

### `tags=["users"]`

Groups these routes under **users** in the Swagger/OpenAPI documentation.

---

## 4. Why Use Routers?

Without routers:

```text
main.py
 ├── User routes
 ├── Post routes
 ├── Authentication routes
 ├── Comment routes
 └── ...
```

As the application grows, `main.py` becomes difficult to maintain.

With routers:

```text
main.py
   │
   ├── users.py
   ├── posts.py
   ├── comments.py
   └── auth.py
```

Each module is responsible for its own routes.

> **`APIRouter` = a way to group and organize related routes into separate modules.**

---

## 5. Important: Route Names Must Be Unique

FastAPI uses the **function name as the route name by default**.

For example:

```python
@router.get("/")
def home():
    ...
```

The route name is:

```text
home
```

So avoid creating another route with the same function name if you rely on route names, especially when using `url_for()`.

For example, don't unnecessarily have:

```python
# users.py

def home():
    ...
```

and:

```python
# main.py

def home():
    ...
```

Instead, use descriptive and unique function names:

```python
# users.py
def users_home():
    ...

# main.py
def home():
    ...
```

Or explicitly provide a route name:

```python
@router.get("/", name="users_home")
def home():
    ...
```

This is especially important when using:

```jinja2
{{ url_for("users_home") }}
```

because `url_for()` looks for the **route name**, not the URL path.

---

## 🧠 Key Takeaways

* `APIRouter` organizes related routes into separate modules.
* `router.get()` / `router.post()` define routes inside a router.
* `app.include_router()` adds those routes to the main FastAPI application.
* `prefix` adds a common URL prefix to all routes in that router.
* `tags` groups routes in Swagger documentation.
* FastAPI uses the **endpoint function name as the route name by default**.
* Keep route names unique or explicitly provide `name=` when necessary.
