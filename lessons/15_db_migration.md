# Lesson 15: PostgreSQL & Alembic — Database Migrations for Production

**Video:** https://www.youtube.com/watch?v=e8NnDz8uT7o&list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI&index=17

---

# 1. PostgreSQL Driver

```bash
uv add psycopg[binary]
```

`psycopg` is a Python driver that allows Python applications to communicate with PostgreSQL.

```text
Python application
       ↓
    Psycopg
       ↓
   PostgreSQL
```

`[binary]` installs the binary dependencies needed to make setup easier.

---

# 2. Database URL

Instead of hardcoding the database URL:

```python
SQLALCHEMY_DATABASE_URL = "..."
```

we put it in environment configuration:

```env
DATABASE_URL=postgresql+psycopg://...
```

Then the application reads it through our settings.

This is important because development, staging, and production use different databases.

```text
Development → Development DB
Staging     → Staging DB
Production  → Production DB
```

The application code doesn't need to change.

---

# 3. What We Were Doing Before

Previously we used:

```python
Base.metadata.create_all(engine)
```

This essentially means:

> "Create the tables described by my models if they don't already exist."

For example:

```text
Application starts
       ↓
create_all()
       ↓
Does users table exist?
       ↓
No → Create it
```

But what happens if the table already exists?

```text
users table already exists
       ↓
You add a new column to your model
       ↓
create_all()
       ↓
Existing table is NOT modified
```

`create_all()` is **not a database migration system**.

---

# 4. The Problem

Suppose our original table is:

```text
users

id
username
email
```

Later we change our model:

```text
users

id
username
email
bio       ← NEW
```

The database still has:

```text
id
username
email
```

because the table already exists.

We need a way to safely change the database structure.

That's what **database migrations** solve.

---

# 5. What Is a Migration?

A migration is a **recorded database schema change**.

For example:

```text
Migration 001
Create users table

Migration 002
Add bio column

Migration 003
Add profile_picture column

Migration 004
Create comments table
```

Think of migrations like **Git commits for your database schema**.

```text
Git

commit 1
commit 2
commit 3


Database

migration 1
migration 2
migration 3
```

This gives us a history of how the database structure evolved.

---

# 6. Why Do We Need Migrations?

Without migrations:

```text
Developer changes model
        ↓
How do we update production DB?
        ↓
Manually modify database
        ↓
Easy to make mistakes
```

With migrations:

```text
Developer changes model
        ↓
Create migration
        ↓
Review migration
        ↓
Apply migration
        ↓
Production DB updated
```

---

# 7. Alembic

We use **Alembic** as our migration tool.

Install it:

```bash
uv add alembic
```

Initialize it:

```bash
uv run alembic init -t async alembic
```

### `-t async`

Means:

```text
Use Alembic's asynchronous template
```

### `alembic`

The final argument is the directory where Alembic will create its files.

You'll get something similar to:

```text
alembic/
├── versions/
├── env.py
└── script.py.mako

alembic.ini
```

---

# 8. Alembic Configuration

In `alembic.ini`:

```ini
sqlalchemy.url =
```

We leave the URL empty because we don't want to hardcode our database credentials there.

Instead, we configure it in `env.py` using our application's settings.

---

# 9. `env.py`

We import our models and database configuration:

```python
from fastapi_blog import models

from fastapi_blog.config import settings

from fastapi_blog.database import Base
```

Then:

```python
config.set_main_option(
    "sqlalchemy.url",
    settings.database_url,
)
```

This tells Alembic:

> "Use the database URL from our application's configuration."

---

# 10. `target_metadata`

```python
target_metadata = Base.metadata
```

This is extremely important for autogeneration.

Our SQLAlchemy models describe what the database **should look like**.

For example:

```python
class User(Base):
    __tablename__ = "users"

    id = ...
    username = ...
    email = ...
```

SQLAlchemy builds metadata representing that structure.

