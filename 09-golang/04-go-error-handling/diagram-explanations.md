# Go Error Handling — Diagram Explanations

## Diagram 1: Error Wrapping Chain and `errors.Is` Traversal

When errors are wrapped with `fmt.Errorf("%w", err)`, they form a linked chain. `errors.Is` walks this chain looking for a match.

```
WRAPPING (building the chain):

errors.New("connection refused")
         │
         ▼
┌─────────────────────────────────────────┐
│  errorString                            │
│  msg: "connection refused"              │
│  Unwrap() → nil                         │
└─────────────────────────────────────────┘
         │ fmt.Errorf("dial tcp: %w", err)
         ▼
┌─────────────────────────────────────────┐
│  wrapError                              │
│  msg: "dial tcp: connection refused"    │
│  Unwrap() ──────────────────────────────┼──► errorString
└─────────────────────────────────────────┘
         │ fmt.Errorf("connect db: %w", err)
         ▼
┌─────────────────────────────────────────┐
│  wrapError                              │
│  msg: "connect db: dial tcp: ..."       │
│  Unwrap() ──────────────────────────────┼──► wrapError above
└─────────────────────────────────────────┘
         │ fmt.Errorf("init app: %w", err)
         ▼
┌─────────────────────────────────────────┐
│  wrapError                              │
│  msg: "init app: connect db: ..."       │
│  Unwrap() ──────────────────────────────┼──► wrapError above
└─────────────────────────────────────────┘
         │
         ▼
       err3 (what the caller receives)

─────────────────────────────────────────

errors.Is(err3, ErrConnRefused) TRAVERSAL:

err3 ──► check == ErrConnRefused? NO
          │
          └─► Unwrap() → err2
                          │
                          ├─► check == ErrConnRefused? NO
                          │
                          └─► Unwrap() → err1
                                          │
                                          ├─► check == ErrConnRefused? YES ✓
                                          │
                                          └─► RETURN true
```

---

## Diagram 2: `errors.Is` vs `errors.As` Comparison Flow

```
Given error chain:
  outerErr → middleErr → innerErr (type *DBError, code 404)

─────────────────────────────────────────────────────────────────

errors.Is(outerErr, sentinelErr):

  outerErr
     │
     ├── outerErr == sentinelErr ?
     │        YES → return true
     │        NO  ↓
     ├── outerErr.Is(sentinelErr) ?  (if custom Is() implemented)
     │        YES → return true
     │        NO  ↓
     └── Unwrap() → middleErr
              │
              ├── middleErr == sentinelErr ?
              │        YES → return true
              │        NO  ↓
              └── Unwrap() → innerErr
                       │
                       ├── innerErr == sentinelErr ?
                       │        YES → return true
                       │        NO  ↓
                       └── Unwrap() → nil → return false

─────────────────────────────────────────────────────────────────

errors.As(outerErr, &target):    (target is **DBError)

  outerErr
     │
     ├── outerErr assignable to *DBError ?
     │        YES → *target = outerErr.(*DBError); return true
     │        NO  ↓
     └── Unwrap() → middleErr
              │
              ├── middleErr assignable to *DBError ?
              │        YES → *target = middleErr.(*DBError); return true
              │        NO  ↓
              └── Unwrap() → innerErr
                       │
                       ├── innerErr assignable to *DBError ?
                       │        YES → *target = innerErr.(*DBError); return true
                       │            (caller now has the typed error)
                       │        NO  ↓
                       └── Unwrap() → nil → return false

─────────────────────────────────────────────────────────────────

KEY DIFFERENCE:

  errors.Is  →  identity match     (is it THIS specific value?)
  errors.As  →  type match         (is it ANY value of this type?)
```

---

## Diagram 3: panic / recover Execution Flow

```
Normal execution flow:

  main() → funcA() → funcB() → funcC()
               ↑          ↑          ↑
           defer f1    defer f2    panic("oops")


When panic("oops") is called:

  1. funcC stops executing immediately
  2. funcC's deferred functions run (none in this example)
  3. Stack unwinds to funcB
  4. funcB's deferred functions run:
     └─► defer f2 runs
         ├─ if f2 calls recover() → PANIC INTERCEPTED
         │     panic value returned, normal flow resumes in f2
         │     f2 returns normally (with named return err set)
         └─ if f2 does NOT call recover() → continue unwinding

  5. Stack unwinds to funcA
  6. funcA's deferred functions run:
     └─► defer f1 runs
         └─ same check for recover()

  7. If nobody recovers → program crashes with stack trace

─────────────────────────────────────────────────────────────────

Code with recovery:

func riskyOp() (err error) {
    defer func() {                    ◄── step 2: this defer runs
        if r := recover(); r != nil { ◄── step 3: intercepts panic
            err = fmt.Errorf(...)     ◄── step 4: sets named return
        }
    }()

    doWork()   ◄── step 1: panics here
    return nil
}

Timeline:
  doWork() panics
       │
       ▼
  deferred func executes
       │
       ▼
  recover() captures panic value
       │
       ▼
  err is set via named return
       │
       ▼
  riskyOp() returns (nil, err) to caller
       │
       ▼
  caller handles err normally — no crash

─────────────────────────────────────────────────────────────────

CROSS-GOROUTINE PANIC (cannot be recovered):

  goroutine 1 (main):             goroutine 2:
  ┌─────────────────────┐         ┌─────────────────────┐
  │ defer recover() ← ─ ─ ─ ─ ─ ─│─ CANNOT cross here  │
  │                     │         │                     │
  │ go func() {─────────┼─────────┼► panic("boom")      │
  │   ...               │         │  ↓                  │
  │ }()                 │         │  program crashes     │
  │                     │         │  (whole process)    │
  └─────────────────────┘         └─────────────────────┘

  RULE: Each goroutine must have its own recovery.
```

