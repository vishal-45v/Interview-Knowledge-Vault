# Chapter 06: Testing & pytest — Analogy Explanations

## Analogy 1: Fixtures as a Restaurant Mise en Place

**The Story:**
Before a busy dinner service, a chef prepares their *mise en place* — everything in its place. Sauce is reduced, vegetables are chopped, proteins are portioned. When an order comes in, the chef doesn't start from scratch. They grab what they need from the prepared station, cook the dish, and the station is cleaned between orders (or shared for the whole service if it's a long-lasting prep like a stock).

**The Python Connection:**
A pytest fixture is the mise en place for a test. Function-scope fixtures are like ingredients prepped per plate (reset per test). Module-scope is like a sauce made once per cooking session. Session-scope is like a stock that simmers all day. The `yield` statement is the moment the food leaves the kitchen — setup happened before, teardown (cleaning the station) happens after.

```python
@pytest.fixture(scope="function")   # "per plate" — fresh each test
def user():
    return User(name="Alice", email="alice@test.com")

@pytest.fixture(scope="session")    # "stock simmering all day" — made once
def db_engine():
    engine = create_engine(TEST_URL)
    yield engine
    engine.dispose()   # clean up at end of the day

def test_user_can_login(user, db_engine):
    # mise en place is ready — just cook the dish
    result = login_service.authenticate(user.email, "password123")
    assert result.success
```

---

## Analogy 2: Test Doubles as Movie Stunt Doubles

**The Story:**
A film features an expensive A-list actor. For a car chase sequence, they use a stunt double who looks like the actor but does the dangerous work. For a scene that just needs a silhouette in the background, they use an extra (mannequin). For a scene where the actor coaches someone else, the real actor is present but their movements are secretly tracked by a motion-capture suit. For a key dramatic scene, the actor's own reactions are scripted in advance by the director.

**The Python Connection:**
- **Stub**: The mannequin. Returns hardcoded data, no questions asked.
- **Fake**: A stunt double with real skills. It works like the real thing but shortcuts the dangerous/expensive parts (in-memory DB instead of PostgreSQL).
- **Spy**: The motion-capture suit on the real actor. Lets the real code run but records every move for later review.
- **Mock**: The actor with scripted directions. Pre-programmed with expected calls; if the scene doesn't match the script, the director (test) calls it out.

```python
# Stub — mannequin, always says the same thing
class StubRepo:
    def find(self, id): return {"id": id, "name": "Alice"}

# Fake — stunt double with real skills, but uses a shortcut
class FakeRepo:
    _db = {}
    def save(self, obj): self._db[obj["id"]] = obj
    def find(self, id): return self._db.get(id)

# Spy — real method runs, but calls are tracked
spy = mocker.spy(real_service, "process")

# Mock — pre-programmed, verifies the script was followed
mock_repo = Mock()
mock_repo.save.assert_called_once_with({"id": 1})
```

---

## Analogy 3: `monkeypatch` as a Theater Set Dresser

**The Story:**
A theater production is set in 1920s New York but is performed in London in 2024. The set dresser replaces modern objects with period pieces: the smartphone on the desk becomes a candlestick phone, the laptop becomes a typewriter. After each performance, the set is restored to the default state for the next show (or for the crew who uses the space normally).

**The Python Connection:**
`monkeypatch` is the set dresser of your test. It temporarily swaps real objects (environment variables, class attributes, module-level values, system paths) with test-appropriate replacements. After each test, pytest automatically restores the originals — no manual cleanup needed.

```python
import os
from myapp.config import get_database_url

def test_uses_test_database(monkeypatch):
    # Set dresser replaces the environment variable for this "performance"
    monkeypatch.setenv("DATABASE_URL", "sqlite:///test.db")
    url = get_database_url()
    assert url == "sqlite:///test.db"
    # After the test, DATABASE_URL is automatically restored

def test_request_timeout(monkeypatch):
    import myapp.http_client as client
    # Swap the real timeout for a test-friendly one
    monkeypatch.setattr(client, "DEFAULT_TIMEOUT", 0.001)
    with pytest.raises(TimeoutError):
        client.fetch("https://slow-server.example.com")
```

---

## Analogy 4: Coverage as a Highway Inspection Route

**The Story:**
A transportation inspector must verify every road in a county is in working order. Line coverage is like checking that every road segment was *driven on* at least once. But what about the entrance ramps and exit ramps (branches)? A road that looks fine in normal traffic might have a broken off-ramp that leads to a cliff. Branch coverage requires driving both onto and off of every junction.

**The Python Connection:**
Line coverage tells you every line of code was executed. Branch coverage tells you every decision point was tested in both directions — every `if` was tested when True AND when False.

```python
def classify_age(age):
    if age < 0:                    # branch A: age < 0, branch B: age >= 0
        raise ValueError("negative")
    if age < 18:                   # branch C: < 18, branch D: >= 18
        return "minor"
    if age >= 65:                  # branch E: >= 65, branch F: 18-64
        return "senior"
    return "adult"
```

