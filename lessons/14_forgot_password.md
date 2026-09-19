# Lesson 14: Password Reset — Email, Tokens, and Background Tasks

**Video:** https://www.youtube.com/watch?v=4HxjBvZMAg8&list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI&index=14&t=78s

---

# 1. Password Reset — Core Requirements

A password reset token should be:

* **Single-use** → It should become invalid after being used.
* **Short-lived** → It should expire quickly.
* **Non-guessable** → An attacker should not be able to predict it.

The basic flow:

```text
User forgets password
        ↓
Enter email
        ↓
Server creates random reset token
        ↓
Store HASH of token in database
        ↓
Send original token through email
        ↓
User clicks reset link
        ↓
Server hashes received token
        ↓
Compare with database
        ↓
If valid → allow password reset
        ↓
Delete token
        ↓
Password changed
```

---

# 2. Email Library

```bash
uv add aiosmtplib
```

`aiosmtplib` is an **asynchronous SMTP client** for sending emails.

We use it instead of Python's built-in `smtplib` because `smtplib` is synchronous.

Since our FastAPI application is asynchronous, using an async email library avoids blocking the event loop while communicating with the mail server.

### SMTP

SMTP stands for:

> Simple Mail Transfer Protocol

It is the protocol used to send emails between mail clients and mail servers.

---

# 3. Email Configuration

```python
reset_token_expire_minutes: int = 60

mail_server: str = "localhost"
mail_port: int = 587
mail_username: str = ""
mail_password: SecretStr = SecretStr("")
mail_from: str = "noreply@example.com"
mail_use_tls: bool = True

frontend_url: str = "http://localhost:8000"
```

### Reset token expiration

```python
reset_token_expire_minutes = 60
```

The reset token is valid for only 60 minutes.

A production system can choose a shorter period depending on its security requirements.

---

# 4. Why Store `frontend_url` in Settings?

```python
frontend_url: str = "http://localhost:8000"
```

We use this to construct the reset link.

For example:

```text
http://localhost:8000/reset-password?token=abc123
```

We **don't take the frontend URL from the request**.

Why?

A request can be manipulated by an attacker.

If we blindly use a URL supplied by the request, an attacker could potentially cause the application to generate a password-reset link pointing to an attacker-controlled domain.

So:

```text
Trusted application configuration
        ↓
Frontend URL
        ↓
Reset link
```

instead of:

```text
User request
        ↓
Frontend URL
        ↓
Reset link
```

> **Important:** URLs used for security-sensitive links should come from trusted server configuration, not from untrusted request data.

---

# 5. Testing Emails with Mailtrap

During development, we can use **Mailtrap Sandbox** to test emails.

Instead of sending emails to real users:

```text
Application
    ↓
Mailtrap
    ↓
Test inbox
```

This allows us to inspect the email without actually delivering it to a real mailbox.

---

# 6. Rendering the Email Template

```python
template = templates.env.get_template(
    "email/password_reset.html"
)

html_content = template.render(
    reset_url=reset_url,
    username=username
)
```

Here we're generating an **HTML string** for the email.

We use:

```python
templates.env.get_template()
```

rather than:

```python
templates.TemplateResponse()
```

because `TemplateResponse` is designed for an **HTTP response**.

For example:

```text
Browser request
      ↓
TemplateResponse
      ↓
HTTP response
```

For email, we aren't responding to a browser request.

We simply need:

```text
Template
   ↓
HTML string
   ↓
Email body
   ↓
SMTP
```

So:

```text
TemplateResponse → Web page

template.render() → Email HTML
```

---

# 7. Password Reset Token Model

We create a separate database model for reset tokens.

Conceptually:

```text
PasswordResetToken

id
user_id
token_hash
expires_at
```

The database stores:

* Which user the token belongs to
* A hash of the token
* When the token expires

It does **not** need to store the original token.

---

# 8. Generate the Reset Token

```python
def generate_reset_token() -> str:
    return secrets.token_urlsafe(32)
```

This generates a cryptographically secure random token.

The important property is:

```text
Random + unpredictable
```

An attacker should not be able to guess another user's token.

---

# 9. Why SHA-256 Instead of Argon2?

For passwords we use:

```text
Argon2
```

For reset tokens we use:

```text
SHA-256
```

The reason is that passwords and reset tokens have different security properties.

### Password

