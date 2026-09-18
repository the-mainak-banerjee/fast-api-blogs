# Authentication — Registration and Login with JWT

**Lesson:** Authentication — Registration and Login with JWT
**Video:** https://www.youtube.com/watch?v=Go4wYJJhR3k&t=1820s

---

# 1. Authentication vs Authorization

Before implementing authentication, understand the difference.

### Authentication

> **"Who are you?"**

Example:

```text
User enters email + password
        ↓
Backend verifies them
        ↓
User is authenticated
```

### Authorization

> **"What are you allowed to do?"**

Example:

```text
Authenticated user
        ↓
Can read posts?        → Yes
Can delete this post?  → Maybe
Can access admin page? → Maybe not
```

---

# 2. Packages

We installed:

```bash
uv add "pwdlib[argon2]" pyjwt pydantic-settings
```

## `pwdlib[argon2]`

Used for **secure password hashing and verification**.

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()
```

We use hashing because passwords should **not be stored in their original form**.

`pwdlib` provides a simple interface around password-hashing algorithms such as Argon2 and bcrypt.

### Why Argon2?

Argon2 is designed specifically for password hashing and is memory-hard, making large-scale password cracking more expensive.

For this project:

```text
Password
   ↓
Argon2
   ↓
Password hash
   ↓
Database
```

We never store:

```text
mypassword123
```

We store a hash instead.

### Alternatives

* `argon2-cffi`
* `bcrypt`
* `scrypt`
* `passlib` — historically popular, but its maintenance status has been problematic; `pwdlib` is a more modern choice.

---

# 3. Why Hash Passwords Instead of Encrypting Them?

### Encryption

Encryption is designed to be **reversible**.

```text
Original password
       ↓
   Encryption
       ↓
Encrypted value
       ↓
   Decryption
       ↓
Original password
```

### Hashing

Hashing is designed to be **one-way**.

```text
Original password
       ↓
      Hash
       ↓
Hash value
```

You don't decrypt the hash.

During login:

```text
User enters password
       ↓
Hash/verify against stored hash
       ↓
Match?
  ↓       ↓
 Yes      No
  ↓       ↓
Login    Reject
```

So:

> **Passwords are hashed because the server only needs to verify them, not recover the original password.**

---

# 4. `pyjwt`

```bash
pyjwt
```

`PyJWT` is a Python library for creating and verifying **JSON Web Tokens (JWTs)**.

We use it to create the access token after successful login.

```text
Login successful
      ↓
Create JWT
      ↓
Send JWT to client
      ↓
Client sends JWT with future requests
```

### Alternatives

JWT itself is a standard, while PyJWT is a Python implementation.

Other Python options include:

* `python-jose`
* `authlib`

For this project, `PyJWT` is being used as the JWT implementation.

---

# 5. `pydantic-settings`

```bash
pydantic-settings
```

Used for managing **application configuration and environment variables**.

For example:

```text
SECRET_KEY
DATABASE_URL
API_KEY
TOKEN_EXPIRATION
```

Instead of hardcoding secrets inside Python:

```python
secret_key = "super-secret-key"
```

we keep them outside the source code:

```text
.env
```

and load them through `pydantic-settings`.

---

# 6. User Database Model

We add:

```python
password_hash: Mapped[str] = mapped_column(
    String(200),
    nullable=False
)
```

The database stores the **password hash**, not the original password.

Example:

```text
email: mainak@example.com
password_hash: $argon2id$v=19$...
```

The actual password is never stored.

---

# 7. User Creation Schema

Our Pydantic schema now accepts a password:

```python
class UserCreate(UserBase):
    password: str = Field(min_length=8)
```

This means:

> The client is allowed to send a password when creating an account.

For example:

```json
{
    "username": "mainak",
    "email": "mainak@example.com",
    "password": "mypassword123"
}
```

The password should then be:

```text
Request
   ↓
Pydantic validation
   ↓
Password
   ↓
Hash password
   ↓
