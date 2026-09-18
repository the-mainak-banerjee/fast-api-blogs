# Lesson 12: File Upload

**Video:** https://www.youtube.com/watch?v=AExumWjfbyo&list=PL-osiE80TeTsak-c-QsVeg0YYG_0TeyXI&index=12

---

# 1. Installing Pillow

```bash
uv add pillow
```

**Pillow** is a Python library for working with images.

We use it to:

* Open and validate images
* Resize images
* Crop images
* Convert image formats
* Correct image orientation
* Compress images
* Save processed images

---

# 2. Overall File Upload Flow

The important thing is to understand the system rather than the syntax.

```text
User selects image
       ↓
Frontend sends image
       ↓
Backend receives file
       ↓
Authenticate user
       ↓
Check user is allowed to update
       ↓
Read file
       ↓
Check file size
       ↓
Validate actual image content
       ↓
Process image
       ↓
Generate unique filename
       ↓
Save new image
       ↓
Update database
       ↓
Delete old image
       ↓
Return updated user
```

The database stores the **filename/reference**, while the actual image is stored on the filesystem.

```text
Database
   ↓
image_file = "abc123.jpg"

Filesystem
   ↓
media/profile_pics/abc123.jpg
```

---

# 3. Why `uuid`?

```python
import uuid
```

We use UUID to generate a unique filename.

```python
filename = f"{uuid.uuid4().hex}.jpg"
```

Instead of:

```text
profile.jpg
```

we get something like:

```text
a8f4c7e91....jpg
```

This prevents users from accidentally overwriting each other's files.

It also avoids relying on the filename supplied by the user.

---

# 4. Why `Path`?

```python
from pathlib import Path

PROFILE_PICS_DIR = Path("media/profile_pics")
```

`Path` provides a clean, cross-platform way to work with filesystem paths.

Instead of manually building strings:

```text
media/profile_pics/image.jpg
```

we can use:

```python
PROFILE_PICS_DIR / filename
```

---

# 5. Processing the Image

```python
def process_profile_image(content: bytes) -> str:
```

The function receives the image as raw bytes.

Conceptually:

```text
Uploaded file
     ↓
Bytes
     ↓
Image processing
     ↓
Saved image
     ↓
Filename returned
```

---

# 6. Opening the Image

We use this context manager to open the image
```python
with Image.open(BytesIO(content)) as original:
```

The uploaded file is currently just bytes.

```text
bytes
  ↓
BytesIO
  ↓
Pillow
  ↓
Image
```

Pillow can then inspect the actual image.

This is better than simply trusting:

```text
Content-Type: image/jpeg
```

because a client can lie about the MIME type.

---

# 7. Fixing Image Orientation

```python
img = ImageOps.exif_transpose(original)
```

Some cameras and phones store the image orientation inside EXIF metadata instead of physically rotating the pixels.

This function uses that information to make sure the image is displayed in the correct orientation.

---

# 8. Resize and Crop

```python
img = ImageOps.fit(
    img,
    (300, 300),
    method=Image.Resampling.LANCZOS
)
```

This creates a **300 × 300** image.

`ImageOps.fit()` maintains the image's proportions while cropping as necessary.

For example:

```text
Original
1200 × 800
     ↓
Fit
300 × 300
```

Some parts may be cropped to achieve the square shape.

`LANCZOS` is a high-quality resampling method used when resizing images.

---

# 9. Convert Image Mode

```python
if img.mode in ("RGBA", "LA", "P"):
    img = img.convert("RGB")
```

JPEG does not support transparency.

Some uploaded images may contain:

```text
RGBA → RGB + Alpha/transparency
LA   → grayscale + alpha
P    → palette-based image
```

So we convert them to:

```text
RGB
```

before saving as JPEG.

---

# 10. Generate the Filename

```python
filename = f"{uuid.uuid4().hex}.jpg"
```

The original filename is not used.

For example:

```text
mainak_profile.png
```

might become:

```text
7f92c4....jpg
```

This gives us a unique server-controlled filename.

---

# 11. Create the Directory

```python
PROFILE_PICS_DIR.mkdir(
    parents=True,
    exist_ok=True
)
```

This makes sure:

```text
media/profile_pics
```

exists.

### `parents=True`

Creates missing parent directories too.

### `exist_ok=True`

Doesn't throw an error if the directory already exists.

---

# 12. Save the Image

```python
img.save(
    filepath,
    "JPEG",
    quality=85,
    optimize=True
)
```

The processed image is saved as JPEG.

