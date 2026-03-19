# Chapter 10: Python Internals — Diagram Explanations

## Diagram 1: CPython Object Memory Layout (PyObject)

```
Every Python object in memory (simplified C structure):

┌─────────────────────────────────────────────────────────┐
│  PyObject (base of ALL Python objects)                  │
│                                                         │
│  ob_refcnt: Py_ssize_t  ← reference count              │
│  ob_type:   *PyTypeObject ← pointer to type object      │
└─────────────────────────────────────────────────────────┘

Extended for Python int (PyLongObject):
┌─────────────────────────────────────────────────────────┐
│  ob_refcnt: 3                                           │
│  ob_type:   → <class 'int'>                             │
│  ob_digit:  [42]       ← actual integer digits          │
└─────────────────────────────────────────────────────────┘

Extended for Python list (PyListObject):
┌─────────────────────────────────────────────────────────┐
│  ob_refcnt: 1                                           │
│  ob_type:   → <class 'list'>                            │
│  ob_size:   3           ← current number of items       │
│  ob_alloc:  4           ← allocated capacity            │
│  ob_item:   ────────────────────────────────────────────┼──►
└─────────────────────────────────────────────────────────┘
                                                           │
                          ┌────────────────────────────────┘
                          ▼
                    [*PyObject, *PyObject, *PyObject, NULL]
                          │          │          │
                    (pointer   (pointer    (pointer
                     to int 1)  to int 2)   to str "x")
```

---

## Diagram 2: Reference Counting Lifecycle

```
STEP 1: a = [1, 2, 3]
────────────────────────────────────────
  Name table:              Heap:
  ┌─────────┐             ┌──────────────────┐
  │ a ──────┼────────────►│ list [1,2,3]     │
  └─────────┘             │ ob_refcnt = 1    │
                          └──────────────────┘

STEP 2: b = a
────────────────────────────────────────
  Name table:              Heap:
  ┌─────────┐             ┌──────────────────┐
  │ a ──────┼────────────►│ list [1,2,3]     │
  ├─────────┤        ┌───►│ ob_refcnt = 2    │
  │ b ──────┼────────┘    └──────────────────┘
  └─────────┘

STEP 3: del a
────────────────────────────────────────
  Name table:              Heap:
  ┌─────────┐             ┌──────────────────┐
  │ (a gone)│             │ list [1,2,3]     │
  ├─────────┤        ┌───►│ ob_refcnt = 1    │ ← still alive! b holds it
  │ b ──────┼────────┘    └──────────────────┘
  └─────────┘

STEP 4: del b (or function returns with b as local)
────────────────────────────────────────
  Name table:              Heap:
  ┌─────────┐             ┌──────────────────┐
  │ (empty) │             │ list [1,2,3]     │
  └─────────┘             │ ob_refcnt = 0    │ ← FREED IMMEDIATELY
                          └──────────────────┘
                          CPython calls list.__del__ → decrements
                          refcounts of 1, 2, 3 as well
```

---

## Diagram 3: GIL Thread Switching — CPU vs I/O Bound

```
CPU-BOUND: Two threads, one GIL
───────────────────────────────────────────────────────────────────

Time ──────────────────────────────────────────────────────────────►
       5ms          5ms          5ms          5ms
  ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
  │ Thread 1 ││ Thread 2 ││ Thread 1 ││ Thread 2 │
  │ (running)││ (running)││ (running)││ (running)│
  └──────────┘└──────────┘└──────────┘└──────────┘
  Thread 2 waits     Thread 1 waits
  the whole time     the whole time
  Result: No parallelism. Threads take turns on ONE core.


I/O-BOUND: Two threads, GIL released during I/O
───────────────────────────────────────────────────────────────────

  Thread 1: ┌──────┐     ┌──────┐     ← executing Python
            │ CPU  │     │ CPU  │
            └──────┘─────└──────┘
                    │I/O wait│
            (GIL released immediately during I/O syscall)

  Thread 2:         ┌──────────────┐  ← runs during T1's I/O wait
                    │   CPU work   │
                    └──────────────┘

  Result: Effective parallelism for I/O-bound work!
          Both threads make progress during the same wall-clock time.


FIX FOR CPU-BOUND: Use multiprocessing
───────────────────────────────────────────────────────────────────

  Process 1: ┌─────────────────────────┐  Core 1: fully utilized
             │    Thread + own GIL     │
             └─────────────────────────┘

  Process 2: ┌─────────────────────────┐  Core 2: fully utilized
             │    Thread + own GIL     │
             └─────────────────────────┘

  Each process has its own Python interpreter and GIL.
  True parallelism across cores.
```

---

## Diagram 4: Python Import System Flow

