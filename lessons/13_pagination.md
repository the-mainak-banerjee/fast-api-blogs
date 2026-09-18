# Lesson 13: Pagination

**Video:** https://www.youtube.com/watch?v=f1zggIOxmJg&list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI&index=13

---

## 1. Define the Response Shape

Before implementing pagination, first decide what the API response should look like.

```python
class PaginatedPostsResponse(BaseModel):
    posts: list[PostResponse]
    total: int
    skip: int
    limit: int
    has_more: bool
```

Example response:

```json
{
  "posts": [],
  "total": 100,
  "skip": 20,
  "limit": 10,
  "has_more": true
}
```

The response contains:

* `posts` → Posts returned in this request
* `total` → Total number of posts
* `skip` → Number of posts skipped
* `limit` → Maximum number of posts requested
* `has_more` → Whether more posts are available

---

## 2. Why `skip` and `limit`?

Databases naturally support:

```text
OFFSET → skip records
LIMIT  → return this many records
```

For example:

```text
skip = 20
limit = 10
```

means:

```text
Skip first 20 posts
Return next 10 posts
```

With page-based pagination:

```text
page = 3
per_page = 10
```

the backend still needs to calculate:

```text
skip = (page - 1) × per_page
     = 20
```

So `skip` and `limit` map directly to the database query.

---

## 3. Pagination Flow

```text
Request
   ↓
Count total posts
   ↓
Query posts
   ↓
OFFSET(skip)
   ↓
LIMIT(limit)
   ↓
Calculate has_more
   ↓
Convert SQLAlchemy objects → Pydantic models
   ↓
Return posts + pagination metadata
```

---

## 4. Count Total Posts

```python
count_result = await db.execute(
    select(func.count()).select_from(models.Post)
)

total = count_result.scalar() or 0
```

This asks the database:

> "How many posts exist in total?"

If the database contains 100 posts:

```text
total = 100
```

We only count the records here; we don't retrieve all 100 posts.

---

## 5. Get the Requested Posts

```python
result = await db.execute(
    select(models.Post)
    .options(selectinload(models.Post.author))
    .order_by(models.Post.date_posted.desc())
    .offset(skip)
    .limit(limit)
)
```

For:

```text
skip = 20
limit = 10
```

the database returns posts:

```text
1 ... 20 | 21 ... 30 | 31 ...
          ↑
       Returned
```

### `offset(skip)`

Tells the database how many records to skip.

### `limit(limit)`

Tells the database the maximum number of records to return.

---

## 6. `has_more`

```python
has_more = skip + len(posts) < total
```

Example:

```text
total = 100
skip = 20
len(posts) = 10
```

Therefore:

```text
20 + 10 < 100
30 < 100
```

So:

```text
has_more = True
```

At the end:

```text
total = 100
skip = 90
len(posts) = 10
```

```text
90 + 10 < 100
100 < 100
```

So:

```text
has_more = False
```

### Why `has_more`?

It makes the frontend easier to build.

```text
has_more = true
    ↓
Load more / continue infinite scroll

has_more = false
    ↓
Stop loading
```

---

## 7. Why `model_validate()`?

```python
posts=[
    PostResponse.model_validate(post)
    for post in posts
]
```

The database gives us **SQLAlchemy objects**:

```text
SQLAlchemy Post
      ↓
id
title
content
date_posted
author
```

But our API response is defined using:

```python
PostResponse
```

So we convert:

```text
SQLAlchemy object
       ↓
Pydantic model
       ↓
JSON response
```

`model_validate()` performs this conversion.

---

## 8. Why `from_attributes=True`?

Our response model contains:

```python
class PostResponse(PostBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    user_id: int
    date_posted: datetime
    author: UserResponse
```

`from_attributes=True` tells Pydantic:

> "Read values from the object's attributes."

SQLAlchemy objects are accessed like:

```python
post.title
post.content
post.author
```

not like a dictionary:

```python
post["title"]
```

Therefore:

```python
PostResponse.model_validate(post)
```

can read the SQLAlchemy object's attributes and create the Pydantic response model.

---

## 9. Why Not Simply Return `posts`?

We could technically return the SQLAlchemy objects and let FastAPI serialize them in many cases.

But explicitly converting them:

```python
PostResponse.model_validate(post)
```

makes the boundary clear:

```text
Database Model
      ↓
Pydantic Response Schema
      ↓
API Response
```

The database model may contain fields that we don't want to expose.

For example:

```text
Database
├── id
├── title
├── content
├── user_id
└── internal fields
```

While our API might expose only:

```text
Response
├── id
├── title
├── content
├── user_id
└── author
```

The Pydantic schema acts as the **API contract**.

---

## 10. Final Response

```python
return PaginatedPostsResponse(
    posts=[
        PostResponse.model_validate(post)
        for post in posts
    ],
    total=total,
    skip=skip,
    limit=limit,
    has_more=has_more,
)
```

The API returns both:

### Data

```text
posts
```

### Metadata

```text
total
skip
limit
has_more
```

---

## 11. Complete Mental Model

```text
                  DATABASE
                     │
                     ▼
              Count all posts
                     │
                 total = 100
                     │
                     ▼
              Query requested
                  records
                     │
              skip = 20
              limit = 10
                     │
                     ▼
               Get 10 posts
                     │
                     ▼
             Load post.author
                     │
                     ▼
            Calculate has_more
                     │
                     ▼
         SQLAlchemy Post objects
                     │
                     ▼
       PostResponse.model_validate()
                     │
                     ▼
            Pydantic models
                     │
                     ▼
              JSON response
```

## Key Takeaway

> **Pagination controls how much data we retrieve, while Pydantic controls what data we expose.**

`model_validate()` is not specifically required because of pagination. It is used to convert the SQLAlchemy database objects into the Pydantic response models defined by our API.
