# FastAPI — `HTTPException` vs `RequestValidationError`

## 1. `HTTPException`

`HTTPException` is an exception **we intentionally raise in our application** when we want to tell the client that something went wrong with their request.

```python
from fastapi import HTTPException, status

raise HTTPException(
    status_code=status.HTTP_404_NOT_FOUND,
    detail="Post not found"
)
```

### Example

```text
GET /api/posts/999
        ↓
Post doesn't exist
        ↓
We raise HTTPException
        ↓
404 Not Found
```

Common cases:

* Resource not found → `404`
* Invalid request/business condition → `400`
* Unauthorized → `401`
* Forbidden → `403`

---

## 2. `RequestValidationError`

`RequestValidationError` is raised by **FastAPI automatically** when the incoming request doesn't match the validation rules defined by our endpoint/Pydantic schema.

For example:

```python
@app.get("/api/posts/{post_id}")
def get_post(post_id: int):
    ...
```

If the client sends:

```text
/api/posts/abc
```

FastAPI expects an `int`, but receives `"abc"`.

FastAPI's validation fails:

```text
"abc"
 ↓
Expected int
 ↓
RequestValidationError
 ↓
422 Unprocessable Content
```

Another example with Pydantic:

```python
class PostCreate(BaseModel):
    title: str = Field(min_length=1)
```

If the client sends an invalid value:

```json
{
    "title": ""
}
```

FastAPI/Pydantic raises a validation error.

---

## 3. Main Difference

|                  | `HTTPException`                                | `RequestValidationError`                  |
| ---------------- | ---------------------------------------------- | ----------------------------------------- |
| Who triggers it? | **Developer/application**                      | **FastAPI automatically**                 |
| When?            | Application detects a problem                  | Request doesn't match validation rules    |
| Purpose          | Tell client about an application/request error | Tell client their input failed validation |
| Typical status   | `400`, `401`, `403`, `404`, etc.               | `422`                                     |
| Example          | Post doesn't exist                             | `post_id="abc"` when `int` is expected    |

---

## 4. Easy Way to Remember

### `HTTPException`

> **"I received your request, but I can't fulfill it."**

Example:

```text
You asked for Post 999.
Post 999 doesn't exist.
→ 404
```

### `RequestValidationError`

> **"Your request doesn't have the data in the format I expected."**

Example:

```text
You sent "abc".
I expected an integer.
→ 422
```

---

## 5. Custom Exception Handlers

We can create custom handlers for both:

```python
from starlette.exceptions import HTTPException as StarletteHTTPException
from fastapi.exceptions import RequestValidationError
```

### HTTP Exception Handler

Used to customize how HTTP errors such as `404` or `400` are returned.

```python
@app.exception_handler(StarletteHTTPException)
def http_exception_handler(request, exception):
    ...
```

### Validation Exception Handler

Used to customize how FastAPI validation errors are returned.

```python
@app.exception_handler(RequestValidationError)
def validation_exception_handler(request, exception):
    ...
```

---

## 🧠 Final Mental Model

```text
                  Incoming Request
                         ↓
                  FastAPI Validation
                         ↓
             ┌───────────┴───────────┐
             ↓                       ↓
          Invalid                  Valid
             ↓                       ↓
RequestValidationError          Your route runs
             ↓                       ↓
            422              Application logic
                                     ↓
                              Something wrong?
                                     ↓
                              HTTPException
                                     ↓
                              400 / 404 / etc.
```

> **`RequestValidationError` = FastAPI rejects the input before your route logic can properly process it.**

> **`HTTPException` = your application deliberately rejects the request during/after processing.**
