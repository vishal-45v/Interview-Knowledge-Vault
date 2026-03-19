# Chapter 09: Web Frameworks — Structured Answers

## Q1: Explain the N+1 query problem and its Django solution

**Answer:**

The N+1 problem occurs when code fetches N objects and then makes 1 additional query *per object* to fetch related data — resulting in N+1 total queries instead of 1 or 2.

**The broken code:**
```python
# views.py — returns 50 orders with customer names
def order_list(request):
    orders = Order.objects.all()[:50]   # Query 1: SELECT * FROM orders LIMIT 50
    data = []
    for order in orders:
        data.append({
            "id": order.id,
            "customer": order.customer.name,  # Query 2,3,4,...51: SELECT FROM customers WHERE id=?
            "item_count": order.line_items.count(),  # Query 52,53,...101: SELECT COUNT FROM line_items
        })
    return JsonResponse({"orders": data})
# Total: 101 queries for 50 orders!
```

**The fix with `select_related` and `prefetch_related`:**
```python
def order_list(request):
    orders = (
        Order.objects
        .select_related("customer")         # JOIN customers into same query
        .prefetch_related("line_items")     # separate query, Python-side join
        [:50]
    )
    # Total queries: 2 — regardless of how many orders

    data = []
    for order in orders:
        data.append({
            "id": order.id,
            "customer": order.customer.name,         # hits cache, no query
            "item_count": len(order.line_items.all()),  # hits prefetch cache
        })
    return JsonResponse({"orders": data})
```

**What SQL this generates:**
```sql
-- Query 1: JOIN orders with customers
SELECT orders.*, customers.*
FROM orders
INNER JOIN customers ON orders.customer_id = customers.id
LIMIT 50;

-- Query 2: fetch all line_items for these order IDs
SELECT * FROM line_items
WHERE order_id IN (1, 2, 3, ... 50);
```

**Detecting N+1 in production:**
- Django Debug Toolbar (development): shows query count per request
- `django-silk` or `django-querycount`: production-safe query profiling
- Log slow queries: `LOGGING` config with `django.db.backends` level DEBUG
- `nplusone` library: raises warnings when N+1 patterns are detected in tests

**`Prefetch` object for filtered prefetches:**
```python
from django.db.models import Prefetch

# Only prefetch active line items
active_items = LineItem.objects.filter(status="active")
orders = Order.objects.prefetch_related(
    Prefetch("line_items", queryset=active_items, to_attr="active_line_items")
)
# order.active_line_items → list of active items, no extra queries
```

---

## Q2: How does FastAPI dependency injection work for authentication?

**Answer:**

FastAPI's `Depends()` builds a dependency graph. Dependencies are functions (sync or async) that can themselves depend on other dependencies. FastAPI resolves the graph, calls each dependency once per request (or cached per request scope), and injects the results.

**Complete JWT auth setup:**
```python
from fastapi import FastAPI, Depends, HTTPException, Security, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()
security = HTTPBearer()

SECRET = "your-secret-key"
ALGORITHM = "HS256"

# Layer 1: Extract token from Authorization header
async def get_token(
    credentials: HTTPAuthorizationCredentials = Security(security)
) -> str:
    return credentials.credentials

# Layer 2: Decode and validate JWT
async def get_current_user_id(token: str = Depends(get_token)) -> int:
    try:
        payload = jwt.decode(token, SECRET, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return user_id
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Layer 3: Load user from database
async def get_current_user(
    user_id: int = Depends(get_current_user_id),
    db: AsyncSession = Depends(get_db),
) -> User:
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    return user

# Layer 4: Check permissions
async def require_admin(
    current_user: User = Depends(get_current_user),
) -> User:
    if not current_user.is_admin:
        raise HTTPException(status_code=403, detail="Admin required")
    return current_user

# Usage in routes
@app.get("/profile")
async def get_profile(user: User = Depends(get_current_user)):
    return {"id": user.id, "email": user.email}

@app.delete("/users/{user_id}")
async def delete_user(user_id: int, admin: User = Depends(require_admin)):
    ...
```

**Overriding dependencies in tests:**
```python
# test_api.py
from fastapi.testclient import TestClient
from myapp.main import app
from myapp.dependencies import get_current_user
from myapp.models import User

def test_get_profile():
    # Override the dependency with a fake user
    def fake_user():
        return User(id=1, email="test@example.com", is_admin=False)

    app.dependency_overrides[get_current_user] = fake_user

    client = TestClient(app)
    response = client.get("/profile")
    assert response.status_code == 200
    assert response.json()["email"] == "test@example.com"

    app.dependency_overrides.clear()  # cleanup
```

---

## Q3: Explain the Django REST Framework serializer validation lifecycle

**Answer:**

DRF validation runs in three stages when you call `serializer.is_valid()`:

**Stage 1: Field-level type validation** (each field's `to_internal_value`)
**Stage 2: Field-level custom validation** (`validate_<fieldname>` methods)
**Stage 3: Object-level validation** (`validate` method)

```python
from rest_framework import serializers
from django.contrib.auth.models import User

class UserRegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ["id", "email", "first_name", "last_name",
                  "password", "password_confirm"]
        read_only_fields = ["id"]
        extra_kwargs = {
            "email": {"required": True},
        }

    # Stage 2a: validate a single field
    def validate_email(self, value: str) -> str:
        value = value.lower().strip()
        if User.objects.filter(email=value).exists():
            raise serializers.ValidationError("Email already registered")
        return value

    # Stage 2b: validate another single field
    def validate_password(self, value: str) -> str:
        if value.isdigit():
            raise serializers.ValidationError("Password cannot be all numbers")
        return value

    # Stage 3: cross-field validation
    def validate(self, attrs: dict) -> dict:
        if attrs["password"] != attrs.pop("password_confirm"):
            raise serializers.ValidationError({
                "password_confirm": "Passwords do not match"
            })
        return attrs

    def create(self, validated_data: dict) -> User:
        # validated_data is clean at this point
        return User.objects.create_user(**validated_data)
```

**Using the serializer in a view:**
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class RegisterView(APIView):
    def post(self, request):
        serializer = UserRegistrationSerializer(data=request.data)
        if serializer.is_valid():
            user = serializer.save()
            return Response({"id": user.id}, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
        # serializer.errors = {
        #     "email": ["Email already registered"],
        #     "password_confirm": ["Passwords do not match"]
        # }
```

---

## Q4: Explain WSGI vs ASGI and the request/response lifecycle

**Answer:**

**WSGI (PEP 3333):** Synchronous, one-request-per-thread model. The server calls your application as a callable:
```python
def application(environ: dict, start_response) -> Iterable[bytes]:
    # environ contains HTTP request data (METHOD, PATH, headers, body)
    status = "200 OK"
    headers = [("Content-Type", "application/json")]
    start_response(status, headers)
    return [b'{"hello": "world"}']
```

**ASGI (PEP 3000-ish / Starlette spec):** Async, coroutine-based, handles HTTP, WebSockets, and lifespan events:
```python
async def application(scope: dict, receive, send) -> None:
    # scope: connection type, path, headers
    # receive: coroutine to read incoming events (request body, WS messages)
    # send: coroutine to send outgoing events (response start, body)
    if scope["type"] == "http":
        event = await receive()              # read request body
        await send({
            "type": "http.response.start",
            "status": 200,
            "headers": [[b"content-type", b"application/json"]],
        })
        await send({
            "type": "http.response.body",
            "body": b'{"hello": "world"}',
        })
```

**FastAPI request lifecycle:**
```
HTTP Request arrives at Uvicorn
         │
         ▼
    ASGI scope, receive, send passed to Starlette (FastAPI base)
         │
         ▼
    Middleware stack (outermost → innermost)
    ┌── CORS middleware
    ├── Exception handler middleware
    └── (custom middleware)
         │
         ▼
    Router: match path to endpoint
         │
         ▼
    Dependency injection: resolve Depends() graph
         │
         ▼
    Pydantic: validate request body / query params
         │
         ▼
    Your endpoint function runs
         │
         ▼
    Pydantic: serialize response model
         │
         ▼
    Middleware stack (innermost → outermost) — response goes back up
         │
         ▼
    BackgroundTasks run (after response sent)
```

---

## Q5: How do you implement Alembic migrations with SQLAlchemy?

**Answer:**

Alembic tracks database schema migrations as Python files with `upgrade()` and `downgrade()` functions.

**Setup:**
```bash
pip install alembic sqlalchemy
alembic init alembic/   # creates alembic/ directory and alembic.ini
```

**`alembic/env.py` configuration:**
```python
from myapp.models import Base
from myapp.config import DATABASE_URL

config.set_main_option("sqlalchemy.url", DATABASE_URL)
target_metadata = Base.metadata   # points to your models
```

**Define models:**
```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Integer, DateTime
import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime, default=datetime.datetime.utcnow
    )
```

**Workflow:**
```bash
# Generate migration by comparing models to current DB schema
alembic revision --autogenerate -m "add users table"

# Review the generated migration (ALWAYS review before applying!)
cat alembic/versions/xxxx_add_users_table.py

# Apply to database
alembic upgrade head

# Check current version
alembic current

# Roll back one migration
alembic downgrade -1

# Roll back to a specific revision
alembic downgrade abc123

# Show migration history
alembic history --verbose
```

**What autogenerate detects:**
- Added/removed tables and columns
- Changed column types, nullable, server_default
- Added/removed indexes and unique constraints
- Added/removed foreign keys

**What autogenerate DOES NOT detect:**
- Changes to stored procedures, views, triggers
- Renamed tables or columns (shows as drop + add)
- Changes to enum values in some databases
- `CHECK` constraints defined outside SQLAlchemy ORM

**Data migration example:**
```python
# alembic/versions/xxxx_backfill_display_name.py
def upgrade():
    # Schema change
    op.add_column("users", sa.Column("display_name", sa.String(255)))

    # Data migration — run in same transaction
    connection = op.get_bind()
    connection.execute(
        text("UPDATE users SET display_name = first_name || ' ' || last_name")
    )

    # Make it non-nullable after backfill
    op.alter_column("users", "display_name", nullable=False)

def downgrade():
    op.drop_column("users", "display_name")
```