Store password_hash
```

The plain password should **not** be stored in the database.

---

# 8. `UserPublic` vs `UserPrivate`

Previously we had:

```python
class UserResponse(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
    image_file: str | None
    image_path: str
```

Now we separate public and private information.

## Public User

```python
class UserPublic(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    username: str
    image_file: str | None
    image_path: str
```

This contains information that is safe to expose publicly.

## Private User

```python
class UserPrivate(UserPublic):
    email: EmailStr
```

`UserPrivate` inherits everything from `UserPublic` and additionally exposes the email.

This gives us control over what different endpoints return.

For example:

```text
Public profile
    ↓
UserPublic

Current user's profile
    ↓
UserPrivate
```

Notice that **neither schema contains `password_hash`**.

That is extremely important.

---

# 9. `config.py`

```python
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8"
    )

    secret_key: SecretStr
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30


settings = Settings()
```

This file defines the configuration required by our application.

---

# 10. What is `BaseSettings`?

```python
class Settings(BaseSettings):
```

`BaseSettings` is a Pydantic class designed specifically for **application configuration**.

Instead of manually doing:

```python
import os

secret_key = os.getenv("SECRET_KEY")
```

we define what our application expects:

```python
class Settings(BaseSettings):
    secret_key: SecretStr
    algorithm: str = "HS256"
```

Pydantic Settings then reads the values from the configured sources and validates/converts them to the declared types.

---

# 11. What is `model_config`?

```python
model_config = SettingsConfigDict(
    env_file=".env",
    env_file_encoding="utf-8"
)
```

`model_config` is where we configure **how a Pydantic model behaves**.

It is not application data.

For example:

```python
class User(BaseModel):

    model_config = ConfigDict(
        from_attributes=True
    )
```

configures how that Pydantic model behaves.

Similarly:

```python
class Settings(BaseSettings):

    model_config = SettingsConfigDict(
        env_file=".env"
    )
```

configures how the settings model reads configuration.

Think:

```text
Fields
  ↓
What data does this model contain?

model_config
  ↓
How should this model behave?
```

---

# 12. `.env` File

Example:

```env
SECRET_KEY=your-secret-key
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
```

The `.env` file contains configuration values.

They are initially read as text.

For example:

```text
"30"
```

But Pydantic sees:

```python
access_token_expire_minutes: int
```

and converts it to:

```python
30
```

So:

```text
.env
"30"
 ↓
Pydantic validation/conversion
 ↓
Python int
30
```

---

# 13. Environment Variable Names

Our field:

```python
secret_key: SecretStr
```

matches the environment variable:

```env
SECRET_KEY=...
```

Pydantic Settings is case-insensitive by default, so environment variable naming does not need to match Python field casing exactly.

---

# 14. Configuration Priority

There are multiple possible sources for settings.

Conceptually, higher-priority sources override lower-priority ones.

For our use case:

```text
Explicit values passed to Settings()
        ↓
Environment variables
        ↓
.env file
        ↓
Default values defined in Settings
```

For example:

```python
access_token_expire_minutes: int = 30
```

If nothing else provides a value:

```text
30
```

is used.

But if `.env` contains:

```env
ACCESS_TOKEN_EXPIRE_MINUTES=60
```

then:

```text
60
```

is used.

And if the system environment contains:

```env
ACCESS_TOKEN_EXPIRE_MINUTES=90
```

that environment variable takes precedence over the `.env` value.

---

# 15. `SecretStr`

```python
secret_key: SecretStr
```

`SecretStr` is a Pydantic type designed to prevent secrets from being accidentally exposed when values are represented or printed.

For example, instead of casually displaying the actual secret, Pydantic masks it.

When we genuinely need the value:

```python
settings.secret_key.get_secret_value()
```

we explicitly retrieve the underlying string.

---

# 16. `.env` Is Not Encrypted

Important:

```text
.env
```

does **not** encrypt secrets.

It is simply a convenient way of keeping configuration outside the source code.

Therefore:

```text
.env
     ↓
Plain text
```

You should generally add it to:

```text
.gitignore
```

and avoid committing production secrets to Git.

In production, secrets are commonly provided through the hosting platform's environment variables or a dedicated secret-management system.

---

# 17. Other Simple Ways to Handle `.env`

### `python-dotenv`

A common simple approach:

```python
from dotenv import load_dotenv
import os

load_dotenv()

secret_key = os.getenv("SECRET_KEY")
```

This directly loads `.env` values into environment variables.

### `os.environ`

You can also rely directly on system environment variables:

```python
import os

secret_key = os.environ["SECRET_KEY"]
```

No `.env` file is required.

### Pydantic Settings

Our approach:

```python
class Settings(BaseSettings):
    secret_key: SecretStr
```

This is useful when you want:

* Type validation
* Automatic conversion
* Defaults
* Centralized configuration
* `.env` support
* Environment variable support

For a FastAPI application, this provides a clean configuration layer.

---

# 18. Generate a Secret Key

We can generate a strong random secret:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

Example output:

```text
a8f1...long-random-value...
```

Put the generated value into `.env`:

```env
SECRET_KEY=...
```

The secret should be unpredictable and kept private.

---

# 19. `auth.py`

Our authentication utilities:

```python
from datetime import UTC, datetime, timedelta

import jwt

from fastapi.security import OAuth2PasswordBearer
from pwdlib import PasswordHash

from .config import settings
```

These provide:

* Password hashing
* JWT creation/verification
* Token expiration
* OAuth2 bearer-token extraction
* Application configuration

---

# 20. Password Hasher

```python
password_hash = PasswordHash.recommended()
```

`pwdlib` creates a password-hashing helper using recommended settings.

We then have:

```python
def hash_password(password: str) -> str:
    return password_hash.hash(password)
```

This converts:

```text
mypassword123
```

into something like:

```text
$argon2id$v=19$...
```

---

# 21. Verifying a Password

```python
def verify_password(
    plain_password: str,
    hashed_password: str
) -> bool:
    return password_hash.verify(
        plain_password,
        hashed_password
    )
```

We don't hash the password ourselves and compare strings.

The password-hashing library handles the verification process, including the salt and parameters stored in the hash.

```text
Entered password
       ↓
Password verifier
       ↓
Stored hash
       ↓
Match?
```

---

# 22. What is JWT?

JWT = **JSON Web Token**.

It is a compact token containing claims that can be used to represent information about an authenticated user.

A JWT commonly looks like:

```text
xxxxx.yyyyy.zzzzz
```

It has three parts:

```text
Header.Payload.Signature
```

### Header

Contains information such as the signing algorithm.

### Payload

Contains claims.

For example:

```json
{
    "sub": "123",
    "exp": 1780000000
}
```

Here:

```text
sub → subject/user identifier
exp → expiration time
```

### Signature

The signature allows the backend to verify that the token was created using the expected signing key and has not been modified.

---

# 23. JWT Is Signed, Not Encrypted

This is extremely important.

A normal JWT payload should **not be treated as secret data**.

The client can generally decode the payload.

The signature provides **integrity/authenticity**, not confidentiality.

Therefore don't put things like:

```text
password
credit card number
private secrets
```

inside the JWT.

---

# 24. Creating an Access Token

```python
def create_access_token(
    data: dict,
    expires_delta: timedelta | None = None
) -> str:
```

The function receives:

```text
data
```

which contains information we want in the token.

Example:

```python
{
    "sub": "123"
}
```

---

# 25. What is `timedelta`?

`timedelta` represents a **duration of time**.

For example:

```python
timedelta(minutes=30)
```

means:

> 30 minutes from now.

It is useful when calculating expiration.

```python
datetime.now(UTC) + timedelta(minutes=30)
```

means:

> Current UTC time + 30 minutes.

---

# 26. Token Expiration

```python
expire = datetime.now(UTC) + expires_delta
```

If the token expires in 30 minutes:

```text
10:00
  +
30 minutes
  ↓
10:30
```

The token contains that expiration time.

```python
to_encode.update({"exp": expire})
```

---

# 27. Encoding the JWT

```python
encoded_jwt = jwt.encode(
    to_encode,
    settings.secret_key.get_secret_value(),
    algorithm=settings.algorithm,
)
```

Conceptually:

```text
User information
      +
Expiration
      +
Secret key
      ↓
JWT
```

The secret key is used to sign the token.

---

# 28. Verifying a JWT

```python
def verify_access_token(token: str) -> str | None:
```

The backend receives a token and tries to decode/verify it:

```python
payload = jwt.decode(
    token,
    settings.secret_key.get_secret_value(),
    algorithms=[settings.algorithm],
    options={"require": ["exp", "sub"]},
)
```

It verifies things such as:

* Signature
* Algorithm
* Expiration
* Required claims

If verification fails:

```python
except jwt.InvalidTokenError:
    return None
```

Otherwise:

```python
return payload.get("sub")
```

returns the user ID stored in the `sub` claim.

---

# 29. Why `sub`?

`sub` means **subject**.

It is commonly used to identify the entity that the token represents.

In our application:

```python
data={"sub": str(user.id)}
```

means:

> "This token represents user 123."

So:

```text
JWT
 ↓
sub = "123"
 ↓
User ID = 123
```

---

# 30. OAuth2PasswordBearer

```python
oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="api/users/token"
)
```

This tells FastAPI:

> "This API uses a Bearer token for authentication, and the client gets that token from this token endpoint."

The `OAuth2PasswordBearer` dependency extracts the bearer token from the request's:

```http
Authorization: Bearer <token>
```

header.

It also integrates the security scheme into OpenAPI/Swagger.

---

# 31. Important: `tokenUrl`

`tokenUrl` describes **where the client obtains the token**.

It does not create that endpoint.

For example:

```python
OAuth2PasswordBearer(tokenUrl="api/users/token")
```

means the token endpoint is expected to be:

```text
/api/users/token
```

The actual login/token route must match this path.

---

# 32. Complete Login System — Language-Agnostic

Forget Python for a moment.

A login system fundamentally needs these steps:

### Step 1 — Registration

User provides:

```text
Email
Username
Password
```

### Step 2 — Validate Input

Check:

```text
Is email valid?
Is password long enough?
Is username already taken?
```

### Step 3 — Hash Password

```text
Plain password
       ↓