We give that metadata to Alembic:

```text
SQLAlchemy Models
       ↓
   Base.metadata
       ↓
     Alembic
```

Alembic can then compare:

```text
Current database
        vs
SQLAlchemy models
```

and determine what schema changes are required.

---

# 11. The Important Mental Model

Think of your SQLAlchemy models as the **desired state**.

```text
        SOURCE OF TRUTH

      SQLAlchemy Models
              ↓
       Base.metadata
              ↓
           Alembic
              ↓
       Compare with DB
              ↓
       Migration script
```

So when you change:

```python
class User:
    ...
    bio: Mapped[str]
```

Alembic can detect:

```text
Database:
id
username
email

Model:
id
username
email
bio

Difference:
+ bio
```

and generate a migration.

---

# 12. Autogenerate a Migration

```bash
uv run alembic revision --autogenerate -m "initial schema"
```

This does **NOT** change the database.

It:

1. Looks at your SQLAlchemy models.
2. Looks at the current database schema.
3. Compares them.
4. Generates a migration file.

Think:

```text
Models
   +
Current DB
   ↓
Comparison
   ↓
Migration file
```

The migration file will be placed inside:

```text
alembic/versions/
```

---

# 13. Important: Autogenerate Does Not Mean "Automatically Apply"

This command:

```bash
alembic revision --autogenerate
```

only creates the migration.

It doesn't modify the database.

You should inspect the generated migration before applying it.

---

# 14. Apply the Migration

```bash
uv run alembic upgrade head
```

This applies migrations up to the latest migration.

```text
Migration 001
     ↓
Migration 002
     ↓
Migration 003
     ↓
     HEAD
```

`head` means:

> The latest migration in the migration history.

---

# 15. Adding a New Column

Suppose we add:

```python
likes: Mapped[int] = mapped_column(
    Integer,
    default=0,
    server_default="0",
)
```

The important part is:

```python
server_default="0"
```

Why?

Imagine the existing table already contains:

```text
Post 1
Post 2
Post 3
Post 4
```

Now we add:

```text
likes
```

What should the existing rows contain?

We need:

```text
Post 1 → likes = 0
Post 2 → likes = 0
Post 3 → likes = 0
Post 4 → likes = 0
```

A non-null column needs a valid value for existing rows.

---

# 16. `default` vs `server_default`

### `default`

```python
default=0
```

This is primarily an ORM/application-side default.

When SQLAlchemy creates a new object, it can use `0`.

### `server_default`

```python
server_default="0"
```

This tells the **database itself** to use `0` when no value is supplied.

Think:

```text
default
   ↓
Application / SQLAlchemy


server_default
   ↓
Database
```

For schema migrations, a database-level default can be important when adding a non-null column to an existing table.

---

# 17. PostgreSQL and Timezones

With PostgreSQL, we can use proper timezone-aware database types.

We don't need the SQLite-specific timezone workaround we previously used.

PostgreSQL has native support for timezone-aware timestamps.

For example:

```text
TIMESTAMP WITH TIME ZONE
```

This allows PostgreSQL to handle timezone-aware datetime values appropriately.

---

# 18. Language-Agnostic Database Workflow

Forget FastAPI, Python, SQLAlchemy, and Alembic for a moment.

The general database lifecycle is:

```text
Application
     ↓
Database
     ↓
Schema
     ↓
Schema changes
     ↓
Migration
     ↓
Apply migration
     ↓
New database schema
```

---

# 19. Database Creation

First we need a database.

```text
Application
      ↓
Database Server
      ↓
Database
      ↓
Tables
      ↓
Columns
      ↓
Indexes / Constraints
```

For example:

```text
Blog Database

users
posts
comments
```

---

# 20. Database Schema

A **database schema** describes the structure of the database.

It includes things like:

```text
Tables
Columns
Data types
Primary keys
Foreign keys
Indexes
Constraints
Relationships
```

For example:

