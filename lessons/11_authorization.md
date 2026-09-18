# Lesson 11: Authorization

**Video:** https://www.youtube.com/watch?v=MY0TFMMm9B0&list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI&index=11&t=1s

---

## 1. Authentication vs Authorization

**Authentication** asks:

> Who are you?

Example:

```text
Email + Password
       ↓
    Login
       ↓
   JWT issued
```

**Authorization** asks:

> Are you allowed to perform this action?

Example:

```text
Authenticated user
       ↓
Try to delete a post
       ↓
Does this user own the post?
       ↓
Yes → Allow
No  → Reject
```

---

# 2. Authorization Process

Our `get_current_user()` function is a **dependency of other dependencies/endpoints**.

Its job is:

```text
Request
  ↓
Extract JWT
  ↓
Verify JWT
  ↓
Extract user ID
  ↓
Find user in database
  ↓
Return User object
```

Once we have the current user, other dependencies can use that user to perform authorization checks.

---

# 3. `get_current_user()`

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)]
):
```

FastAPI automatically provides:

### `token`

```python
Depends(oauth2_scheme)
```

Extracts the token from:

```http
Authorization: Bearer <token>
```

### `db`

```python
Depends(get_db)
```

Provides the database session for this request.

So the dependency receives everything it needs:

```text
JWT + Database Session
        ↓
get_current_user()
```

---

# 4. Verify the Token

```python
user_id = verify_access_token(token)
```

The JWT is verified.

If it is:

* expired
* incorrectly signed
* malformed
* missing required claims

then:

```python
if user_id is None:
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Invalid or expired token",
        headers={"WWW-Authenticate": "Bearer"},
    )
```

The request is rejected.

---

# 5. Convert User ID

The JWT contains:

```json
{
    "sub": "123"
}
```

`sub` is a string, while our database ID is an integer.

So:

```python
user_id_int = int(user_id)
```

converts:

```text
"123" → 123
```

If the value cannot be converted, the token is treated as invalid.

---

# 6. Find the User

```python
result = await db.execute(
    select(models.User).where(
        models.User.id == user_id_int
    )
)

user = result.scalars().first()
```

This asks the database:

> "Give me the user whose ID is 123."

If the user doesn't exist:

```python
if not user:
    raise HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="User not found",
        headers={"WWW-Authenticate": "Bearer"},
    )
```

Otherwise:

```python
return user
```

The dependency now returns the actual SQLAlchemy `User` object.

---

# 7. Creating a Reusable `CurrentUser`

Instead of writing this everywhere:

```python
current_user: Annotated[
    models.User,
    Depends(get_current_user)
]
```

we create an alias:

```python
CurrentUser = Annotated[
    models.User,
    Depends(get_current_user)
]
```

Now an endpoint can simply use:

```python
async def some_endpoint(
    current_user: CurrentUser
):
    ...
```

FastAPI understands:

```text
CurrentUser
    ↓
Depends(get_current_user)
    ↓
Run authentication
    ↓
Get User object
    ↓
Pass User object to endpoint
```

This is why it is called a **dependency**.

---

# 8. Dependency of a Dependency

This is an important FastAPI concept.

Imagine:

```text
Endpoint
   ↓
Authorization dependency
   ↓
Current user dependency
   ↓
OAuth2 dependency
   ↓
JWT
```

For example:

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)]
):
```

`get_current_user()` itself depends on:

```text
oauth2_scheme
get_db
```

And an endpoint can depend on:

```text
get_current_user
```

So dependencies can form a **dependency tree**.

---

# 9. Authentication vs Authorization in This System

Our current `get_current_user()` mainly establishes:

```text
Who is making this request?
        ↓
Current User
```

That's authentication.

Authorization comes afterward.

For example:

```text
Request
   ↓
JWT verification
   ↓
Identify User
   ↓
Is this user allowed to delete this post?
   ↓
Yes → Continue
No  → Reject
```

So you can think of it as:

```text
Authentication
     ↓
"Who are you?"
     ↓
Authorization
     ↓
"What can you do?"
```

---

# 10. 401 vs 403

These two status codes are extremely important.

## `401 Unauthorized`

Means:

> The request does not have valid authentication credentials.

Examples:

```text
No token
Expired token
Invalid token
Malformed token
Invalid authentication credentials
```

Example:

```text
GET /api/users/me

Authorization: Bearer invalid-token
                    ↓
                   401
```

Simple mental model:

> **"I don't know who you are."**

---

## `403 Forbidden`

Means:

> The server knows who you are, but you are not allowed to perform this action.

Example:

```text
User A
  ↓
Attempts to delete User B's post
  ↓
Authenticated?
Yes
  ↓
Allowed to delete?
No
  ↓
403
```

Simple mental model:

> **"I know who you are, but you can't do this."**

---

# 11. 401 vs 403 — Simple Example

Imagine entering an office.

### 401

You arrive without an ID card.

```text
Security: "Who are you?"
You: "..."
Security: "I can't verify you."
             ↓
            401
```

### 403

You show your employee ID.

```text
Security: "Who are you?"
You: "I'm an employee."
Security: "Okay, but this room is only for admins."
             ↓
            403
```

---

# 12. Authorization Example

Suppose we have:

```python
async def delete_post(
    post_id: int,
    current_user: CurrentUser,
):
```

The system can then check:

```text
JWT
 ↓
Current User = User 10
 ↓
Post 50
 ↓
Post owner = User 20
 ↓
User 10 != User 20
 ↓
403 Forbidden
```

If:

```text
User 10 == Post owner User 10
```

then:

```text
Allow deletion
```

---

# 13. The Mental Model

The complete security flow is:

```text
                 REQUEST
                    ↓
              Has credentials?
                    ↓
             Verify JWT
              ↙         ↘
          Invalid       Valid
             ↓            ↓
            401       Identify user
                           ↓
                    Authorization check
                      ↙          ↘
                   Allowed     Not allowed
                      ↓            ↓
                   Continue       403
```

### Remember:

```text
401 → Authentication problem
      "Who are you?"

403 → Authorization problem
      "I know who you are, but you can't do this."
```
