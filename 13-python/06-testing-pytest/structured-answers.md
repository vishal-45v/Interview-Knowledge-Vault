# Chapter 06: Testing & pytest — Structured Answers

## Q1: Explain pytest fixtures, their scopes, and how `conftest.py` works

**Answer:**

A pytest fixture is a function decorated with `@pytest.fixture` that provides a test with some resource — a database connection, a temporary file, a configured object, etc. Tests declare their dependency on a fixture simply by listing its name as a function parameter. pytest injects the fixture's return (or yielded) value automatically.

**Fixture scope** controls how long the fixture lives and how many times it is called:

| Scope | Lifetime | Re-use |
|---|---|---|
| `function` (default) | Created/destroyed per test | One per test |
| `class` | One per test class | Shared within a class |
| `module` | One per `.py` file | Shared within a module |
| `session` | One per test run | Shared across the entire run |

```python
import pytest
import sqlalchemy as sa

# Session-scoped: expensive setup runs once for the whole test run
@pytest.fixture(scope="session")
def db_engine():
    engine = sa.create_engine("postgresql://localhost/testdb")
    sa.text("SELECT 1")  # verify connection
    yield engine
    engine.dispose()

# Function-scoped: each test gets an isolated, rolled-back transaction
@pytest.fixture
def session(db_engine):
    conn = db_engine.connect()
    trans = conn.begin()
    sess = sa.orm.Session(bind=conn)
    yield sess
    sess.close()
    trans.rollback()
    conn.close()
```

**`conftest.py`** is a special pytest file that is automatically loaded — no import needed. Fixtures defined in a `conftest.py` are available to all tests in that directory and all subdirectories. This makes `conftest.py` the natural home for shared fixtures. You can have multiple `conftest.py` files at different directory levels; a fixture in a lower-level file shadows one with the same name from a higher level for tests in that subtree.

**Yield fixtures** allow teardown:
```python
@pytest.fixture
def redis_client():
    client = redis.Redis(host="localhost", port=6379, db=15)
    yield client          # test body runs here
    client.flushdb()      # teardown runs after test
    client.close()
```

---

## Q2: How does `unittest.mock.patch` work and what is the "patch where used" rule?

**Answer:**

`patch` temporarily replaces an attribute on a module or class for the duration of a test, then restores the original. It can be used as a decorator, context manager, or called manually.

**The "patch where used" rule:** When a module does `from foo import Bar`, Python binds the name `Bar` in that module's namespace. Patching `foo.Bar` changes the source but the already-imported binding in the target module is unchanged. You must patch `target_module.Bar` — where the name is *used*.

```python
# payments.py
from stripe import Charge   # binds the name 'Charge' in payments namespace

def charge_customer(amount):
    return Charge.create(amount=amount, currency="usd")
```

```python
# test_payments.py
from unittest.mock import patch, MagicMock

# WRONG: patches stripe.Charge, but payments.Charge is already bound
@patch("stripe.Charge")
def test_wrong(mock_charge):
    from payments import charge_customer
    charge_customer(100)
    # mock_charge.create may not have been called

# CORRECT: patches the name as seen inside the payments module
@patch("payments.Charge")
def test_correct(mock_charge):
    from payments import charge_customer
    charge_customer(100)
    mock_charge.create.assert_called_once_with(amount=100, currency="usd")
```

**`patch.object`** patches an attribute on a specific *object* (class or instance), which is safer because you reference the real object rather than a string path:

```python
import payments
from unittest.mock import patch

with patch.object(payments.Charge, "create", return_value={"id": "ch_123"}) as mock_create:
    result = payments.charge_customer(100)
    mock_create.assert_called_once()
```

---

## Q3: Explain `side_effect`, `return_value`, and `spec` on Mock objects

**Answer:**

**`return_value`** is what the mock returns when called:
```python
m = Mock(return_value=42)
assert m() == 42
assert m("anything") == 42  # always returns 42 regardless of args
```

**`side_effect`** overrides `return_value` and has three modes:

1. **Exception class/instance** — raised when mock is called:
```python
m = Mock(side_effect=ConnectionError("timeout"))
m()   # raises ConnectionError
```

2. **Iterable** — each call returns the next value; exceptions in the iterable are raised:
```python
m = Mock(side_effect=[1, 2, ValueError("done")])
m()   # returns 1
m()   # returns 2
m()   # raises ValueError
```

3. **Callable** — called with same args as the mock, its return value is used:
```python
def fake_get(url, **kwargs):
    if "users" in url:
        return Mock(status_code=200, json=lambda: [{"id": 1}])
    return Mock(status_code=404)

with patch("requests.get", side_effect=fake_get):
    resp = requests.get("https://api.example.com/users")
    assert resp.status_code == 200
```

**`spec`** restricts the mock to only attributes that exist on the given class/object, catching attribute typos:
```python
class UserService:
    def get_user(self, user_id): ...
    def create_user(self, data): ...

mock_svc = Mock(spec=UserService)
mock_svc.get_user(1)        # OK
mock_svc.dlete_user(1)      # AttributeError — typo caught at test time, not prod
```