```text
users

id          INTEGER
username    VARCHAR
email       VARCHAR
```

---

# 21. Application Model vs Database Schema

These are related but not the same thing.

```text
Application model
        ↓
Describes expected structure
```

```text
Database schema
        ↓
Actual structure stored in database
```

A migration keeps the two synchronized.

```text
Application Models
        ↓
Migration
        ↓
Database Schema
```

---

# 22. Migration Workflow

Suppose we add `bio`.

### Step 1 — Change model

```text
User
 ├── id
 ├── username
 ├── email
 └── bio ← NEW
```

### Step 2 — Generate migration

```bash
uv run alembic revision --autogenerate -m "add user bio"
```

### Step 3 — Review migration

Check what Alembic generated.

### Step 4 — Apply

```bash
uv run alembic upgrade head
```

Now:

```text
Application model
        ↕
Database schema
```

are synchronized.

---

# 23. Migration Rollback

Sometimes we need to undo a migration.

For example:

```text
Migration 001
Migration 002
Migration 003 ← Current
```

We can move back:

```bash
uv run alembic downgrade -1
```

This means:

> Go back one migration.

Result:

```text
Migration 001
Migration 002 ← Current
```

The migration file normally contains two operations:

```text
upgrade()
downgrade()
```

Conceptually:

```text
upgrade()
    ↓
Apply change


downgrade()
    ↓
Undo change
```

---

# 24. Migration History

To see migration history:

```bash
uv run alembic history
```

This shows the chain of migrations.

Conceptually:

```text
001 → 002 → 003 → 004
```

---

# 25. Check Current Database Revision

```bash
uv run alembic current
```

This tells you which migration revision the database is currently using.

---

# 26. Create an Empty Migration

Sometimes Alembic can't automatically detect a change or you want to write the migration manually.

```bash
uv run alembic revision -m "custom migration"
```

This creates a migration without autogeneration.

You then write the upgrade/downgrade operations yourself.

---

# 27. Upgrade to a Specific Revision

You can migrate to a specific revision:

```bash
uv run alembic upgrade <revision>
```

Or upgrade one migration:

```bash
uv run alembic upgrade +1
```

---

# 28. Downgrade to a Specific Revision

```bash
uv run alembic downgrade <revision>
```

Or go back one:

```bash
uv run alembic downgrade -1
```

---

# 29. Common Alembic Commands

| Command                                        | Purpose                                            |
| ---------------------------------------------- | -------------------------------------------------- |
| `alembic init -t async alembic`                | Initialize Alembic                                 |
| `alembic revision -m "message"`                | Create empty migration                             |
| `alembic revision --autogenerate -m "message"` | Generate migration from model/database differences |
| `alembic upgrade head`                         | Apply all pending migrations                       |
| `alembic upgrade +1`                           | Apply one migration                                |
| `alembic upgrade <revision>`                   | Upgrade to specific revision                       |
| `alembic downgrade -1`                         | Roll back one migration                            |
| `alembic downgrade <revision>`                 | Roll back to specific revision                     |
| `alembic current`                              | Show current DB revision                           |
| `alembic history`                              | Show migration history                             |

---

# 30. Production Migration Flow

This is the important real-world workflow.

A developer changes the application:

```text
Developer
   ↓
Change SQLAlchemy model
   ↓
Generate migration
   ↓
Review migration
   ↓
Commit model + migration
   ↓
Push to Git
```

Then CI/CD deploys the application:

```text
Git
 ↓
CI/CD
 ↓
Build application
 ↓
Run database migrations
 ↓
Deploy application
```

Production database:

```text
Current DB
    ↓
Migration 001
    ↓
Migration 002
    ↓
Migration 003
    ↓
Latest schema
```

---

# 31. Why Migrations Are Version Controlled

Imagine five developers are working on the same project.

Developer A adds:

```text
bio
```

Developer B adds:

```text
profile_picture
```

Developer C creates:

