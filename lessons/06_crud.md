# FastAPI — Lesson 6: Complete CRUD Operations

**Lesson:** 6
**Video:** https://youtu.be/VyoGAoxQhxM?si=nMgQw4SRLYVGrIaL

## Learnings

### 1. `PostUpdate` Pydantic Model

For updating a post partially with `PATCH`, we don't necessarily want the client to send every field.

```python
class PostUpdate(BaseModel):
    title: str | None = Field(
        default=None,
        min_length=1,
        max_length=100
    )
    content: str | None = Field(
        default=None,
        min_length=1
    )
```

* `str | None` → the field can contain a string or `None`.
* `default=None` → the field is optional when making an update.
* `Field()` → still validates the value when the field is provided.

This allows requests such as:

```json
{
    "title": "Updated title"
}
```

without requiring `content`.

---

## 2. Full Update — `PUT`

```python
@app.put("/api/posts/{post_id}", response_model=PostResponse)
def update_post_full(
    post_id: int,
    post_data: PostCreate,
    db: Annotated[Session, Depends(get_db)]
):
```

`PUT` is generally used when we want to **replace/update the complete resource**.

### Find the Post

```python
result = db.execute(
    select(models.Post).where(models.Post.id == post_id)
)

post = result.scalars().first()
```

Finds the post whose `id` matches the `post_id` from the URL.

If it doesn't exist:

```python
if not post:
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail="Post not found"
    )
```

---

### Check the New User

```python
if post_data.user_id != post.user_id:
```

Checks whether the user assigned to the post is being changed.

If it is changing, we verify that the new user exists:

```python
result = db.execute(
    select(models.User).where(
        models.User.id == post_data.user_id
    )
)

user = result.scalars().first()
```

If the user doesn't exist, return `404`.

---

### Update the Post

```python
post.title = post_data.title
post.content = post_data.content
post.user_id = post_data.user_id
```

Updates the SQLAlchemy object with the new values.

Then:

```python
db.commit()
```

saves the changes to the database.

```python
db.refresh(post)
```

reloads the updated object from the database.

```python
return post
```

returns the updated post, which FastAPI validates using `PostResponse`.

---

## 3. Partial Update — `PATCH`

For `PATCH`, we use `PostUpdate` because only the fields that need changing should be sent.

```python
update_data = post_data.model_dump(exclude_unset=True)
```

`model_dump()` converts the Pydantic model into a dictionary.

`exclude_unset=True` means:

> Only include fields that the client actually sent.

For example:

```json
{
    "title": "New title"
}
```

becomes:

```python
{
    "title": "New title"
}
```

instead of:

```python
{
    "title": "New title",
    "content": None
}
```

### Dynamically Update Fields

```python
for field, value in update_data.items():
    setattr(post, field, value)
```

* `field` → field name, e.g. `"title"`
* `value` → new value, e.g. `"New title"`
* `setattr()` → dynamically sets an object's attribute.

This:

```python
setattr(post, "title", "New title")
```

is equivalent to:

```python
post.title = "New title"
```

So the loop allows us to update **any fields provided by the client without manually writing an assignment for each field**.

---

# 4. HTTP Status Codes

| Status Code                   | Meaning                                   | Example                                               |
| ----------------------------- | ----------------------------------------- | ----------------------------------------------------- |
| **200 OK**                    | Request successful                        | Successful GET, PUT or PATCH                          |
| **201 Created**               | Resource successfully created             | Successful POST that creates a new user               |
| **204 No Content**            | Request successful, but no response body  | Successful DELETE where nothing needs to be returned  |
| **400 Bad Request**           | Request is invalid or cannot be processed | Username already exists                               |
| **404 Not Found**             | Requested resource doesn't exist          | Post with ID `10` doesn't exist                       |
| **422 Unprocessable Content** | Request data failed validation            | `post_id` expects an integer but `"abc"` was provided |

---

# 5. Cascade Delete

Cascade delete means that deleting a parent object can automatically delete its related child objects.

```python
posts: Mapped[list[Post]] = relationship(
    back_populates="author",
    cascade="all, delete-orphan"
)
```

Here:

```text
User
 ├── Post 1
 ├── Post 2
 └── Post 3
```

If the `User` is deleted, the related posts are also deleted.

### `delete-orphan`

`delete-orphan` means that if a `Post` is removed from the user's `posts` relationship and is no longer associated with another parent, SQLAlchemy can delete that post from the database.

So:

```text
Delete User
     ↓
Delete related Posts
```

This helps prevent **orphaned records**.
> An orphaned record is a database record that still exists but no longer has the parent/relationship it is supposed to belong to.

---

## 🧠 Key Takeaways

* `PUT` → full update/replacement.
* `PATCH` → partial update.
* `PostUpdate` makes fields optional for partial updates.
* `model_dump(exclude_unset=True)` → gets only fields actually provided.
* `setattr()` → dynamically updates object attributes.
* `db.commit()` → saves changes.
* `db.refresh()` → reloads the updated database object.
* Cascade delete → automatically handles related records when a parent is deleted.
