# Chapter 09: Web Frameworks — Follow-Up Traps

## Trap 1: "`select_related` is always better than `prefetch_related`"

**What most people say:** "I use `select_related` to avoid N+1 queries."

**Correct answer:** `select_related` performs a SQL JOIN and works only for `ForeignKey` and `OneToOne` relationships. It loads the related object in the *same query*, which is ideal for one-to-one or many-to-one lookups. `prefetch_related` works for `ManyToManyField` and reverse `ForeignKey` (one-to-many): it performs two queries (one for the main objects, one for all related objects) and Python-side joins them. Using `select_related` on a `ManyToManyField` would be invalid. The confusion: `prefetch_related` CAN be used on ForeignKey too, but will issue 2 queries instead of 1 JOIN.

```python
# WRONG: select_related on a ManyToManyField
orders = Order.objects.select_related("tags")   # Error or unexpected behavior

# CORRECT patterns:
# ForeignKey (one customer per order): SELECT ... JOIN
orders = Order.objects.select_related("customer")

# Reverse FK (one order has many line items): 2 queries, Python join
orders = Order.objects.prefetch_related("line_items")

# ManyToMany: 2 queries
orders = Order.objects.prefetch_related("tags")

# COMBINED: 1 JOIN for customer + 1 extra query for line_items
orders = Order.objects.select_related("customer").prefetch_related("line_items")
```

---

## Trap 2: "Django signals are the right way to decouple business logic"

**What most people say:** "I use `post_save` signals to trigger side effects when a model changes — it keeps things decoupled."

**Correct answer:** Django signals appear to provide decoupling but introduce hidden coupling. Problems: (1) Signals fire whenever `save()` is called — including `bulk_create`, `update()`, `loaddata`, and during tests — often at the wrong times. (2) Signal handlers are invisible at the call site; debugging why an email was sent becomes a treasure hunt. (3) Signals do not participate in transactions by default (use `transaction.on_commit` if you need post-commit behavior). (4) They make testing harder because you have to disconnect signals or mock them. True decoupling is achieved with a service layer where the `ShipmentService.ship(order)` explicitly calls `email.send()` and `webhook.fire()`, making the behavior visible and testable.

---

## Trap 3: "FastAPI is async, so all my code automatically runs concurrently"

**What most people say:** "I switched from Flask to FastAPI so now my app is async."

**Correct answer:** Defining an endpoint as `async def` makes it run in the event loop. But if that endpoint calls a synchronous blocking function (e.g., `requests.get()`, a synchronous SQLAlchemy query, `time.sleep()`), the *entire event loop is blocked* — no other requests can be processed while that blocking call runs. For true concurrency, you must: (1) use `async`-native libraries (`httpx` not `requests`, `asyncpg`/async SQLAlchemy not sync SQLAlchemy), (2) run blocking code in a thread pool via `asyncio.to_thread()` or `loop.run_in_executor()`. FastAPI also accepts plain `def` (non-async) endpoints, which it automatically runs in a thread pool via `anyio`.

```python
from fastapi import FastAPI
import asyncio
import httpx

app = FastAPI()

# BAD: blocks the event loop — no other requests can run during requests.get()
@app.get("/bad")
async def bad_endpoint():
    import requests
    response = requests.get("https://api.example.com/data")  # BLOCKING!
    return response.json()

# GOOD: async HTTP client yields control back to the event loop
@app.get("/good")
async def good_endpoint():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return response.json()

# ALSO FINE: FastAPI runs sync endpoints in a thread pool automatically
@app.get("/sync-ok")
def sync_endpoint():
    import requests
    response = requests.get("https://api.example.com/data")
    return response.json()  # runs in thread pool, event loop not blocked
```

---

## Trap 4: "JWT tokens are secure because they are signed"

**What most people say:** "JWTs are secure — the signature verifies they haven't been tampered with."

**Correct answer:** Signing a JWT only proves the token was issued by the holder of the signing key and has not been modified. It does NOT mean the token is still valid for authorization purposes. Critical considerations: (1) JWTs cannot be invalidated before expiry without a server-side blocklist (defeating the "stateless" advantage). (2) The `alg: none` attack — some libraries accept "none" as the algorithm, bypassing signature verification. Always explicitly specify allowed algorithms. (3) Storing JWTs in `localStorage` exposes them to XSS attacks; `httpOnly` cookies are safer. (4) The payload is only base64-encoded, not encrypted — never put sensitive data in a JWT payload without using JWE (encrypted JWT).

```python
import jwt

# BAD: does not specify algorithms — vulnerable to alg:none attack
payload = jwt.decode(token, secret, options={"verify_signature": False})

# CORRECT: always specify the expected algorithm
payload = jwt.decode(token, secret, algorithms=["HS256"])

# For production: use RS256 (asymmetric) so verification services
# don't need the signing key:
private_key = open("private.pem").read()
public_key  = open("public.pem").read()
token = jwt.encode(payload, private_key, algorithm="RS256")
verified = jwt.decode(token, public_key, algorithms=["RS256"])
```

---