```text
comments
```

Without migrations, everyone could manually change the production database differently.

With migrations:

```text
001_add_bio
002_add_profile_picture
003_create_comments
```

Git tracks these files.

Everyone knows exactly how the database evolved.

---

# 32. Production Mental Model

Think of the database migration system like application deployment.

```text
CODE

v1 → v2 → v3 → v4


DATABASE

v1 → v2 → v3 → v4
```

The application version and database schema need to evolve together.

---

# 33. Important Production Rule

Never think:

> "I'll just modify the production database manually."

Instead:

```text
Change model
     ↓
Create migration
     ↓
Review migration
     ↓
Test migration
     ↓
Commit migration
     ↓
Deploy
     ↓
Run migration
```

This creates a reproducible database history.

---

# 34. Migration ≠ Backup

A migration tells you:

> **How to change the database schema.**

A backup tells you:

> **How to recover the database/data.**

They solve different problems.

```text
Migration → Schema evolution

Backup    → Disaster recovery
```

You need both in a production system.

---

# 35. What Happens If Migration Fails?

Imagine:

```text
Migration 001 ✓
Migration 002 ✓
Migration 003 ✗
```

The database should not simply be assumed to be fully migrated.

The migration system tracks revisions so we know where the database currently stands.

For important production migrations, teams test them against realistic data before deployment.

---

# 36. Migration Safety

Schema changes can become dangerous when databases are large.

For example:

```text
Add column
```

may be simple.

But:

```text
Rewrite millions of rows
```

can be expensive.

Production migrations should consider:

* Table size
* Locks
* Downtime
* Index creation
* Existing data
* Backward compatibility
* Rollback strategy

---

# 37. Zero-Downtime Migration Pattern

For large production systems, we often avoid:

```text
Change application
       +
Immediately break old application
```

Instead we can use an **expand → migrate → contract** strategy.

### Expand

Add the new database structure without removing the old one.

```text
Old column
New column
```

### Migrate

Move/update data gradually.

```text
Old data
   ↓
New structure
```

### Application migration

Deploy code that understands the new structure.

### Contract

After everything has migrated, remove the old structure.

```text
Old column → Remove
```

This is useful when multiple application instances are running during deployment.

---

# 38. Complete Production Workflow

```text
                 DEVELOPMENT

Developer changes model
          ↓
Generate migration
          ↓
Review migration
          ↓
Test migration
          ↓
Commit to Git
          ↓
          ↓
        CI/CD
          ↓
          ↓
     Production deploy
          ↓
   Run pending migrations
          ↓
      Database updated
          ↓
     Deploy application
```

The exact deployment ordering depends on whether the migration is backward-compatible, but the key idea is:

> **Application code and database schema must be evolved in a controlled, versioned way.**

---

# 39. The Big Picture

Our complete database architecture now looks like:

```text
                    FastAPI
                       │
                       ↓
                 SQLAlchemy ORM
                       │
                       ↓
                  DB Session
                       │
                       ↓
                    Engine
                       │
                       ↓
                  DB Driver
                       │
                       ↓
                  PostgreSQL
```

And schema management sits alongside it:

```text
SQLAlchemy Models
       ↓
   Base.metadata
       ↓
     Alembic
       ↓
Migration Files
       ↓
PostgreSQL Schema
```

---

# 40. Final Mental Model

Remember these five things:

```text
ENGINE
→ Manages database connectivity.

SESSION
→ Provides a unit of interaction/transaction with the database.

MODEL
→ Describes database structure to the application/ORM.

MIGRATION
→ Describes how the database structure changes over time.

ALEMBIC
→ Manages and applies those migrations.
```

And the most important production concept:

```text
Model change
     ↓
Migration
     ↓
Review
     ↓
Test
     ↓
Deploy
     ↓
Upgrade database
```

**`create_all()` creates missing tables.
Alembic migrations evolve existing databases safely and reproducibly.**
