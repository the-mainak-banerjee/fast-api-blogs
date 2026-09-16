# FastAPI — Lesson 3: Path Parameters, Validation & Error Handling

**Lesson:** 3
**Video:** https://www.youtube.com/watch?v=WRjXIA5pMtk&list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI&index=5&t=1s

## Learnings

### 1. Path Parameters

A **path parameter** is a variable part of the URL.

```python
@app.get("/api/posts/{post_id}")
def get_post(post_id: int):
    for post in posts:
        if post.get("id") == post_id:
            return post

    return {"error": "Post not found"}
```

Here, `post_id` is a path parameter.

FastAPI uses the **type hint (`int`) to validate the value**.

For example:

```text
/api/posts/1       → valid
/api/posts/abc     → validation error
```

---

### 2. `HTTPException`

Instead of manually returning an error, FastAPI provides `HTTPException`.

```python
from fastapi import HTTPException, status

raise HTTPException(
    status_code=status.HTTP_404_NOT_FOUND,
    detail="Post not found"
)
```

`status` provides readable names for HTTP status codes, making the code clearer than using `404` directly.

---

### 3. `url_for()` with Dynamic Routes

`url_for()` can also generate URLs containing path parameters.

```html
{{ url_for("post_page", post_id=post.id) }}
```

Here, `post_id` is passed as a keyword argument to generate the dynamic URL.

---

## 4. Handling HTTP Exceptions

FastAPI uses **Starlette** underneath, and some HTTP exceptions—such as a **404 for a page that doesn't exist**—are handled by Starlette.

We can create a global handler for these exceptions:

```python
from starlette.exceptions import HTTPException as StarletteHTTPException
from fastapi.responses import JSONResponse


@app.exception_handler(StarletteHTTPException)
def general_http_exception_handler(
    request: Request,
    exception: StarletteHTTPException
):
    message = (
        exception.detail
        if exception.detail
        else "An error occurred. Please check your request and try again"
    )

    if request.url.path.startswith("/api"):
        return JSONResponse(
            status_code=exception.status_code,
            content={"details": message}
        )

    return templates.TemplateResponse(
        request,
        "error.html",
        {
            "status_code": exception.status_code,
            "title": exception.status_code,
            "message": message,
        },
        status_code=exception.status_code,
    )
```

### What does this handler do?

1. **Gets the error message**

   * Uses `exception.detail` if available.
   * Otherwise, uses a default message.

2. **Checks the requested URL**

```python
request.url.path.startswith("/api")
```

If the request is for an API endpoint, return a JSON error:

```json
{
    "details": "Post not found"
}
```

3. **For normal web pages**

Instead of returning JSON, it renders our `error.html` template.

The HTTP status code is also preserved using:

```python
status_code=exception.status_code
```

So a 404 error still returns an actual **404 HTTP response**, even though we show a custom HTML error page.

---

## 5. Handling Validation Errors

FastAPI raises `RequestValidationError` when request data doesn't match the expected type or validation rules.

```python
from fastapi.exceptions import RequestValidationError


@app.exception_handler(RequestValidationError)
def validation_exception_handler(
    request: Request,
    exception: RequestValidationError
):
    if request.url.path.startswith("/api"):
        return JSONResponse(
            status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
            content={"detail": exception.errors()},
        )

    return templates.TemplateResponse(
        request,
        "error.html",
        {
            "status_code": status.HTTP_422_UNPROCESSABLE_CONTENT,
            "title": status.HTTP_422_UNPROCESSABLE_CONTENT,
            "message": "Invalid request. Please check your input and try again.",
        },
        status_code=status.HTTP_422_UNPROCESSABLE_CONTENT,
    )
```

### Difference from `HTTPException`

`HTTPException` contains:

* A status code
* A `detail` message

Example:

```python
raise HTTPException(
    status_code=404,
    detail="Post not found"
)
```

`RequestValidationError` is different because it contains **validation error details**:

```python
exception.errors()
```

This returns a list containing information about what went wrong, such as the location, input value, and validation error.

Validation errors use **422 Unprocessable Content**.

---



## 🧠 Key Takeaways

* **Path parameters** are dynamic values in the URL.
* FastAPI validates path parameters using Python type hints.
* `HTTPException` is used to return HTTP errors.
* `status.HTTP_404_NOT_FOUND` is more readable than `404`.
* `url_for()` can generate dynamic URLs by passing path parameters.
* **Starlette** handles some HTTP exceptions used by FastAPI.
* `@app.exception_handler()` lets us customize how exceptions are handled.
* API errors can return **JSON**, while frontend errors can render an **HTML error page**.
* `RequestValidationError` handles invalid request data.
* Validation errors return **422 Unprocessable Content** and provide detailed information through `exception.errors()`.