Password hashing algorithm
       ↓
Password hash
```

Store only the hash.

### Step 4 — Store User

Database:

```text
id
username
email
password_hash
```

### Step 5 — Login

User sends:

```text
Email
Password
```

### Step 6 — Find User

Search database using the email.

### Step 7 — Verify Password

Compare the entered password with the stored password hash.

### Step 8 — Create Access Token

If the password is correct:

```text
User ID
+
Expiration
+
Other claims
       ↓
Signed JWT
```

### Step 9 — Send Token

Backend sends:

```json
{
    "access_token": "...",
    "token_type": "bearer"
}
```

### Step 10 — Client Stores/Uses Token

For future protected requests:

```http
Authorization: Bearer <token>
```

### Step 11 — Backend Verifies Token

For every protected request:

```text
Receive token
    ↓
Verify signature
    ↓
Check expiration
    ↓
Extract user ID
    ↓
Find/validate user
    ↓
Allow request
```

---

# 33. Login Endpoint

```python
@router.post("/login", response_model=Token)
async def login_for_access_token(
    form_data: Annotated[
        OAuth2PasswordRequestForm,
        Depends()
    ],
    db: Annotated[
        AsyncSession,
        Depends(get_db)
    ]
):
```

There are two important concepts here:

```text
OAuth2PasswordRequestForm
        ↓