### `quality=85`

Controls JPEG compression/quality.

### `optimize=True`

Asks Pillow to optimize the JPEG representation.

The goal is:

```text
Large uploaded image
        ↓
Resize
        ↓
Compress
        ↓
Smaller optimized image
```

This improves storage and delivery performance.

---

# 13. Delete Profile Image

```python
def delete_profile_image(filename: str | None) -> None:
```

This removes an existing profile image.

If there is no image:

```python
if filename is None:
    return
```

Otherwise:

```python
filepath = PROFILE_PICS_DIR / filename
```

finds the file.

Then:

```python
if filepath.exists():
    filepath.unlink()
```

deletes it.

---

# 14. Why Image Processing Is Synchronous

This function:

```python
process_profile_image()
```

does CPU-heavy work:

```text
Decode image
Resize
Crop
Convert
Compress
Encode JPEG
```

It isn't primarily waiting for another service.

It's **CPU-bound work**.

Our API endpoint, however, is asynchronous:

```python
async def upload_profile_picture(...)
```

We don't want heavy CPU work blocking the event loop.

---

# 15. Threadpool

So we use:

```python
await run_in_threadpool(
    process_profile_image,
    content
)
```

Conceptually:

```text
Async event loop
       │
       ├── Request A → waiting for DB
       ├── Request B → waiting for DB
       ├── Request C → waiting for API
       │
       └── Image processing
               ↓
          Worker thread
```

The event loop can continue handling other asynchronous work while the synchronous image-processing function runs in a worker thread.

> **Async is useful for waiting; CPU-heavy synchronous work can be moved away from the event loop.**

---

# 16. Upload Endpoint

```python
@router.patch("/{user_id}/picture")
```

This endpoint is specifically responsible for changing the profile picture.

It is intentionally separate from:

```text
PATCH /users/{user_id}
```

The general user-update endpoint should not handle file uploads.

---

# 17. Authorization Check

```python
if current_user.id != user_id:
```

We check:

```text
Who is logged in?
        ↓
Who are they trying to modify?
        ↓
Same user?
```

If not:

```python
raise HTTPException(
    status_code=status.HTTP_403_FORBIDDEN,
    detail="Not authorized to update this user's picture",
)
```

This is **403 Forbidden** because the user is authenticated, but they aren't allowed to modify that user's picture.

---

# 18. Read the File

```python
content = await file.read()
```

The uploaded file is read into memory as bytes.

```text
Uploaded file
      ↓
bytes
      ↓
Image processing
```

This is an I/O operation, so it is awaited.

---

# 19. Check File Size

```python
if len(content) > settings.max_upload_size_bytes:
```

We prevent excessively large uploads.

Why?

A malicious or careless client could upload:

```text
500 MB
2 GB
10 GB
```

and consume server memory/storage.

So we enforce a limit.

---

# 20. Validate the Actual Image

We don't rely only on:

```python
file.content_type
```

because the client controls that information.

Instead, Pillow actually tries to open/process the content.

```python
try:
    new_filename = await run_in_threadpool(
        process_profile_image,
        content
    )
```

If Pillow cannot recognize it:

```python
except UnidentifiedImageError:
```

we return:

```text
400 Bad Request
```

This means:

> The uploaded content isn't a valid supported image.

---

# 21. Why Save the New Image Before Deleting the Old One?

This is a very important reliability detail.

Suppose the user's existing image is:

```text
old.jpg
```

We want to upload:

```text
new.jpg
```

### Bad order

```text
Delete old.jpg
      ↓
Process new image
      ↓
Processing fails
```

Now the user has **no profile picture**.

### Better order

```text
Process/save new.jpg
      ↓
Update database
      ↓
Delete old.jpg
```

If processing the new image fails, the old image remains available.

This is a basic **failure-safety pattern**:

> Don't destroy the working resource until the replacement is ready.

---

# 22. Save the New Filename in Database

```python
old_filename = current_user.image_file

current_user.image_file = new_filename
```

We first remember the old filename.

Then update the database object:

```text
Before:

image_file = old.jpg

After:

image_file = new.jpg
```

Then:

```python
await db.commit()
```

persists the database change.

---

# 23. Refresh the User

```python
await db.refresh(current_user)
```

This refreshes the SQLAlchemy object with the current database state.

Then we delete:

```python
if old_filename:
    delete_profile_image(old_filename)
```

Now the old file is no longer needed.

---

# 24. Delete Profile Picture Endpoint

The delete flow is:

```text
Request
  ↓
Authenticate
  ↓
Check ownership
  ↓
Check picture exists
  ↓
Set image_file = None
  ↓
Commit database
  ↓
Delete physical image
  ↓
Return user
```

---

# 25. Why Update Database Before Deleting the File?

We do:

```python
current_user.image_file = None
await db.commit()
await db.refresh(current_user)
```

before:

```python
delete_profile_image(old_filename)
```

This ensures the database no longer points to the image before we remove the actual file.

First remove the application's reference to the file, then remove the file itself.

---

# 26. Account Deletion Must Also Delete the Image

When deleting a user account, we should also remove their profile picture.

Otherwise:

```text
User deleted
   ↓
Database record deleted
   ↓
Profile image remains
   ↓
Unused file
```

These are called **orphaned files**.

So the user deletion flow should be:

```text
Get user
   ↓
Remember image_file
   ↓
Delete user from database
   ↓
Commit
   ↓
Delete profile image
```

For example, conceptually:

```python
old_filename = user.image_file

await db.delete(user)
await db.commit()

if old_filename:
    delete_profile_image(old_filename)
```

The exact implementation may also need to account for related posts and database cascade rules.

---

# 27. Remove `image_file` From `UserUpdate`

The general user-update schema should **not** contain:

```python
image_file
```

Profile image changes have their own endpoint:

```text
PATCH /users/{user_id}/picture
```

While normal user updates use:

```text
PATCH /users/{user_id}
```

This gives each endpoint a clear responsibility.

```text
User update
 ├── username
 └── email

Picture update
 └── image file
```

This is cleaner API design and also prevents someone from trying to manipulate the stored filename directly.

The client should upload the **actual image**, not provide:

```json
{
    "image_file": "some-other-users-image.jpg"
}
```

The server should control the generated filename.

---

# 28. Important Security Boundary

Never trust these values from the client:

```text
Original filename
Content-Type
File extension
Image dimensions
```

The server should:

```text
Receive file
   ↓
Check size
   ↓
Inspect actual content
   ↓
Process/convert image
   ↓
Generate its own filename
   ↓
Save in controlled directory
```

This is much safer than simply doing:

```text
uploaded_filename → save directly
```

---

# 29. Filesystem vs Database

For this project:

### Database

Stores metadata/reference:

```text
image_file = "abc123.jpg"
```

### Filesystem

Stores the actual image:

```text
media/profile_pics/abc123.jpg
```

Think:

```text
Database
"What file belongs to this user?"

Filesystem
"Here is the actual file."
```

For a larger production system, images are commonly stored in object storage such as S3-compatible storage rather than the application's local filesystem.

---

# 30. Complete Mental Model

**Upload Image** 

┌──────────────────────┐
│ FE uploads image     │
│ PATCH /picture       │
└──────────┬───────────┘
           ↓
┌─────────────────────────────┐
│ Authenticate + authorize   │
│ Is current user allowed?   │
└──────────┬──────────────────┘
           │
      ┌────┴────┐
      │         │
     NO        YES
      │         │
      ↓         ↓
┌──────────┐  ┌──────────────────┐
│ 403      │  │ Read uploaded    │
│ Forbidden│  │ file as bytes    │
└──────────┘  └────────┬─────────┘
                       ↓
              ┌──────────────────────┐
              │ Check file size      │
              │ <= max upload size?  │
              └──────────┬───────────┘
                         │
                    ┌────┴────┐
                    │         │
                   NO        YES
                    │         │
                    ↓         ↓
             ┌────────────┐  ┌─────────────────────┐
             │ 400        │  │ Process image in    │
             │ File too   │  │ threadpool          │
             │ large      │  └──────────┬──────────┘
             └────────────┘             ↓
                              ┌──────────────────────┐
                              │ Is it a valid image? │
                              └──────────┬───────────┘
                                         │
                                    ┌────┴────┐
                                    │         │
                                   NO        YES
                                    │         │
                                    ↓         ↓
                              ┌──────────┐  ┌────────────────────┐
                              │ 400     │  │ New image saved    │
                              │ Invalid │  │ + new filename     │
                              │ image   │  └─────────┬──────────┘
                              └──────────┘            ↓
                                      ┌────────────────────────┐
                                      │ Store old image file   │
                                      │ name/path              │
                                      └───────────┬────────────┘
                                                  ↓
                                      ┌────────────────────────┐
                                      │ Set user.image_file    │
                                      │ = new filename         │
                                      └───────────┬────────────┘
                                                  ↓
                                      ┌────────────────────────┐
                                      │ Commit database         │
                                      └───────────┬────────────┘
                                                  ↓
                                      ┌────────────────────────┐
                                      │ Refresh user object    │
                                      └───────────┬────────────┘
                                                  ↓
                                      ┌────────────────────────┐
                                      │ Delete old image       │
                                      └───────────┬────────────┘
                                                  ↓
                                      ┌────────────────────────┐
                                      │ Return updated user    │
                                      └────────────────────────┘