Users often choose predictable passwords:

```text
password123
qwerty
mainak123
```

Therefore password hashing should be deliberately expensive and slow.

```text
Password
   ↓
Argon2
   ↓
Slow hash
```

This makes brute-force attacks more expensive.

### Reset token

Our reset token is generated using a cryptographically secure random generator:

```text
secrets.token_urlsafe(32)
```

It already has high entropy and should be practically impossible to guess.

Therefore we don't need an intentionally slow password-hashing algorithm.

```text
Random token
      ↓
SHA-256
      ↓
Token hash
```

### Important mental model

> **Passwords need slow hashing because they can be weak and predictable. Reset tokens are already random and unpredictable, so a fast cryptographic hash such as SHA-256 is sufficient for storing the token securely.**

---

# 10. Store the Hash, Not the Token

Suppose the generated token is:

```text
abcXYZ123...
```

We send this token to the user:

```text
Email
   ↓
https://example.com/reset-password?token=abcXYZ123
```

But the database stores:

```text
SHA256(abcXYZ123...)
```

not:

```text
abcXYZ123...
```

If the database is compromised, the attacker doesn't immediately obtain usable reset tokens.

---

# 11. Forgot Password Flow

Endpoint:

```python
@router.post("/forgot-password")
```

The complete flow is:

```text
User enters email
       ↓
POST /forgot-password
       ↓
Find user by email
       ↓
Does user exist?
   ↙           ↘
 No            Yes
 ↓              ↓
Return same     Delete existing
generic         reset tokens
response              ↓
                  Generate token
                       ↓
                  Hash token
                       ↓
                  Set expiration
                       ↓
                  Store hash in DB
                       ↓
                  Commit
                       ↓
              Schedule email task
                       ↓
                  Return response
```

---

# 12. Find the User

```python
result = await db.execute(
    select(models.User).where(
        func.lower(models.User.email)
        == request_data.email.lower(),
    ),
)

user = result.scalars().first()
```

We search for the user using a case-insensitive email comparison.

For example:

```text
MAINAK@example.com
mainak@example.com
Mainak@Example.com
```

can be treated as the same email.

---

# 13. Delete Existing Reset Tokens

```python
await db.execute(
    sql_delete(models.PasswordResetToken).where(
        models.PasswordResetToken.user_id == user.id,
    ),
)
```

Before creating a new token, we delete previous tokens for that user.

This keeps the system simple:

```text
Old token → invalid
New token → valid
```

So requesting another reset invalidates the previous reset token.

---

# 14. Create and Store the Token

```python
token = generate_reset_token()

token_hash = hash_reset_token(token)

expires_at = datetime.now(UTC) + timedelta(
    minutes=settings.reset_token_expire_minutes,
)
```

We now have:

```text
Raw token
    ↓
SHA-256
    ↓
Token hash
```

and:

```text
Current time + expiration period
    ↓
expires_at
```

Then we create the database record.

```text
User ID
Token hash
Expiration time
```

---

# 15. Why Store the Hash but Email the Original Token?

Because the user needs the original token to prove that they received the reset email.

```text
SERVER

Raw token ───────────────→ Email
   │
   ↓
SHA-256
   │
   ↓
Database
```

Later:

```text
User sends raw token
        ↓
Server hashes it
        ↓
Compare with database
```

If they match:

```text
Valid token
```

---

# 16. Background Tasks

```python
background_tasks.add_task(
    send_password_reset_email,
    to_email=user.email,
    username=user.username,
    token=token,
)
```

FastAPI's `BackgroundTasks` lets us schedule work to run **after the response is sent**.

Instead of:

```text
Request
  ↓
Create token
  ↓
Send email
  ↓
Wait for SMTP
  ↓
Response
```

we can do:

```text
Request
  ↓
Create token
  ↓
Store token
  ↓
Schedule email
  ↓
Response
  ↓
Send email in background
```

This means the user doesn't have to wait for the email server before receiving the API response.

---

# 17. Important Background Task Limitation

FastAPI `BackgroundTasks` are tied to the application process.

If the server crashes before the task runs:

```text
API
 ↓
Background task scheduled
 ↓
SERVER CRASH
 ↓
Task lost
```

Therefore, `BackgroundTasks` are suitable for lightweight/non-critical work.

For critical or guaranteed jobs, production systems often use a **durable task queue** such as:

