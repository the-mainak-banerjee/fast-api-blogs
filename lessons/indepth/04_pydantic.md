# Pydantic

Pydantic is a Python library for **data validation and data modeling**.

It is commonly used in FastAPI to validate:

```text
Client Request
      ↓
   Pydantic
      ↓
Validated data
      ↓
Application
```

---

## 1. BaseModel

Pydantic models normally inherit from `BaseModel`.

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
```

`BaseModel` provides Pydantic's:

* Validation
* Type conversion
* Serialization
* Error handling
* Schema generation

---

## 2. Type Annotations

Always provide proper type annotations:

```python
name: str
age: int
is_active: bool
```

Pydantic uses these annotations to understand what type of data is expected.

For example:

```python
class User(BaseModel):
    age: int
```

If:

```python
User(age="25")
```

Pydantic can convert `"25"` to:

```python
25
```

if the conversion is valid.

But:

```python
User(age="hello")
```

cannot be converted to an integer, so Pydantic raises a validation error.

> **Important:** Pydantic is not simply "strictly rejecting everything with the wrong type." By default, it may perform safe/coercible conversions.

---

## 3. Creating a Model from a Dictionary

When passing a dictionary to a Pydantic model, use `**` to unpack it:

```python
user_data = {
    "name": "Mainak",
    "age": 25
}

user = User(**user_data)
```

`**` converts:

```python
{
    "name": "Mainak",
    "age": 25
}
```

into:

```python
name="Mainak", age=25
```

---

# Python Typing + Pydantic

You can freely combine Python's typing features with Pydantic.

```python
from typing import List, Dict, Optional
from pydantic import BaseModel

class User(BaseModel):
    name: str
    tags: List[str]
    scores: Dict[str, int]
    nickname: Optional[str] = None
```

### Important clarification

`List`, `Dict`, and `Optional` come from **Python's `typing` module**, not Pydantic.

```text
Pydantic
→ BaseModel
→ Field
→ field_validator
→ ConfigDict

Python typing
→ List
→ Dict
→ Optional
→ Union
```

Modern Python can also use:

```python
list[str]
dict[str, int]
str | None
```

---

# 4. Default Values

Provide a default value when a field is optional or has a meaningful default.

```python
class User(BaseModel):
    name: str
    age: int = 18
    nickname: str | None = None
```

Here:

```text
name     → required
age      → defaults to 18
nickname → optional, defaults to None
```

---

# 5. Field

`Field()` allows you to add constraints and metadata to a field.

```python
from pydantic import BaseModel, Field

class Employee(BaseModel):
    name: str = Field(
        min_length=3,
        description="Name of the employee"
    )
```

Now:

```python
name="Al"
```

will fail because the minimum length is 3.

### Field can define:

```python
Field(
    min_length=3,
    max_length=50,
    description="Employee name"
)
```

It is also useful for generating better API documentation.

---

# 6. Computed Fields

A computed field is a value calculated from other model fields.

```python
from pydantic import BaseModel, computed_field

class Product(BaseModel):
    price: float
    items: int

    @computed_field
    @property
    def total_amount(self) -> float:
        return self.price * self.items
```

Usage:

```python
product.total_amount
```

If:

```text
price = 100
items = 3
```

then:

```text
total_amount = 300
```

### Why both decorators?

```python
@computed_field
@property
```

`@property` makes it behave like an attribute:

```python
product.total_amount
```

`@computed_field` tells Pydantic:

> Include this calculated value as part of the Pydantic model's serialization/schema.

---

# 7. Field Validation

Use `@field_validator` when you want to modify or validate a specific field.

```python
from pydantic import BaseModel, field_validator

class User(BaseModel):
    email: str

    @field_validator("email")
    @classmethod
    def normalize_email(cls, v):
        return v.lower().strip()
```

Input:

```python
email="  MAINAK@EMAIL.COM "
```

becomes:

```text
mainak@email.com
```

### Why use this?

To enforce rules such as:

```text
Normalize email
Validate username
Clean phone number
Validate a specific format
```

---

# 8. Nested Models

A Pydantic model can contain another Pydantic model.

```python
class Address(BaseModel):
    street: str
    city: str
    postal_code: str


class User(BaseModel):
    id: int
    name: str
    address: Address