Gets login credentials

AsyncSession
        ↓
Talks to database
```

---

# 34. What is `OAuth2PasswordRequestForm`?

OAuth2's password flow expects login credentials as **form data** with these field names:

```text
username
password
```

It does not mean the user must actually have a username.

In our application we use:

```text
username = email
```

So the frontend sends:

```text
username=mainak@example.com
password=mypassword123
```

even though we treat `username` as the user's email.

This is because the OAuth2 password flow defines those field names.

FastAPI provides `OAuth2PasswordRequestForm` so we don't have to manually parse those form fields.

---

# 35. Why Not JSON?

Normally we might send:

```json
{
    "email": "mainak@example.com",
    "password": "mypassword123"
}
```

But the OAuth2 password flow expects:

```text
username=mainak@example.com
password=mypassword123
```

as form data.

That's why we use:

```python
OAuth2PasswordRequestForm
```

If we weren't following this OAuth2 form convention, we could simply create our own Pydantic login schema and accept JSON.

---

# 36. Find the User

```python
result = await db.execute(
    select(models.User).where(
        func.lower(models.User.email)
        == form_data.username.lower()
    )
)

user = result.scalars().first()
```

Conceptually:

```text
Email from login
       ↓
Convert to lowercase
       ↓
Search database
       ↓
Find user
```

---

# 37. Case-Insensitive Email Search

```python
func.lower(models.User.email)
```

converts the database email to lowercase for comparison.

```python
form_data.username.lower()
```

converts the incoming email to lowercase.

Therefore:

```text
MAINAK@EMAIL.COM
mainak@email.com
Mainak@Email.com
```

can all match:

```text
mainak@email.com
```

### Important

Email addresses technically have case-related rules, but most applications intentionally normalize email addresses for account lookup.

The important architectural point is:

> Decide on a consistent normalization policy and use it during registration and login.

For example:

```text
Registration
    ↓
lowercase email
    ↓
store normalized email

Login
    ↓
lowercase email
    ↓