```text
Celery
RQ
Dramatiq
```

with an appropriate message broker/backend.

---

# 18. Important: Database Session and Background Task

The background task runs **after the response is returned**.

Therefore, don't assume the request's database session will still be available when the background task executes.

For example:

```text
Request
   ↓
Database session
   ↓
Commit
   ↓
Response
   ↓
Request session lifecycle ends
   ↓
Background task
```

If the background task needs a database connection, it should obtain its own appropriate resources rather than relying on the request-scoped session.

In our case, the email task doesn't need the database session.

---

# 19. Why Don't We Tell the User Whether the Email Exists?

We always return:

```python
return {
    "message": "If an account exists with this email, "
               "you will receive password reset instructions.",
}
```

We don't say:

```text
Email exists
```

or:

```text
Email doesn't exist
```

Why?

Otherwise an attacker could test millions of email addresses and discover which ones have accounts.

This is called **user/account enumeration**.

---

# 20. Why `202 Accepted`?

The endpoint uses:

```python
status_code=status.HTTP_202_ACCEPTED
```

`202 Accepted` communicates that the request has been accepted for processing.

This fits our flow because email delivery happens as a background task.

More importantly, we return the **same response regardless of whether the email exists**.

```text
Existing email
     ↓
202 + generic message

Unknown email
     ↓
202 + same generic message
```

This helps prevent account enumeration.

---

# 21. Reset Password Flow

Endpoint:

```python
@router.post("/reset-password")
```

Complete flow:

```text
User submits token + new password
              ↓
          Hash token
              ↓
      Find token in database
              ↓
       Token exists?
        ↙          ↘
      NO           YES
       ↓             ↓
     400        Check expiration
                    ↓
              Expired?
              ↙      ↘
            YES      NO
             ↓        ↓
            400    Find user
                       ↓
                  User exists?
                   ↙       ↘
                 NO        YES
                  ↓          ↓
                 400     Hash new password
                              ↓
                         Update user
                              ↓
                    Delete reset token(s)
                              ↓
                           Commit
                              ↓
                           Success
```

---

# 22. Hash the Received Token

```python
token_hash = hash_reset_token(request_data.token)
```

The user sends:

```text
RAW TOKEN
```

We transform it:

```text
RAW TOKEN
    ↓
SHA-256
    ↓
HASH
```

Then search for that hash:

```python
select(models.PasswordResetToken).where(
    models.PasswordResetToken.token_hash == token_hash
)
```

---

# 23. Check Whether Token Exists

```python
if not reset_token:
    raise HTTPException(
        status_code=status.HTTP_400_BAD_REQUEST,
        detail="Invalid or expired reset token",
    )
```

No matching token means:

```text
Invalid token
```

---

# 24. Check Expiration

```python
if reset_token.expires_at.replace(tzinfo=UTC) < datetime.now(UTC):
```

Conceptually:

```text
Current time > expiration time
       ↓
Token expired
       ↓
Reject
```

If expired, we delete the token:

```python
await db.delete(reset_token)
await db.commit()
```

---

# 25. Find the User

The reset token contains the user's ID.

So we retrieve the user:

```text
Reset Token
     ↓
user_id
     ↓
Database
     ↓
User
```

If the user doesn't exist, the reset request is rejected.

---

# 26. Hash the New Password

```python
user.password_hash = hash_password(
    request_data.new_password
)
```

We **never store the new password directly**.

Instead:

```text
New password
      ↓
Argon2
      ↓
Password hash
      ↓
Database
```

---

# 27. Delete Reset Tokens After Successful Reset

```python
await db.execute(
    sql_delete(models.PasswordResetToken).where(
        models.PasswordResetToken.user_id == user.id,
    ),
)
```

This makes the reset process single-use.

After the password has been changed:

```text
Old reset token
      ↓
Deleted
      ↓
Cannot be reused
```

---

# 28. Change Password While Logged In

There are two different password-changing scenarios.

### Forgot password

```text
Don't know current password
        ↓
Email reset flow
        ↓
Reset token
        ↓
New password
```

### Change password

```text
Already logged in
        ↓
Provide current password
        ↓
Provide new password
        ↓
Change password
```

---

# 29. Change Password Flow

