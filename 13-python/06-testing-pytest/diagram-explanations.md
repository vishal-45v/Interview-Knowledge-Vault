# Chapter 06: Testing & pytest — Diagram Explanations

## Diagram 1: pytest Test Discovery Flow

```
pytest invoked
      │
      ▼
┌─────────────────────────────────────────┐
│  Read pytest.ini / pyproject.toml /     │
│  setup.cfg for testpaths, rootdir       │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│  Collect conftest.py files              │
│  (root → current dir, depth-first)      │
│  Register fixtures, plugins, hooks      │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│  Walk testpaths looking for             │
│  files matching test_*.py or *_test.py  │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│  Within each file collect:              │
│  ├── test_* functions at module level   │
│  ├── Test* classes (no __init__)        │
│  │    └── test_* methods inside         │
│  └── @pytest.fixture functions (skip)  │
└───────────────────┬─────────────────────┘
                    │
                    ▼
            ┌───────────────┐
            │  Test Node ID │
            │  e.g.:        │
            │  tests/unit/  │
            │  test_auth.py │
            │  ::TestLogin  │
            │  ::test_valid │
            └───────────────┘
```

**Key rules:**
- Files must match `test_*.py` or `*_test.py` (configurable via `python_files`)
- Classes must match `Test*` with no `__init__` method
- Functions/methods must match `test_*` (configurable via `python_functions`)
- `conftest.py` is loaded automatically, never imported explicitly

---

## Diagram 2: Fixture Scope Lifecycle

```
TEST RUN BEGINS
│
├─ session scope setup ──────────────────────────────────────────┐
│   (runs once at start)                                         │
│                                                                │
│  ┌── module: tests/test_auth.py ─────────────────────────┐    │
│  │  module scope setup                                    │    │
│  │                                                        │    │
│  │  ┌── class: TestLogin ──────────────────────────┐     │    │
│  │  │  class scope setup                           │     │    │
│  │  │                                              │     │    │
│  │  │  function scope setup ──► test_valid_login   │     │    │
│  │  │  function scope teardown                     │     │    │
│  │  │                                              │     │    │
│  │  │  function scope setup ──► test_wrong_pass    │     │    │
│  │  │  function scope teardown                     │     │    │
│  │  │                                              │     │    │
│  │  │  class scope teardown                        │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  │                                                        │    │
│  │  function scope setup ──► test_reset_password          │    │
│  │  function scope teardown                               │    │
│  │                                                        │    │
│  │  module scope teardown                                 │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                │
├─ (more modules...)                                             │
│                                                                │
└─ session scope teardown ───────────────────────────────────────┘
TEST RUN ENDS
```

**Rule:** A fixture can only depend on fixtures of the same or wider scope. A `function`-scoped fixture can use a `session` fixture. A `session` fixture CANNOT use a `function` fixture (it would be torn down too early).

---

## Diagram 3: `unittest.mock.patch` — "Where Used" Rule

```
FILESYSTEM LAYOUT
─────────────────
stripe/
  __init__.py
    └── class Charge: ...        ← DEFINED HERE

myapp/
  payments.py
    ├── from stripe import Charge  ← NAME BOUND HERE
    └── def charge_customer():
            Charge.create(...)     ← USED HERE


WHAT HAPPENS AT IMPORT TIME
─────────────────────────────

stripe module:          myapp.payments module:
┌──────────────┐        ┌──────────────────────┐
│ Charge ──────┼──────► │ Charge ──────────────┼──► real Charge class
│ (original)   │        │ (local reference)    │
└──────────────┘        └──────────────────────┘


WRONG PATCH: patch("stripe.Charge")
─────────────────────────────────────

stripe module:          myapp.payments module:
┌──────────────┐        ┌──────────────────────┐
│ Charge ──────┼──► 🔄  │ Charge ──────────────┼──► STILL real Charge
│ (now a Mock) │        │ (NOT updated)        │
└──────────────┘        └──────────────────────┘
                        charge_customer() uses REAL Charge ← BUG


CORRECT PATCH: patch("myapp.payments.Charge")
──────────────────────────────────────────────

stripe module:          myapp.payments module:
┌──────────────┐        ┌──────────────────────┐
│ Charge       │        │ Charge ──────────────┼──► Mock object  ✓
│ (unchanged)  │        │ (replaced by patch)  │
└──────────────┘        └──────────────────────┘
                        charge_customer() uses Mock ← CORRECT
```