Use `autospec=True` with `patch` to also enforce method signatures.

---

## Q4: How does `@pytest.mark.parametrize` work, and when is `hypothesis` better?

**Answer:**

`@pytest.mark.parametrize` runs the test function once per parameter set, each as a separate test with its own pass/fail status:

```python
import pytest
from myapp.math import is_prime

@pytest.mark.parametrize("n, expected", [
    (2, True),
    (3, True),
    (4, False),
    (17, True),
    (1, False),
    (0, False),
    (-5, False),
])
def test_is_prime(n, expected):
    assert is_prime(n) == expected
```

Each combination appears as a separate test in the output (e.g., `test_is_prime[2-True]`), making failures easy to locate. You can also parametrize fixtures:

```python
@pytest.fixture(params=["sqlite", "postgresql"])
def database_url(request):
    return DB_URLS[request.param]
```

**When hypothesis is better:** `parametrize` requires you to enumerate examples manually. For functions where the input space is vast or the interesting edge cases are not obvious, `hypothesis` explores the space automatically:

```python
from hypothesis import given, strategies as st

@given(st.integers(), st.integers())
def test_add_commutative(a, b):
    assert add(a, b) == add(b, a)

@given(st.text())
def test_serialize_roundtrip(s):
    assert deserialize(serialize(s)) == s
```

`hypothesis` generates random inputs, finds a failing case, then **shrinks** it to the minimal reproducing example (e.g., reduces a 1000-character string to the 2-character string that actually triggers the bug). It also has a database that replays previously found failures on every run.

Use `parametrize` for known, documented edge cases. Use `hypothesis` for invariant/property testing where the full input space matters.

---

## Q5: Explain test doubles — stub, fake, spy, mock — with examples

**Answer:**

Test doubles are objects that stand in for real dependencies during testing. The taxonomy (from Gerard Meszaros' *xUnit Patterns*):

**Stub** — returns canned responses, no behavior verification:
```python
class StubUserRepo:
    def get_by_id(self, user_id):
        return User(id=user_id, name="Alice", email="alice@example.com")

def test_send_welcome_email():
    repo = StubUserRepo()
    service = EmailService(user_repo=repo)
    service.send_welcome(user_id=1)
    # We're testing EmailService's logic, not repo. Stub provides predictable data.
```

**Fake** — a working implementation with shortcuts (in-memory instead of real DB):
```python
class FakeUserRepo:
    def __init__(self):
        self._store = {}

    def save(self, user):
        self._store[user.id] = user

    def get_by_id(self, user_id):
        return self._store.get(user_id)
```

**Spy** — a real object that also records interactions for later assertion:
```python
# pytest-mock's mocker.spy wraps the real method
def test_discount_applied(mocker):
    service = PricingService()
    spy = mocker.spy(service, "apply_discount")
    service.calculate_final_price(100, is_member=True)
    spy.assert_called_once_with(100)  # real method ran AND we verified it was called
```

**Mock** — pre-programmed with expectations; the test verifies those expectations were met:
```python
from unittest.mock import Mock

def test_charge_called_on_purchase():
    mock_payment = Mock()
    order_service = OrderService(payment_gateway=mock_payment)
    order_service.complete_purchase(order_id=42, amount=99.99)
    mock_payment.charge.assert_called_once_with(amount=99.99, currency="USD")
```

`unittest.mock.Mock` blurs the line between stub, spy, and mock — it can do all three. The important thing is understanding *what* you are testing: **state** (check return value or object state after) or **interactions** (check what calls were made and with what arguments).

---

## Q6: How do you measure and enforce coverage in a CI pipeline?

**Answer:**

Install `pytest-cov`:
```
pip install pytest-cov
```

Run with coverage:
```bash
# Line coverage
pytest --cov=myapp --cov-report=term-missing

# Branch coverage (recommended)
pytest --cov=myapp --cov-branch --cov-report=term-missing --cov-report=html

# Fail CI if coverage drops below threshold
pytest --cov=myapp --cov-branch --cov-fail-under=85
```

**`pyproject.toml` configuration:**
```toml
[tool.pytest.ini_options]
addopts = "--cov=myapp --cov-branch --cov-fail-under=85"

[tool.coverage.run]
source = ["myapp"]
branch = true
omit = ["myapp/migrations/*", "myapp/settings/*", "*/__init__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

**Branch coverage example:**
```python
def process(items):
    if not items:           # branch 1a: items is empty
        return []           # only this line executes in the empty case
    return [transform(i) for i in items]  # branch 1b: items is non-empty
```

With only `test_process([1, 2, 3])`, line coverage is 100% (all lines execute). Branch coverage shows the `if not items` True branch was never taken. `pytest --cov-branch` catches this.

**`pragma: no cover`** tells coverage to ignore a specific line:
```python
if __name__ == "__main__":  # pragma: no cover
    main()
```
