# Chapter 06: Testing & pytest — Follow-Up Traps

## Trap 1: "I patch the module where it's defined"

**What most people say:** "I import `smtplib` in `myapp/email.py`, so I patch `smtplib.SMTP`."

**Correct answer:** You must patch the name in the namespace where it is *used*, not where it is *defined*. If `myapp/email.py` does `from smtplib import SMTP`, you must patch `myapp.email.SMTP`. If it does `import smtplib` and calls `smtplib.SMTP(...)`, you patch `myapp.email.smtplib.SMTP`. Patching `smtplib.SMTP` directly patches the source but the already-imported reference in `myapp.email` is unaffected.

```python
# email.py
from smtplib import SMTP

def send(msg):
    with SMTP("localhost") as s:
        s.sendmail(...)

# WRONG - patches the original, not the already-imported name
@patch("smtplib.SMTP")
def test_send_wrong(mock_smtp): ...

# CORRECT - patches the name in the module that uses it
@patch("myapp.email.SMTP")
def test_send_correct(mock_smtp): ...
```

---

## Trap 2: "Mock and MagicMock are basically the same thing"

**What most people say:** "MagicMock is just a more powerful Mock, so I always use MagicMock."

**Correct answer:** `MagicMock` pre-configures all magic/dunder methods (`__len__`, `__iter__`, `__enter__`, `__exit__`, etc.) to return sensible defaults. Using `MagicMock` everywhere can mask bugs: if your code accidentally iterates over something that should not be iterable, a plain `Mock` would raise `TypeError` (catching the bug), while `MagicMock` would silently return an empty iterator. Use `Mock` when you want strict duck-typing checks; use `MagicMock` when you explicitly need protocol support (context managers, iteration).

---

## Trap 3: "session-scoped fixtures are always better because they run once"

**What most people say:** "I'll make my database fixture session-scoped to speed up tests."

**Correct answer:** Session-scoped fixtures share state across all tests in the session. If any test mutates the shared object (e.g., writes records to the DB), subsequent tests inherit that dirty state, causing order-dependent failures. The correct approach is: session-scope the *expensive setup* (e.g., creating the schema / container), but use function-scope transactional rollback fixtures for per-test isolation. Django's `TestCase` uses this pattern with `transaction.atomic` per test.

```python
@pytest.fixture(scope="session")
def db_engine():
    engine = create_engine(TEST_DB_URL)
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)

@pytest.fixture
def db_session(db_engine):
    connection = db_engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session
    session.close()
    transaction.rollback()   # isolation per test
    connection.close()
```

---

## Trap 4: "100% line coverage means the code is well tested"

**What most people say:** "Our coverage is 100%, so we're good."

**Correct answer:** Line coverage only tells you every line was *executed* at least once. Branch coverage additionally checks that every Boolean branch (both True and False paths of every `if`/`while`/ternary) was exercised. You can have 100% line coverage while completely missing a branch.

```python
def discount(price, is_member):
    if is_member:          # line executed — True branch tested
        return price * 0.9
    return price           # line executed — but was is_member=False ever tested?
```

Use `pytest --cov --cov-branch` and aim for high branch coverage, not just line coverage.

---

## Trap 5: "assert_called_with checks that the mock was called at some point"

**What most people say:** "I use `mock.assert_called_with(args)` to verify it was called with those arguments."

**Correct answer:** `assert_called_with` checks only the **most recent** call. If your mock was called twice and you care about the first call, `assert_called_with` will check only the last one and may give a false positive. Use `assert_any_call` if any call with those args suffices, or `assert_has_calls([call(...), call(...)])` to check an ordered sequence of calls. Also note: misspelling assertion methods like `assert_called_once` (without `_with`) does nothing on a `Mock` — it creates a new attribute instead of raising.

```python
m = Mock()
m("first")
m("second")

m.assert_called_with("first")   # PASSES — but it shouldn't! Last call was "second"
m.assert_called_with("second")  # passes correctly
m.assert_any_call("first")      # correct way to check "first" was called at some point
```

---

## Trap 6: "side_effect replaces return_value"

**What most people say:** "If I set `side_effect`, I can't also use `return_value`."

**Correct answer:** When `side_effect` is set to a callable, that callable's return value is used (not `return_value`). When `side_effect` is an iterable, each call pops the next item (exceptions are raised, other values are returned). When `side_effect` is an exception class or instance, it is raised. If `side_effect` is set to `None`, the mock reverts to using `return_value`. You can use `side_effect` as a callable that conditionally raises or returns.

