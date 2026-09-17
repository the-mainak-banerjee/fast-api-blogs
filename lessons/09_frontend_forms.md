# Lesson 9: Frontend Forms

**Lesson:** 9
**Video:** https://youtu.be/vqjZOyT4QRs?si=SEPS3uxZcxWZgL25

---

## 1. Order Posts by Date

To display the newest posts first, we can sort the posts by `date_posted` in **descending order**:

```python
order_by(models.Post.date_posted.desc())
```

### How it works

* `order_by()` → tells SQLAlchemy how to sort the results.
* `models.Post.date_posted` → the column we want to sort by.
* `.desc()` → sorts from highest/newest to lowest/oldest.

So:

```text
Newest post
    ↓
2nd newest
    ↓
3rd newest
    ↓
Oldest post
```

Equivalent SQL:

```sql
ORDER BY date_posted DESC
```

> **`.desc()` = newest → oldest**
> **`.asc()` = oldest → newest**