```

Now `address` must itself follow the `Address` schema.

```python
user = User(
    id=1,
    name="Hitesh",
    address=Address(
        street="123 something",
        city="Jaipur",
        postal_code="100001"
    )
)
```

Pydantic also handles nested dictionaries:

```python
user_data = {
    "id": 1,
    "name": "Hitesh",
    "address": {
        "street": "321 something",
        "city": "Paris",
        "postal_code": "20002"
    }
}

user = User(**user_data)
```

Pydantic validates the nested `address` as an `Address` model.

---

# 9. Optional Nested Models

A nested model doesn't always have to exist.

```python
class Company(BaseModel):
    name: str
    address: Address | None = None


class Employee(BaseModel):
    name: str
    company: Company | None = None
```

This allows:

```python
Employee(name="Mainak")
```

without providing a company.

---

# 10. Recursive / Self-Referencing Models

A model can contain itself.

This is useful for structures like:

```text
Comment
 ├── Reply
 │    ├── Reply
 │    └── Reply
 └── Reply
```

Example:

```python
from typing import Optional
from pydantic import BaseModel

class Comment(BaseModel):
    id: int
    content: str
    replies: Optional[list["Comment"]] = None
```

Here:

```python
replies: Optional[list["Comment"]]
```

means:

```text
replies can be:

None

OR

a list of Comment objects
```

For example:

```python
Comment(
    id=1,
    content="First comment",
    replies=[
        Comment(
            id=2,
            content="Reply"
        )
    ]
)
```

---

## Why `model_rebuild()`?

```python
Comment.model_rebuild()
```

When Pydantic initially creates the `Comment` class, the `Comment` type inside:

```python
list["Comment"]
```

is a **forward reference**.

In other words, you're saying:

> "This field will contain instances of the class I'm currently defining."

`model_rebuild()` tells Pydantic to resolve that reference and rebuild the model's schema once the class definition exists.

In modern Pydantic versions, some forward-reference situations can be resolved automatically, but `model_rebuild()` is still useful when explicit rebuilding is required.

---

# 11. Union Types

A Union means:

> A value can be one of several types.

Traditional syntax:

```python
from typing import Union

age: Union[int, str]
```

Modern Python:

```python
age: int | str
```

This means:

```text
age
 ↓
integer OR string
```

It can also be used with models:

```python
class Response(BaseModel):
    data: User | list[User]
```

Now `data` can contain:

```text
User
```

or:

```text
list[User]
```

This is useful when an API can return different valid structures.

---

# 12. `ConfigDict`

`ConfigDict` is used to configure how a Pydantic model behaves.

```python
from pydantic import BaseModel, ConfigDict

class UserResponse(BaseModel):
    model_config = ConfigDict(
        from_attributes=True
    )
```

Think of it as:

```text
BaseModel
    +
Configuration
    ↓
How should this model behave?
```

For example:

```python
from_attributes=True
```

tells Pydantic:

> "This model is allowed to read values from object attributes, not only dictionaries."

This is particularly useful with SQLAlchemy.

### Without it

Pydantic primarily expects something like:

```python
{
    "id": 1,
    "username": "Mainak"
}
```

### With `from_attributes=True`

It can read:

```python
user.id
user.username
```

from a SQLAlchemy object.

This is why you used:

```python
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
```

with your database models.

---

# 13. `model_dump()`

`model_dump()` converts a Pydantic model into a Python dictionary.

```python
user = User(
    name="Mainak",
    age=25
)

data = user.model_dump()
```

Result:

```python
{
    "name": "Mainak",
    "age": 25
}
```

Think:

```text
Pydantic Model
      ↓
model_dump()
      ↓
Python dict
```

This is useful when you need to:

* Send data somewhere
* Update a database model
* Manipulate the data
* Pass the data to another function

---

# 14. `model_dump_json()`

`model_dump_json()` converts the Pydantic model directly into a JSON string.

```python
user.model_dump_json()
```

Result:

```json
{"name":"Mainak","age":25}
```

Difference:

```text
model_dump()
    ↓
Python dictionary


model_dump_json()
    ↓
JSON string
```

### Example

```python
user.model_dump()
```

gives:

```python
{"name": "Mainak", "age": 25}
```

while:

```python
user.model_dump_json()
```

gives:

```text
'{"name":"Mainak","age":25}'
```

A Python `dict` and a JSON string are **not the same thing**.

---

# 15. Lazy Loading

Lazy loading means:

> Don't load something until it is actually needed.

Imagine:

```python
class User(BaseModel):
    posts: list[Post]