---

## Diagram 4: Custom Error Type Hierarchy

```
Built-in error interface:
┌─────────────────────────────┐
│  interface error            │
│  ┌───────────────────────┐  │
│  │  Error() string       │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
              ▲
              │ implements
     ┌────────┴──────────────────────────────┐
     │                                       │
┌────┴──────────────────┐    ┌───────────────┴──────────────────┐
│  *errors.errorString  │    │  *fmt.wrapError                  │
│  (from errors.New)    │    │  (from fmt.Errorf %w)            │
│  Error() → msg        │    │  Error() → formatted string      │
│  Unwrap() → nil       │    │  Unwrap() → wrapped error        │
└───────────────────────┘    └──────────────────────────────────┘
                                             ▲
              ┌──────────────────────────────┤
              │ your custom types also fit   │
              │                              │
┌─────────────┴─────────────┐  ┌────────────┴──────────────────┐
│  *ValidationError         │  │  *DBError                     │
│  Field   string           │  │  Op     string                │
│  Message string           │  │  Table  string                │
│  Error() → string         │  │  Code   int                   │
│  Unwrap() → nil           │  │  Err    error                 │
└───────────────────────────┘  │  Error() → string             │
                                │  Unwrap() → Err               │
                                └───────────────────────────────┘

errors.As traversal picks the correct box by type.
errors.Is traversal walks Unwrap() chains checking identity.
```

---

## Diagram 5: Early Return Guard Clause Pattern

```
WITHOUT guard clauses (deeply nested — hard to read):

func process(id int) error {
    data, err := fetchData(id)
    if err == nil {
        ┌────────────────────────────────────────────────────┐
        │ result, err := transform(data)                     │
        │ if err == nil {                                    │
        │     ┌──────────────────────────────────────────┐  │
        │     │ validated, err := validate(result)        │  │
        │     │ if err == nil {                           │  │
        │     │     ┌────────────────────────────────┐   │  │
        │     │     │ return save(validated)          │   │  │
        │     │     └────────────────────────────────┘   │  │
        │     │ }                                         │  │
        │     │ return err  ◄── buried return             │  │
        │     └──────────────────────────────────────────┘  │
        │ }                                                   │
        │ return err  ◄── buried return                      │
        └────────────────────────────────────────────────────┘
    }
    return err  ◄── buried return
}
      ↑
  PROBLEMS: error handling buried in nesting
            hard to see the happy path
            error returns spread throughout

─────────────────────────────────────────────────────────────────

WITH guard clauses (early return — idiomatic Go):

func process(id int) error {
    data, err := fetchData(id)         ──► error check
    if err != nil {                         │
        return fmt.Errorf("...: %w", err)  ◄┘ return EARLY
    }
    ↓ happy path continues here (no nesting)

    result, err := transform(data)     ──► error check
    if err != nil {                         │
        return fmt.Errorf("...: %w", err)  ◄┘ return EARLY
    }
    ↓ happy path continues here

    validated, err := validate(result) ──► error check
    if err != nil {                         │
        return fmt.Errorf("...: %w", err)  ◄┘ return EARLY
    }
    ↓ happy path continues here

    return save(validated)             ──► single success return at bottom
}

  BENEFITS: happy path visible as left-aligned code
            errors handled immediately where they occur
            each error adds its own context via %w
            easy to add/remove steps without restructuring
```

---

## Diagram 6: `errors.Join` Multi-Error Tree (Go 1.20+)

```
errors.Join(err1, err2, err3) creates:

┌────────────────────────────────────────────────┐
│  joinError                                     │
│  errs: [err1, err2, err3]                      │
│  Error() → "err1\nerr2\nerr3"                  │
│  Unwrap() []error → [err1, err2, err3]         │
└────────────────────────────────────────────────┘
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
  err1       err2       err3

errors.Is traversal (tree BFS/DFS):

  errors.Is(joined, err2)
  │
  ├─► joined == err2? NO
  │
  └─► Unwrap() []error → [err1, err2, err3]
       ├─► err1 == err2? NO
       ├─► err2 == err2? YES → return true ✓
       └─► (err3 not checked — already found)

─────────────────────────────────────────────────────────────────

COMPARISON: Single vs Multi Unwrap

Single (fmt.Errorf %w):          Multi (errors.Join):

    outerErr                           joinErr
        │                           ┌──┼──┐
        ▼                           ▼  ▼  ▼
    innerErr                      e1  e2  e3
  (linear chain)                (tree structure)

Both are traversed correctly by errors.Is and errors.As.
```