## Trap 5: "Django migrations are just database schema scripts"

**What most people say:** "Migrations are like SQL ALTER TABLE scripts — I can edit them freely."

**Correct answer:** Django migrations are Python files that represent the *state of all models at a point in time*. Django uses the migration dependency graph to reconstruct the model state at any point. If you edit a migration after it has been applied (especially in team environments), you break the migration state — Django's `migrate` may apply it twice, skip it, or produce incorrect state. Migrations also support data migrations (Python code to transform data), which have no SQL equivalent. Never edit applied migrations in a shared repo; always create new ones. Use `squashmigrations` to compress a long history into a single migration for a fresh checkout.

---

## Trap 6: "FastAPI's `BackgroundTasks` handles long-running background jobs"

**What most people say:** "I use `BackgroundTasks` to run my 5-minute data processing job asynchronously."

**Correct answer:** `BackgroundTasks` runs the task in the *same process* after the response is sent, in the same event loop. This means: (1) The task blocks the process if it's CPU-intensive. (2) If the server restarts, the task is lost — no persistence. (3) It does not scale across multiple workers. `BackgroundTasks` is appropriate for fast, non-critical tasks (sending an email confirmation, logging an event). For long-running or mission-critical background work, use Celery, RQ, or a proper task queue with a broker.

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

# APPROPRIATE USE: fast, fire-and-forget
def send_confirmation_email(email: str):
    # fast operation (~100ms)
    smtp_client.send(email, "Order confirmed")

@app.post("/orders")
async def create_order(background_tasks: BackgroundTasks):
    order = await db.create_order()
    background_tasks.add_task(send_confirmation_email, order.customer_email)
    return order   # response sent immediately; email queued to run after

# INAPPROPRIATE USE: 5-minute job
# Use Celery instead:
# @celery_app.task
# def process_large_dataset(dataset_id): ...
# process_large_dataset.delay(id)
```

---

## Trap 7: "WSGI and ASGI are interchangeable for async apps"

**What most people say:** "My Django app with async views works fine with Gunicorn WSGI."

**Correct answer:** Gunicorn (WSGI server) runs each request in a synchronous Python thread. `async def` views in Django need an ASGI server to actually be executed asynchronously. If you run an async Django view under WSGI (Gunicorn), Django's WSGI adapter runs the coroutine synchronously in a thread — it works, but you get zero async concurrency benefits. For true async Django, you need an ASGI server: Uvicorn, Daphne, or Gunicorn with `UvicornWorker`. The `asgi.py` and `wsgi.py` are two entry points Django provides; use ASGI for async workloads.

---

## Trap 8: "DRF's `ModelSerializer` is always the right choice"

**What most people say:** "I always use `ModelSerializer` — it's less code and auto-generates fields."

**Correct answer:** `ModelSerializer` is convenient but has pitfalls: (1) It auto-exposes all model fields unless you explicitly exclude them — a common source of information leakage. (2) Nested writable serializers on `ModelSerializer` require overriding `create()` and `update()` manually, negating the convenience. (3) For complex validation involving multiple fields, a plain `Serializer` gives more explicit control. (4) `ModelSerializer` is tightly coupled to the Django model — when the model changes, the serializer silently changes too. Always explicitly define `fields = [...]` (not `fields = "__all__"`) to avoid accidentally exposing sensitive data.

```python
# DANGEROUS: exposes ALL model fields including internal ones
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = "__all__"  # includes password hash, is_staff, last_login, etc.

# CORRECT: explicit fields
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "first_name", "last_name", "created_at"]
        read_only_fields = ["id", "created_at"]
```

---

## Trap 9: "Using `filter()` instead of `get()` is safer to avoid exceptions"

**What most people say:** "I always use `filter().first()` instead of `get()` so I don't get DoesNotExist errors."

**Correct answer:** Blindly using `filter().first()` returns `None` silently when an object doesn't exist — your code then fails with `AttributeError: 'NoneType' object has no attribute '...'` on the next line, which is harder to debug than a `DoesNotExist` at the lookup. Worse, `filter()` does not protect you from multiple matches the way `get()` does — `get()` raises `MultipleObjectsReturned` if the lookup is ambiguous. Use `get()` when you expect exactly one object and treat its absence as an error. Use `filter().first()` when zero results is a valid state. Use `get_object_or_404()` in views where absence should be a 404.

---

## Trap 10: "SQLAlchemy session is thread-safe"

**What most people say:** "I create one global SQLAlchemy session at startup and share it."

**Correct answer:** A SQLAlchemy `Session` is NOT thread-safe and NEVER safe to share between requests or threads. It maintains per-transaction state including an identity map (cache of loaded objects) that will be corrupted under concurrent access. The correct pattern is: one session per request, properly scoped to the request lifecycle. Use `sessionmaker()` to create a factory, and `scoped_session` for thread-local scoping in threaded servers, or yield-based FastAPI dependencies for async apps.

```python
# FastAPI: one session per request via Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

engine = create_async_engine("postgresql+asyncpg://localhost/mydb")
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session           # session created for this request
        # session automatically closed after request

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()
```