```

and the user has 1,000 posts.

Instead of immediately loading all 1,000 posts:

```text
Load User
   ↓
Wait
   ↓
Access user.posts
   ↓
Load posts
```

This can save unnecessary work when you don't actually need the posts.

### Example mental model

```text
User requested
    ↓
Load user information
    ↓
Do I need posts?
    ↓
No → Don't load posts

Yes → Load posts
```

Lazy loading is more commonly a **database/ORM loading strategy** than a Pydantic feature.

With SQLAlchemy relationships, you might configure how related data is loaded.

For example:

```python
relationship(...)
```

can use lazy loading.

---

# 16. Lazy Loading vs Eager Loading

### Lazy loading

```text
Load User
   ↓
Later request Posts
   ↓
Load Posts
```

### Eager loading

```text
Load User
   +
Load Posts
   ↓
Return everything together
```

For example, with SQLAlchemy:

```python
select(User).options(
    selectinload(User.posts)
)
```

This explicitly asks SQLAlchemy to load the related posts.

This is particularly important when using async database access because implicit lazy database queries can cause problems.

---

# 17. Best Practices — Model Organization

### Define leaf models first

Start with models that have no dependencies.

```text
Address
   ↓
Company
   ↓
Employee
```

### Build upwards

Compose more complex models from simpler ones.

### Use clear names

Prefer:

```text
UserCreate
UserPublic
UserPrivate
PostResponse
```

over vague names such as:

```text
Data
Response
Info
Object
```

### Group related models

For larger projects:

```text
schemas/
├── user.py
├── post.py
├── comment.py
└── auth.py
```

---

# 18. Performance Considerations

### Deep nesting

Very deeply nested models can increase validation and serialization work.

### Large nested lists

Avoid returning thousands of nested objects at once.

Use pagination:

```text
1000 posts
   ↓
20 posts per request
```

### Circular references

Be careful with circular relationships.

For example:

```text
User
 ↓
Post
 ↓
User
 ↓
Post
 ↓
...
```

Poorly designed recursive structures can cause excessive processing or serialization problems.

### Lazy loading

Use lazy loading when related data is expensive and isn't always needed.

Use eager loading when you know the related data is required.

---

# 19. Data Modeling Tips

### Model real-world relationships

Your models should represent your domain.

```text
User
 ↓
Posts
 ↓
Comments
```

### Use Optional appropriately

Don't make everything optional.

```python
email: str
```

means the application requires an email.

```python
bio: str | None = None
```

means a bio isn't required.

### Use Union types carefully

Use unions when the API genuinely supports multiple valid types.

```python
data: User | list[User]
```

Don't use them just to avoid defining a clear schema.

### Validate business rules

Use validators when the rule belongs to the data model.

Examples:

```text
Email normalization
Password requirements
Username format
Date relationships
Cross-field validation
```

---

# 20. Pydantic Mental Model

Think of Pydantic as the **gatekeeper for data**.

```text
             Incoming Data
                   ↓
              Pydantic
                   ↓
        ┌──────────┴──────────┐
        ↓                     ↓
    Valid data            Invalid data
        ↓                     ↓
 Application             Validation Error
```

And when sending data out:

```text
Database/Object
      ↓
Pydantic Response Model
      ↓
Validation + Serialization
      ↓
API Response
```

---

# 21. Most Important Things to Remember

```text
BaseModel
→ Base class for Pydantic models.

Type annotations
→ Tell Pydantic what data is expected.

Field
→ Adds constraints and metadata.

field_validator
→ Validates/transforms a specific field.

computed_field
→ Adds calculated values to serialized Pydantic models.

Nested model
→ A Pydantic model inside another Pydantic model.

Recursive model
→ A model that references itself.

Union
→ A value can have multiple allowed types.

ConfigDict
→ Controls Pydantic model behavior.

from_attributes=True
→ Allows Pydantic to read values from object attributes,
   useful with SQLAlchemy.

model_dump()
→ Pydantic model → Python dict.

model_dump_json()
→ Pydantic model → JSON string.

Lazy loading
→ Load related/expensive data only when needed.
```

### The big picture

```text
                 Pydantic
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
     Validate     Model      Serialize
        │           │           │
        ↓           ↓           ↓
     Input       Structure    Output
```

**Pydantic's main job is to make the data entering and leaving your application predictable, validated, and structured.**
