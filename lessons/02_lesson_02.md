# FastAPI — Lesson 2: HTML Templates & Static Files

**Lesson:** 2
**Video:** https://youtu.be/G4NIB9Rx9Qs?si=AGolVPdMSd3iLTu3

## Learnings

### 1. Jinja2 Templates

Jinja2 allows us to generate dynamic HTML using Python data.

```python
from fastapi import Request
from fastapi.templating import Jinja2Templates

templates = Jinja2Templates(directory="src/fastapi_blog/templates")
```

`directory` specifies where the HTML templates are located.

### 2. Returning a Template

```python
@app.get("/", include_in_schema=False)
@app.get("/posts", include_in_schema=False)
def home(request: Request):
    return templates.TemplateResponse(request, "home.html")
```

* `request: Request` → the incoming HTTP request. `TemplateResponse` needs it to build the response.
* `"home.html"` → the template to render.
* `TemplateResponse()` → renders the template and returns it as an HTTP response.

### 3. Passing Data to Templates

Pass data using a **context dictionary**:

```python
return templates.TemplateResponse(
    request,
    "home.html",
    {"posts": posts}
)
```

The `posts` key becomes available inside the template.

### 4. Using Data in HTML

```html
{% for post in posts %}
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
{% endfor %}
```

* `{% ... %}` → Jinja2 statements/logic
* `{{ ... }}` → displays a value
* Dot notation (`post.title`) can be used to access dictionary values.

### 5. Template Inheritance

A base template can contain common HTML such as the header, navigation, and footer.

Other templates can **extend** the base template and provide their own content.

This avoids repeating common HTML.

### 6. Static Files & `app.mount()`

FastAPI can serve static files such as CSS, JavaScript, and images.

```python
from fastapi.staticfiles import StaticFiles

app.mount(
    "/static",
    StaticFiles(directory="src/fastapi_blog/static"),
    name="static"
)
```

Parameters:

* `"/static"` → URL prefix.
* `StaticFiles(...)` → application that serves the files.
* `directory=...` → folder containing the static files.
* `name="static"` → internal route name used with `url_for()`.

`app.mount()` attaches another ASGI application to a specific path.

ASGI — Asynchronous Server Gateway Interface: A Python standard that defines how web servers communicate with asynchronous Python web applications like FastAPI.

Here, `StaticFiles` is mounted at `/static`:

```text
/static → src/fastapi_blog/static/
```

So:

```text
src/fastapi_blog/static/style.css
```

can be accessed through:

```text
/static/style.css
```

### 7. `url_for()` & Route Names

A **route name is an internal name/identifier for a route. It is not a URL.**

```python
@app.get("/", name="home")
def home():
    ...
```

Here:

```text
URL/path:    /
Route name:  home
Function:    home()
```

In Jinja2:

```html
{{ url_for("home") }}
```

FastAPI finds the route named `home` and generates its URL:

```text
/
```

This is useful because if the URL changes:

```python
@app.get("/home", name="home")
```

we can still use:

```html
{{ url_for("home") }}
```

and it will generate:

```text
/home
```

### 8. Explicitly Naming Routes

When one function has multiple routes, we can explicitly give each route a name:

```python
@app.get("/", include_in_schema=False, name="home")
@app.get("/posts", include_in_schema=False, name="posts")
def home(request: Request):
    ...
```

Now:

```html
{{ url_for("home") }}
```

generates `/`

and:

```html
{{ url_for("posts") }}
```

generates `/posts`.

## 🧠 Key Takeaways

* **Jinja2** → dynamic HTML templates.
* **Context dictionary** → passes Python data to templates.
* `{{ }}` → displays data.
* `{% %}` → template logic.
* **Template inheritance** → reuse common layouts.
* `app.mount()` → attaches another application, such as `StaticFiles`, to a URL path.
* **StaticFiles** → serves CSS, JS, images, etc.
* **Route name** → internal identifier for a route, not another URL.
* `url_for()` → generates a URL from a route name.