```python
from unittest.mock import Mock

def smart_side_effect(x):
    if x < 0:
        raise ValueError("negative")
    return x * 2

m = Mock(side_effect=smart_side_effect)
m(5)    # returns 10
m(-1)   # raises ValueError
```

---

## Trap 7: "pytest fixtures are just setup functions"

**What most people say:** "A fixture runs before the test, like setUp."

**Correct answer:** pytest fixtures are dependency-injected, composable, and can also perform teardown via `yield`. The code after `yield` runs as teardown. Multiple fixtures are composed by listing them as function parameters — pytest builds a dependency graph and resolves the correct order. They also have scope, parametrization (`params`), and can be overridden in nested `conftest.py` files. This is far more flexible than `setUp`/`tearDown` which are coupled to class hierarchy.

```python
@pytest.fixture
def temp_file(tmp_path):
    f = tmp_path / "data.txt"
    f.write_text("hello")
    yield f                  # test runs here with the file available
    # teardown: tmp_path handles cleanup automatically, but you could add explicit steps
```

---

## Trap 8: "hypothesis always finds bugs if you run it once"

**What most people say:** "I ran the property-based test and it passed, so my code is correct."

**Correct answer:** Hypothesis uses a database to remember previous failing examples, but on a fresh run it generates a finite number of examples (default ~100 attempts). It may not explore the specific edge case that triggers a bug on any given run. Additionally, Hypothesis shrinks failing examples to the smallest reproducing case — this is a huge advantage, but means the first run may *not* find a latent bug. To increase confidence, use `@settings(max_examples=1000)`, commit the Hypothesis example database to version control so previously found failures are always replayed, and use `@example(...)` to pin known edge cases.

---

## Trap 9: "conftest.py fixtures are global across the entire project"

**What most people say:** "I put my fixtures in conftest.py and they're available everywhere."

**Correct answer:** A `conftest.py` file makes fixtures available to tests in its **directory and all subdirectories**. A `conftest.py` in `tests/unit/` does NOT make fixtures available to `tests/integration/`. The root-level `conftest.py` (usually next to `pyproject.toml`) makes fixtures globally available. You can have multiple `conftest.py` files at different levels, and a fixture with the same name in a lower-level `conftest.py` will shadow (override) one from a higher level for tests in that subtree.

---

## Trap 10: "monkeypatch.setattr and unittest.mock.patch do the same thing"

**What most people say:** "I can use either one to mock attributes."

**Correct answer:** Both patch attributes on objects, but they differ in important ways. `monkeypatch` is pytest-native, automatically undone after the test, and works on any object attribute (including environment variables, `sys.path`, dict items). `unittest.mock.patch` returns a `Mock` object by default and is the standard choice when you need to assert on call history. `monkeypatch` does not give you a mock with `assert_called_with` — it just swaps the real object for whatever you provide. They are complementary: use `monkeypatch` for simple swaps (config values, env vars), `patch` when you need a full mock with call tracking.

---

## Trap 11: "xfail means the test is expected to fail forever"

**What most people say:** "I mark a test xfail when I know it will always fail."

**Correct answer:** `@pytest.mark.xfail` marks a test as *expected to fail*, not permanently broken. If the test fails, pytest reports it as `xfail` (expected failure) — this is a success in CI. If the test *passes*, pytest reports it as `xpass` (unexpected pass). By default `xpass` still results in the suite passing — which is dangerous because it means the code you thought was broken now works, but nobody noticed. Use `@pytest.mark.xfail(strict=True)` so that an unexpected pass causes the suite to **fail**, forcing the team to remove the xfail marker and acknowledge the fix.

---

## Trap 12: "spec prevents all misuse of a mock"

**What most people say:** "I added `spec=MyClass` so my mock is fully safe."

**Correct answer:** `spec` prevents you from *accessing attributes that don't exist* on the spec object, which catches typos in method names. But it does **not** validate method signatures — you can call `mock.my_method(wrong_number_of_args)` and `spec` won't complain. For signature checking, use `autospec=True` in `patch`, which wraps each method with the real signature. However, `autospec` can break on descriptors, class methods, and `__init__` in subtle ways, so test it before relying on it blindly.

```python
from unittest.mock import create_autospec

class Service:
    def process(self, item, dry_run=False):
        ...

mock_svc = create_autospec(Service)
mock_svc.process("x", True, "extra")  # raises TypeError — correct arg count enforced
```