lookup
```

This is more predictable than only making the query case-insensitive.

---

# 38. Verify Credentials

```python
if not user or not verify_password(
    form_data.password,
    user.password_hash
):
```

This checks two things:

```text
Does user exist?
       AND
Does password match?
```

If either fails:

```python
raise HTTPException(
    status_code=status.HTTP_401_UNAUTHORIZED,
    detail="Incorrect email or password",
)
```

We deliberately use the **same error message** for both cases.

Why?

We don't want to tell an attacker:

```text
"Email exists but password is wrong"
```

or:

```text
"Email doesn't exist"
```

That would help attackers discover valid accounts.

---

# 39. Create the Access Token

```python
access_token_expires = timedelta(
    minutes=settings.access_token_expire_minutes
)
```

This creates a duration.

For example:

```text
30 minutes
```

Then:

```python
access_token = create_access_token(
    data={"sub": str(user.id)},
    expires_delta=access_token_expires,
)
```

The token represents:

```text
User ID = 123
Expires = 30 minutes from now
```

---

# 40. Return the Token

```python
return Token(
    access_token=access_token,
    token_type="bearer"
)
```

The frontend receives the token.

For future protected requests:

```http
Authorization: Bearer eyJ...
```

---

# 41. The Complete Login Flow

```text
                  LOGIN
                    │
                    ▼
             Email + Password
                    │
                    ▼
             Validate input
                    │
                    ▼
             Find user in DB
                    │
             ┌──────┴──────┐
             │             │
           Found         Not Found
             │             │
             ▼             ▼
       Verify password    401
             │
       ┌─────┴─────┐
       │           │
     Match       Wrong
       │           │
       ▼           ▼
  Create JWT       401
       │
       ▼
 Send JWT to client
```

---

# 42. `/me` Endpoint

```python
@router.get(
    "/me",
    response_model=UserPrivate
)
async def get_current_user(...):
```

`/me` means:

> **"Give me the information about the currently authenticated user."**

It doesn't mean:

```text
GET /users/123
```

Instead, the identity comes from the authentication token.

---

# 43. Why Do We Need `/me`?

Imagine the frontend has just logged in.

It has:

```text
JWT
```

But it might need to know:

```text
Who am I?
Username?
Email?
Profile picture?
```

The frontend can call:

```http
GET /api/users/me
Authorization: Bearer <token>
```

The backend then identifies the user from the token and returns the current user's information.

---

# 44. `/me` Authentication Flow

```text
Frontend
   │
   │ GET /me
   │ Authorization: Bearer JWT
   ▼
Backend
   │
   ▼
Extract JWT
   │
   ▼
Verify JWT
   │
   ▼
Extract user ID
   │
   ▼
Find user in database
   │
   ▼
Return current user
```

---

# 45. Extracting the Token

```python
token: Annotated[
    str,
    Depends(oauth2_scheme)
]
```

`oauth2_scheme` reads:

```http
Authorization: Bearer <token>
```

and gives our endpoint the actual token string.

So we don't manually parse the header.

---

# 46. Verify the JWT

```python
user_id = verify_access_token(token)
```

This checks whether the token is valid.

If it isn't:

```python
if user_id is None:
```

we return:

```text
401 Unauthorized
```

---

# 47. Why Convert User ID to Integer?

JWT payload gives us:

```python
"sub": "123"
```

Notice:

```text
"123"
```

is a string.

But our database ID is:

```python
id: int
```

So:

```python
user_id_int = int(user_id)
```

converts:

```text
"123"
```

into:

```text
123
```

The `try/except` also protects against malformed token data.

---

# 48. Find the User in the Database

Even if the JWT is valid, we still query the database:

```python
result = await db.execute(
    select(models.User).where(
        models.User.id == user_id_int
    )
)
```

Why?

Because a valid JWT doesn't necessarily mean the user still exists.

For example:

```text
User logs in
    ↓
JWT created
    ↓
Admin deletes user
    ↓
JWT is still technically valid
```

If we only trusted the JWT, the deleted user could potentially continue accessing protected resources until the token expires.

The database lookup gives us the **current state of the user**.

---

# 49. Why Not Just Trust the JWT?

JWT tells us:

```text
"This token was validly signed and belongs to user 123."
```

It does **not automatically tell us**:

```text
"User 123 still exists."
"User 123 is active."
"User 123 is not banned."
"User 123 still has this role."
```

Those are application/database questions.

Therefore:

```text
JWT
 ↓