```
Statement: import mypackage.utils

Step 1: Check sys.modules cache
────────────────────────────────
  sys.modules = {
    "os": <module 'os'>,
    "json": <module 'json'>,
    "mypackage": <module 'mypackage'>,    ← already loaded?
    ...
  }
  If "mypackage.utils" in sys.modules → return it immediately (done!)


Step 2: Walk sys.meta_path finders
────────────────────────────────────
  sys.meta_path = [
    BuiltinImporter,     ← handles "sys", "builtins", "gc", ...
    FrozenImporter,      ← handles frozen modules
    PathFinder,          ← searches sys.path for files
  ]

  Each finder.find_spec("mypackage.utils", path=None) called in order.
  First non-None result wins.


Step 3: PathFinder searches sys.path
──────────────────────────────────────
  sys.path = [
    "",                  ← current directory
    "/home/user/.venv/lib/python3.12/site-packages",
    "/usr/lib/python3.12",
    ...
  ]

  For each directory, look for:
    mypackage/utils.py
    mypackage/utils/__init__.py
    mypackage/utils.so  (C extension)


Step 4: Load the module
────────────────────────
  SourceFileLoader.exec_module(module):
    1. Read mypackage/utils.py source
    2. Compile to bytecode (or load cached .pyc)
    3. Execute bytecode in module's __dict__
    4. Add to sys.modules["mypackage.utils"]


Step 5: Return
───────────────
  The name "utils" is bound in the importer's namespace
  (for "import mypackage.utils", mypackage.utils is accessible)
```

---

## Diagram 5: C3 MRO (Method Resolution Order) Calculation

```
CLASS HIERARCHY:
                object
               /      \
              A        B
             / \      / \
            C   D    E   F
             \  |   /
              \ | /
               G

PROBLEM: What is G.__mro__?

C3 ALGORITHM:
  L[G] = G + merge(L[C], L[D], L[E], [C, D, E])

  L[A] = [A, object]
  L[B] = [B, object]
  L[C] = [C, A, object]
  L[D] = [D, A, object]
  L[E] = [E, B, object]
  L[F] = [F, B, object]

SIMPLE DIAMOND EXAMPLE: D(B, C), B(A), C(A)

  L[A] = [A, object]
  L[B] = [B, A, object]
  L[C] = [C, A, object]

  L[D] = D + merge([B, A, object], [C, A, object], [B, C])

  merge step 1: B is head of list 1; B not in tail of any list → take B
  L[D] so far: [D, B]
  remaining: merge([A, object], [C, A, object], [C])

  step 2: A is head of list 1; A IS in tail of list 2 [C, A, object] → SKIP
          C is head of list 2; C not in tail of any list → take C
  L[D] so far: [D, B, C]
  remaining: merge([A, object], [A, object], [])

  step 3: A is head; A not in tail of remaining → take A
  L[D] so far: [D, B, C, A]
  step 4: object → take object

  FINAL: D.__mro__ = [D, B, C, A, object]  ✓

  super() in B will call C next (not A!), ensuring A is called exactly once.
```

---

## Diagram 6: Python Stack Frame and Bytecode Execution

```
CALL STACK for: result = add(fibonacci(3), 10)

  ┌──────────────────────────────────────────────────────┐
  │  Frame 3 (top of stack): fibonacci(n=3)              │
  │  ┌────────────────────────────────────────────────┐  │
  │  │  f_code:    fibonacci.__code__                 │  │
  │  │  f_locals:  {'n': 3}                           │  │
  │  │  f_globals: module.__dict__                    │  │
  │  │  f_back: ──────────────────────────────────────┼──┼──►
  │  │  value stack: [3] [1] [2] ...                  │  │
  │  │  instruction pointer: → LOAD_FAST n            │  │
  │  └────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────┐
  │  Frame 2: add(a, b)                                  │
  │  ┌────────────────────────────────────────────────┐  │
  │  │  f_code:    add.__code__                       │  │
  │  │  f_locals:  {'a': 2, 'b': 10}                  │  │
  │  │  f_back: ──────────────────────────────────────┼──┼──►
  │  │  instruction pointer: → waiting for f(3) return│  │
  │  └────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────┐
  │  Frame 1 (bottom): module-level code                 │
  │  ┌────────────────────────────────────────────────┐  │
  │  │  f_code:    module's code object               │  │
  │  │  f_locals:  module.__dict__ (same as globals)  │  │
  │  │  f_back:    None (bottom of stack)             │  │
  │  └────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────┘


BYTECODE EXECUTION LOOP (simplified):
  while True:
      opcode, arg = next_instruction(frame)
      match opcode:
          case LOAD_FAST:   push(frame.f_locals[arg])
          case LOAD_GLOBAL: push(frame.f_globals[frame.co_names[arg]])
          case BINARY_OP:   right=pop(); left=pop(); push(left op right)
          case CALL:        new_frame = setup_frame(pop_callable(), pop_args())
                            result = execute(new_frame)
                            push(result)
          case RETURN_VALUE: return pop()
          case ...: check eval_breaker (GIL release, signals, GC)
```