---

## Diagram 4: Test Double Taxonomy

```
                    TEST DOUBLES
                         │
         ┌───────────────┼───────────────┐
         │               │               │
      INDIRECT        INDIRECT        DIRECT
      INPUT           OUTPUT          OUTPUT
      (stubs,         (mocks,         (fakes,
       dummies)        spies)          spies)
         │               │
         ▼               ▼
   ┌──────────┐    ┌──────────┐
   │  STUB    │    │  MOCK    │
   │          │    │          │
   │ Returns  │    │Pre-prog- │
   │ canned   │    │rammed    │
   │ data     │    │expects   │
   │          │    │verifies  │
   │ No call  │    │calls at  │
   │ verif.   │    │end       │
   └──────────┘    └──────────┘

   ┌──────────┐    ┌──────────┐
   │  FAKE    │    │  SPY     │
   │          │    │          │
   │ Working  │    │ Wraps    │
   │ impl.    │    │ real     │
   │ with     │    │ object,  │
   │ shortcuts│    │ records  │
   │ (in-mem  │    │ all      │
   │  DB etc) │    │ calls    │
   └──────────┘    └──────────┘

VERIFICATION?   REAL BEHAVIOR?
   Stub: NO         Stub: NO (canned)
   Fake: NO         Fake: YES (real algo, fake infra)
   Spy:  YES        Spy:  YES (real algo runs)
   Mock: YES        Mock: NO  (pre-programmed)
```

---

## Diagram 5: Coverage — Line vs Branch

```
SOURCE CODE                     LINE COV    BRANCH COV
──────────────────────────────────────────────────────

def process(items, flag=False):
    result = []                  ← executed    N/A

    if items:                    ← executed    ┬ True:  tested ✓
        for i in items:          ← executed    └ False: MISSING ✗
            result.append(i)     ← executed

    if flag:                     ← executed    ┬ True:  MISSING ✗
        result = sorted(result)  ← NOT EXEC    └ False: tested ✓

    return result                ← executed    N/A

──────────────────────────────────────────────────────
Test: process([3, 1, 2])

Line coverage:  5/6 lines = 83%   (sorted line missed)
Branch coverage: 2/4 branches = 50%

With --cov-branch:
  Missing branches highlighted:
  ► items=[] (falsy path of first if)
  ► flag=True (truthy path of second if)
```

---

## Diagram 6: `conftest.py` Fixture Resolution

```
project/
├── conftest.py              (A) session-wide fixtures
├── pyproject.toml
└── tests/
    ├── conftest.py          (B) test-wide fixtures
    ├── unit/
    │   ├── conftest.py      (C) unit-test fixtures
    │   └── test_service.py  ← runs here
    └── integration/
        ├── conftest.py      (D) integration fixtures
        └── test_api.py      ← runs here

FIXTURE LOOKUP for test_service.py:
────────────────────────────────────
test_service.py asks for fixture "db"

Step 1: Look in test_service.py itself
Step 2: Look in tests/unit/conftest.py  (C)  ← found here? USE IT
Step 3: Look in tests/conftest.py       (B)  ← if not in C, check B
Step 4: Look in project/conftest.py     (A)  ← if not in B, check A
Step 5: pytest built-ins (tmp_path, capsys, monkeypatch, ...)
Step 6: Installed plugins

OVERRIDE EXAMPLE:
  conftest.py (A) defines: @pytest.fixture def db() → real PostgreSQL
  tests/unit/conftest.py (C) defines: @pytest.fixture def db() → SQLite

  test_service.py gets SQLite version (C wins — closest wins)
  test_api.py gets PostgreSQL version (C not in its path; B→A chain used)
```