Who does this request claim to be?
 ↓
Database
 ↓
Does that user currently exist / have access?
```

---

# 50. Can the Frontend Verify the JWT?

The frontend can **decode** a JWT payload.

For example, it can read:

```json
{
    "sub": "123",
    "exp": 1780000000
}
```

This can be useful for UI purposes.

However:

> **The frontend must not be trusted to decide whether a user is authenticated or authorized.**

For HS256, the signing secret is shared between the token issuer and verifier. You must **never put that secret in browser JavaScript**, because users can inspect the frontend code.

Therefore:

```text
Frontend
   ↓
Can decode/read JWT payload
   ↓
Useful for UI

Backend
   ↓
Verifies JWT signature
   ↓
Checks expiration
   ↓
Checks user/access
   ↓
Makes security decision
```

The backend is the authority.

---

# 51. Frontend JWT Decoding vs `/me`

### Frontend decoding

Can answer:

> "What information is inside this token?"

### `/me`

Can answer:

> "Who is the currently authenticated user according to the backend right now?"

For example:

```text
JWT says:
User ID = 123

/me says:
User 123
Username = Mainak
Email = mainak@example.com
Profile image = ...
```

The `/me` endpoint can also reflect the latest database state.

---

# 52. Full Authentication Architecture

```text
                   REGISTER
                      │
                      ▼
              Validate input
                      │
                      ▼
              Hash password
                      │
                      ▼
                Save user
                      │
                      ▼
                   LOGIN
                      │
                      ▼
              Find user by email
                      │
                      ▼
             Verify password hash
                      │
                      ▼
                Create JWT
                      │
                      ▼
              Send token to FE
                      │
                      ▼
              Future API request
                      │
                      ▼
             Authorization header
                      │
                      ▼
             Verify JWT signature
                      │
                      ▼
               Extract user ID
                      │
                      ▼
              Find current user
                      │
                      ▼
            Check authorization
                      │
                      ▼
              Execute operation
```

---

# 53. Security Responsibilities

### Password hashing

Protects stored passwords if the database is compromised.

```text
Password → Hash → Database
```

### JWT signature

Protects the integrity of the token.

```text
JWT → Signed with secret
```

If someone modifies:

```json
{
    "sub": "123"
}
```

to:

```json
{
    "sub": "999"
}
```

the signature won't match.

### Expiration

Limits how long an access token remains valid.

```text
JWT
 ↓
expires in 30 minutes
```

### Database lookup

Allows the backend to check the user's current state.

```text
JWT → user ID
       ↓
Database → current user
```

---

# 54. Key Concepts to Remember

| Concept                     | Purpose                                           |
| --------------------------- | ------------------------------------------------- |
| Password hashing            | Securely store passwords                          |
| `pwdlib`                    | Password hashing helper                           |
| Argon2                      | Password hashing algorithm                        |
| JWT                         | Represent authenticated identity/claims           |
| PyJWT                       | Python JWT implementation                         |
| Secret key                  | Signs/verifies JWTs with HS256                    |
| `exp`                       | Token expiration                                  |
| `sub`                       | Token subject/user ID                             |
| Bearer token                | Token sent in `Authorization` header              |
| `OAuth2PasswordRequestForm` | Reads OAuth2 username/password form data          |
| `OAuth2PasswordBearer`      | Extracts bearer token and defines security scheme |
| `/login` / token endpoint   | Verifies credentials and issues JWT               |
| `/me`                       | Gets the current authenticated user's data        |
| `UserPublic`                | Safe public user representation                   |
| `UserPrivate`               | User representation containing private fields     |
| `SecretStr`                 | Protects secret values from accidental exposure   |
| `BaseSettings`              | Defines typed application configuration           |

---

# 🧠 The Most Important Mental Model

Don't memorize the code first.

Understand this:

```text
                    REGISTRATION

Password
   ↓
Hash
   ↓
Database


                    LOGIN

Email + Password
       ↓
Find User
       ↓
Verify Password Hash
       ↓
Create JWT
       ↓
Send JWT


                 PROTECTED REQUEST

JWT
 ↓
Verify Signature
 ↓
Check Expiration
 ↓
Extract User ID
 ↓
Find User
 ↓
Check Permissions
 ↓
Allow Request
```

The core idea is:

> **Password proves who you are during login. JWT represents that authenticated identity afterward. The backend verifies the JWT on protected requests and remains the final authority over access.**
