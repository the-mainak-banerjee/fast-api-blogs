# FastAPI — Lesson 4: Pydantic Schemas — Request & Response Validation

**Lesson:** 4
**Video:** https://www.youtube.com/watch?v=9GHxnttXxrA&list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI&index=4

## Learnings

### 1. What is Pydantic?

**Pydantic** is used to define and validate the data that our API **receives from the client and returns to the client**.

* **Pydantic schema** → defines/validates API input and output.
* **Database schema/model** → defines what data is stored in the database.

Pydantic also improves the automatically generated **Swagger/OpenAPI documentation**.

### 2. Pydantic Imports

```python
from pydantic import BaseModel, Field, ConfigDict
```

* `BaseModel` → base class for creating Pydantic models.
* `Field` → adds validation rules and metadata to fields.
* `ConfigDict` → configures how a Pydantic model behaves.

### 3. Creating a Pydantic Schema

```python
class PostBase(BaseModel):
    title: str = Field(min_length=1, max_length=100)
    content: str = Field(min_length=1)
    author: str = Field(min_length=1, max_length=50)
```

`Field()` allows us to add constraints such as minimum and maximum length.

For example:

```text
title → 1–100 characters
content → at least 1 character
author → 1–50 characters
```

### 4. Response Schema

```python
class PostResponse(PostBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    date_posted: str
```

`PostResponse` inherits the fields from `PostBase` and adds fields that are returned by the API.

### 5. `from_attributes=True`

```python
model_config = ConfigDict(from_attributes=True)
```

This allows Pydantic to create the response model from an **object's attributes**, not only from a dictionary.

This is particularly useful when our data comes from a **database ORM object**.

For example, an ORM object might have:

```python
post.title
post.content
post.author
```

With `from_attributes=True`, Pydantic can read these attributes and convert the object into the `PostResponse` schema.

Without it, Pydantic generally expects dictionary-like input.

## 🧠 Key Takeaways

* **Pydantic** → validates API request and response data.
* **Database model** → defines data stored in the database.
* `BaseModel` → base class for Pydantic schemas.
* `Field()` → adds validation constraints.
* Pydantic schemas also improve Swagger/OpenAPI documentation.
* `from_attributes=True` → allows Pydantic to read data from object attributes, which is useful with ORM/database objects.