```text
Authenticated user
       ↓
Send current + new password
       ↓
Verify current password
       ↓
Correct?
   ↙       ↘
 NO        YES
 ↓          ↓
400     Hash new password
              ↓
        Update user record
              ↓
        Delete reset tokens
              ↓
            Commit
              ↓
           Success
```

---

# 30. Why Verify the Current Password?

Imagine someone gets access to an already-authenticated session.

If changing a password required only:

```text
New password
```

they could immediately change the account password.

Requiring:

```text
Current password + new password
```

adds another security check for this operation.

---

# 31. Why Delete Reset Tokens When Password Changes?

Suppose a user requested a password reset earlier.

Then they log in normally and change their password.

Any previously issued reset tokens should no longer remain valid.

Therefore:

```text
Password changed
      ↓
Invalidate outstanding reset tokens
```

This keeps authentication state consistent.

---

# 32. Reset Password Page

```python
@router.get("/reset-password", include_in_schema=False)
```

This serves the HTML page where the user enters their new password.

It is hidden from Swagger because it is a browser-facing HTML route rather than an API endpoint.

---

# 33. `Referrer-Policy: no-referrer`

```python
response.headers["Referrer-Policy"] = "no-referrer"
```

This is important because reset links contain sensitive information:

```text
/reset-password?token=VERY_SECRET_TOKEN
```

When a browser navigates from one page to another, it can send a `Referer` header containing information about the previous page, depending on browser policy.

We don't want the reset token leaking to another site through a referrer.

So:

```text
Reset URL containing token
          ↓
Browser
          ↓
Referrer-Policy: no-referrer
          ↓
Don't send previous URL as Referer
```

> **Important:** A password-reset token is a credential. Treat it as sensitive and minimize where it can leak.

---

# 34. Complete Password Reset System

```text
                    FORGOT PASSWORD

User
 ↓
Enter email
 ↓
POST /forgot-password
 ↓
Find user
 ↓
Generate secure random token
 ↓
Hash token
 ↓
Store hash + expiration
 ↓
Commit DB
 ↓
Schedule email
 ↓
Return 202
 ↓
Background task sends email
 ↓
User receives reset link
```

Then:

```text
                    RESET PASSWORD

User clicks link
 ↓
Reset page
 ↓
Submit token + new password
 ↓
Hash received token
 ↓
Find matching token
 ↓
Check expiration
 ↓
Find user
 ↓
Hash new password with Argon2
 ↓
Update password
 ↓
Delete reset token
 ↓
Commit
 ↓
Success
```

---

# 35. Security Properties

The system gives us several layers of protection:

```text
Secure random token
        +
Short expiration
        +
Single-use token
        +
Store only token hash
        +
Generic forgot-password response
        +
Trusted frontend URL
        +
Password hashing with Argon2
        +
Referrer-Policy
```

Each solves a different problem.

---

# 36. Important Engineering Points

### 1. Reset tokens should be:

```text
Short-lived
Single-use
Cryptographically random
Non-guessable
```

### 2. Don't store reset tokens in plaintext

```text
Token
 ↓
SHA-256
 ↓
Database
```

### 3. Don't use Argon2 for reset tokens just because it's used for passwords

Passwords can be weak and predictable.

Reset tokens are generated randomly with high entropy.

### 4. Don't reveal whether an account exists

Always return a generic message.

This prevents account enumeration.

### 5. Use trusted configuration for reset URLs

Don't construct security-sensitive URLs from attacker-controlled request values.

### 6. Background tasks aren't guaranteed

If losing an email task is unacceptable, use a durable task queue.

### 7. Don't reuse the request-scoped DB session inside a background task

The request lifecycle may already be finished.

### 8. Delete tokens after successful password reset

This makes them single-use.

### 9. Invalidate old reset tokens when appropriate

For example, when a new reset is requested or the password changes.

### 10. Protect reset URLs from referrer leakage

Use:

```text
Referrer-Policy: no-referrer
```

when serving the reset page.

---

# 37. Key Mental Model

The entire lesson can be remembered as:

```text
PASSWORD
    ↓
Argon2
    ↓
Slow + expensive hash
    ↓
Protect against password guessing


RESET TOKEN
    ↓
Cryptographically random
    ↓
SHA-256
    ↓
Short-lived + single-use
    ↓
Protect password-reset flow
```

> **Password security and password-reset-token security solve different problems, so they don't need the same hashing strategy.**