**Image processing flow**

        Image bytes
             ↓
┌────────────────────────────┐
│ Image.open(BytesIO(bytes)) │
└──────────────┬─────────────┘
               ↓
      Can Pillow recognize it?
          │             │
         NO            YES
          │             │
          ↓             ↓
   InvalidImage    EXIF transpose
     Error              ↓
                  Correct orientation
                        ↓
                 Resize + crop
                   to 300×300
                        ↓
                 LANCZOS resize
                        ↓
              Check image color mode
                        ↓
               RGBA / LA / P ?
                  │          │
                 YES        NO
                  │          │
                  ↓          │
             Convert to RGB  │
                  │          │
                  └────┬─────┘
                       ↓
              Generate UUID filename
                       ↓
                  e.g. abc123.jpg
                       ↓
             Create directory if needed
                       ↓
               Save as JPEG
                       ↓
              quality = 85
              optimize = True
                       ↓
             Return new filename



**Delete Image**

┌──────────────────────────┐
│ FE sends DELETE request  │
│ /users/{user_id}/picture│
└────────────┬─────────────┘
             ↓
┌──────────────────────────────┐
│ Authenticate the user        │
│ Who is making this request?  │
└────────────┬─────────────────┘
             ↓
┌──────────────────────────────┐
│ Is current_user.id ==        │
│ requested user_id?           │
└────────────┬─────────────────┘
             │
        ┌────┴────┐
        │         │
       NO        YES
        │         │
        ↓         ↓
┌───────────┐  ┌──────────────────────┐
│ 403       │  │ Get current          │
│ Forbidden │  │ image_file           │
└───────────┘  └──────────┬───────────┘
                           ↓
                 ┌──────────────────────┐
                 │ Does image exist?    │
                 └──────────┬───────────┘
                            │
                       ┌────┴────┐
                       │         │
                      NO        YES
                       │         │
                       ↓         ↓
               ┌────────────┐  ┌─────────────────────┐
               │ 400        │  │ Store old filename  │
               │ No profile │  └──────────┬──────────┘
               │ picture    │             ↓
               └────────────┘  ┌─────────────────────┐
                               │ Set image_file =     │
                               │ None                 │
                               └──────────┬──────────┘
                                          ↓
                               ┌─────────────────────┐
                               │ Commit DB           │
                               └──────────┬──────────┘
                                          ↓
                               ┌─────────────────────┐
                               │ Refresh user        │
                               └──────────┬──────────┘
                                          ↓
                               ┌─────────────────────┐
                               │ Delete old physical │
                               │ image from storage  │
                               └──────────┬──────────┘
                                          ↓
                               ┌─────────────────────┐
                               │ Return updated user │
                               └─────────────────────┘

---

# 31. Key Concepts

| Concept          | Meaning                                                     |
| ---------------- | ----------------------------------------------------------- |
| Pillow           | Python library for image processing                         |
| UUID             | Generates unique identifiers                                |
| `BytesIO`        | Treats bytes like a file in memory                          |
| EXIF             | Metadata that can contain image orientation                 |
| `ImageOps.fit()` | Resize + crop image to target dimensions                    |
| RGB              | Color mode used for the final JPEG                          |
| Threadpool       | Runs blocking synchronous work outside the async event loop |
| `UploadFile`     | FastAPI abstraction for uploaded files                      |
| `image_file`     | Database reference to the stored image                      |
| Orphaned file    | File remaining after its related database record is gone    |
| 400              | Invalid upload/request                                      |
| 401              | Authentication problem                                      |
| 403              | Authenticated but not allowed                               |
| 404              | User/resource doesn't exist                                 |

---

# 🧠 Main Engineering Lesson

The important pattern from this lesson isn't just "how to upload an image."

It's:

```text
Validate
   ↓
Authorize
   ↓
Process
   ↓
Create replacement
   ↓
Update reference
   ↓
Remove old resource
```

And remember:

> **The database stores the reference; the storage system stores the actual file.**

This separation becomes especially important when moving from a local filesystem to object storage such as S3 in a production system.
