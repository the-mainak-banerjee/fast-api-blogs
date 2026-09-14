# FastAPI — Lesson 1

**Lesson:** 1
**Video:** https://youtu.be/7AMjmCTumuo?si=izEGhEMv95VWoCVG

## Learnings

### 1. Setup with `uv`

```bash
uv init project_name
cd project_name
uv add "fastapi[standard]"
```

`fastapi[standard]` installs FastAPI along with commonly needed standard dependencies, including the FastAPI CLI and Uvicorn.

### 2. Create a FastAPI App

```python
from fastapi import FastAPI

app = FastAPI()
```

`FastAPI()` creates the application instance.

### 3. Multiple Routes for One Function

A single function can handle multiple routes using multiple decorators:

```python
@app.get("/")
@app.get("/posts")
def home():
    return "Hello World"
```

Both `/` and `/posts` call `home()`.

### 4. Return HTML

```python
from fastapi.responses import HTMLResponse

@app.get("/", response_class=HTMLResponse)
def home():
    return "<h1>Hello World</h1>"
```

`HTMLResponse` tells FastAPI that the response is HTML.

### 5. Hide a Route from Swagger Docs

```python
@app.get(
    "/",
    response_class=HTMLResponse,
    include_in_schema=False
)
```

`include_in_schema=False` hides the route from the generated OpenAPI/Swagger documentation.

**Important:** It does **not** make the route private or inaccessible.