Testing only `classify_age(25)` gives 100% line coverage (all lines execute) but misses branches A, C, and E. Run `pytest --cov-branch` to find the unchecked off-ramps.

---

## Analogy 5: `hypothesis` as a Fuzzing Quality Inspector

**The Story:**
A factory makes padlocks. A quality inspector could test ten specific key-cut combinations and declare the lock secure. Or they could use an automated machine that tries millions of key combinations, finds the one that opens the lock, then reports back: "Key number 4,847,293 opens it" — and also reports the *simplest possible* version of that key shape.

**The Python Connection:**
`@pytest.mark.parametrize` is the inspector testing ten specific keys. `hypothesis` is the automated fuzzing machine. It generates hundreds of inputs, finds a failing case, then *shrinks* it to the simplest possible failing example.

```python
from hypothesis import given, strategies as st, settings

# Property: serializing then deserializing should return the original
@given(st.dictionaries(st.text(), st.integers()))
@settings(max_examples=500)
def test_json_roundtrip(data):
    import json
    assert json.loads(json.dumps(data)) == data

# If json.dumps has a bug with a specific dict key containing a null byte,
# hypothesis finds {"\\x00": 0} as the minimal failing case,
# not the original 50-key dict that first triggered it.
```

---

## Analogy 6: Fixture Scope as Office Lease Terms

**The Story:**
A company needs office space. They could rent a hotel meeting room per meeting (function scope — expensive setup, total isolation). They could rent a floor for a team's quarter-long project (module/class scope — shared by one group). They could lease an entire building for the company's annual operations (session scope — always available, but the mess accumulates).

**The Python Connection:**
The shorter the lease, the more isolation you get but the more setup cost you pay. The longer the lease, the faster things run but the more careful you must be about leaving the space clean for the next occupant (test).

```python
# Hotel meeting room — fresh for every meeting (test)
@pytest.fixture(scope="function")
def fresh_cache():
    return {}

# Floor lease — shared by all tests in this module
@pytest.fixture(scope="module")
def loaded_config():
    return load_expensive_config_file("/etc/app/config.yaml")

# Building lease — once per test run, shared by everyone
@pytest.fixture(scope="session")
def db_container():
    container = start_postgres_docker_container()
    yield container
    container.stop()
```

The bug pattern: using session-scope for a mutable object, then test A writes data, test B reads it expecting empty state, and the tests become order-dependent. Always scope teardown correctly.

---

## Analogy 7: `patch` as a Telephone Switchboard Operator

**The Story:**
In early telephone networks, when you called "the bakery," an operator at a switchboard physically connected your line to the bakery's line. During a test, you want to call "the bakery" but route the call to a fake bakery that always says "yes, we have croissants." The operator (patch) intercepts the call, routes it to your fake, and when the call is done, reconnects the original wiring.

The critical detail: if you have a **direct line** to the bakery (a cached reference like `from bakery import take_order`), the switchboard cannot intercept it — you have bypassed the operator. You must intercept at the point where the number is *dialed*, not at the bakery's phone.

**The Python Connection:**
```python
# bakery.py
class Bakery:
    def take_order(self, item): ...

# checkout.py
from bakery import Bakery     # direct line — name 'Bakery' is cached here
bakery = Bakery()

def complete_sale(item):
    return bakery.take_order(item)
```

```python
# test_checkout.py
# Wrong: patches bakery.Bakery but checkout.py already has its own reference
with patch("bakery.Bakery") as mock:
    complete_sale("croissant")   # REAL Bakery.take_order is called!

# Correct: patches the reference AS SEEN from checkout.py
with patch("checkout.Bakery") as mock:
    complete_sale("croissant")   # mock's take_order is called
```

---

## Analogy 8: `capsys` as a Court Reporter

**The Story:**
In a courtroom, a court reporter silently transcribes everything spoken. At any point, the attorney can call for a transcript and review exactly what was said. The reporter captures both the witness's words (stdout) and any outbursts from the gallery (stderr). The transcript is available on demand, and after reading it, a fresh transcript starts.

**The Python Connection:**
`capsys` captures all output written to `sys.stdout` and `sys.stderr` during the test. You call `capsys.readouterr()` to get the transcript and clear it for the next check.

```python
def format_report(data):
    print(f"Total items: {len(data)}")
    print(f"Sum: {sum(data)}")
    if any(x < 0 for x in data):
        import sys
        print("WARNING: negative values detected", file=sys.stderr)

def test_format_report_output(capsys):
    format_report([1, -2, 3])

    captured = capsys.readouterr()     # court reporter delivers transcript
    assert "Total items: 3" in captured.out
    assert "Sum: 2" in captured.out
    assert "WARNING: negative values" in captured.err

    # readouterr() also clears the buffer — fresh transcript from here
    format_report([10, 20])
    captured2 = capsys.readouterr()
    assert "WARNING" not in captured2.err
```
