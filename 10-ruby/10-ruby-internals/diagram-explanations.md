# Chapter 10: Ruby Internals — Diagram Explanations

## Diagram 1: Ruby Execution Pipeline (Source → Running Code)

```
SOURCE FILE: app.rb
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│   STEP 1: LEXING (Tokenization)                             │
│   Source text → Tokens                                      │
│   "def add(a, b)" → [:kDEF, :tIDENT "add", "(", ...]       │
│   Prism lexer (Ruby 3.3+) — 25-45% faster than legacy      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│   STEP 2: PARSING                                            │
│   Tokens → AST (Abstract Syntax Tree)                       │
│                                                             │
│          DEFN                                               │
│         / | \                                               │
│        /  |  \                                              │
│      :add args  body                                        │
│             |    |                                          │
│           [a,b]  CALL(+)                                    │
│                  / \                                        │
│                 a   b                                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│   STEP 3: YARV COMPILATION                                  │
│   AST → Instruction Sequences (bytecode)                    │
│                                                             │
│   getlocal :a    ← push value of a onto stack              │
│   getlocal :b    ← push value of b onto stack              │
│   opt_plus       ← VM-optimized addition                   │
│   leave          ← return top of stack                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
┌──────────────────────┐    ┌───────────────────────────────┐
│  YARV INTERPRETER    │    │  YJIT (if enabled, hot code)  │
│                      │    │                               │
│  Executes bytecodes  │    │  bytecodes → native x86_64   │
│  one at a time       │    │  or arm64 machine code        │
│  ~100M ops/sec       │    │  ~300M-1B ops/sec             │
│  (pure Ruby)         │    │  (compiled to hardware)       │
└──────────────────────┘    └───────────────────────────────┘
```

---

## Diagram 2: Ruby GC Tri-Color Mark Algorithm

```
INITIAL STATE: all objects WHITE (potentially garbage)

  [A] [B] [C] [D] [E] [F] [G]
   W   W   W   W   W   W   W

GC ROOTS identified (stack vars, globals, constants):
  A is a root, B is a root

STEP 1: Mark roots as GREY:

  [A] [B] [C] [D] [E] [F] [G]
   G   G   W   W   W   W   W

  A references: C, D
  B references: E

STEP 2: Process grey A → mark BLACK, mark A's refs GREY:

  [A] [B] [C] [D] [E] [F] [G]
   B   G   G   G   W   W   W

STEP 3: Process grey B → mark BLACK, mark B's refs GREY:

  [A] [B] [C] [D] [E] [F] [G]
   B   B   G   G   G   W   W

  C references: F
STEP 4: Process grey C → mark BLACK, mark C's refs GREY:

  [A] [B] [C] [D] [E] [F] [G]
   B   B   B   G   G   G   W

STEP 5: Process grey D → no new references:

  [A] [B] [C] [D] [E] [F] [G]
   B   B   B   B   G   G   W

STEP 6: Process grey E → no new references:
STEP 7: Process grey F → no new references:

  [A] [B] [C] [D] [E] [F] [G]
   B   B   B   B   B   B   W

SWEEP: G is still WHITE → FREE IT

WRITE BARRIER (incremental GC protection):

  Mid-GC: A is BLACK, G is WHITE
  Ruby code runs: A.ref = G  ← creates BLACK → WHITE reference!

  Without barrier:
    G looks unreachable (never greyed) → freed! → A.ref = dangling pointer!

  With write barrier:
    A is "unblackened" back to GREY → re-processed
    G is marked GREY → eventually BLACK → survives GC
```

---

## Diagram 3: Object Shapes and YJIT Optimization

```
SHAPE-STABLE OBJECT (YJIT can optimize):

  class User
    def initialize(name, email)
      @name = name    ← transition: {} → {name}
      @email = email  ← transition: {name} → {name, email}
    end
  end

  All User instances follow shape #42:
  ┌─────────────────────────────────┐
  │  Shape #42: {name:0, email:1}   │  ← index map known at compile time
  │                                 │
  │  user.@name  → slot 0 (fast!)   │
  │  user.@email → slot 1 (fast!)   │
  └─────────────────────────────────┘

  YJIT compiled code:
    load r1, [object + 0]   ← direct memory access
    (no hash lookup, no symbol search)

SHAPE-UNSTABLE OBJECT (YJIT can't optimize):

  Instance A: @name first, then @email → Shape #42
  Instance B: @email first, then @name → Shape #99 (DIFFERENT!)

  YJIT compiled code for accessing @name:
    check shape_id
    if shape_id == 42: load r1, [object + 0]
    elif shape_id == 99: load r1, [object + 1]
    else: fallback to generic ivar lookup (slow)

  With 20 different shapes: constant shape-checking overhead

SHAPE TRANSITION TREE:

  {} (root shape)
   │
   ├── {name:} ──────── {name:, email:} ──── shape #42 (User shape)
   │
   ├── {email:} ─────── {email:, name:} ──── shape #99 (diverged!)
   │
   └── {id:} ─────────── {id:, value:} ───── shape #7  (different class)
```

---

## Diagram 4: Ractor Memory Model vs Thread Model

```
THREAD MODEL (with GIL):

  Main Thread ──────────────────────────────────────────────────
  Thread 1    ──  BLOCKED (waiting for GIL) ──  RUNS ──  WAIT ─
  Thread 2    ──  RUNS ──  BLOCKED (GIL) ──  WAIT ──  RUNS ────

  SHARED HEAP: [obj1, obj2, obj3, ...]  ← ALL threads can access

  Only ONE thread executes Ruby code at a time (GIL)
  But ALL threads share the same heap (mutation = race conditions)

RACTOR MODEL (no GIL per Ractor):

  Main Ractor  ─── exec ─── exec ─── exec ─────────────────────
  Ractor 1     ─── exec ─── exec ─── exec ─────────────────────  ← PARALLEL
  Ractor 2     ─── exec ─── exec ─── exec ─────────────────────  ← PARALLEL

  HEAP IS ISOLATED:
  ┌───────────────────┐   ┌───────────────────┐   ┌──────────────────┐
  │  Main Ractor      │   │  Ractor 1          │   │  Ractor 2        │
  │  private heap     │   │  private heap      │   │  private heap    │
  │  [obj1, obj2]     │   │  [obj3, obj4]      │   │  [obj5, obj6]    │
  └────────┬──────────┘   └──────────┬─────────┘   └────────┬─────────┘
           │                         │                       │
           └──── COMMUNICATION ──────┘───────────────────────┘
                 via message passing
                 (ONLY frozen/shareable objects)

  Ractor.yield(frozen_value)  → sends to receiver
  Ractor.receive              → blocks until message arrives
```

---

## Diagram 5: Method Dispatch and `method_missing` Chain

```
obj.some_method
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  METHOD LOOKUP (searches in order):                     │
│                                                         │
│  1. obj singleton class (eigenclass)                    │
│  2. Prepended modules (last prepend first)              │
│  3. obj.class                                           │
│  4. Included modules (last include first)               │
│  5. Superclass (repeat 1-4 for each ancestor)           │
│  6. BasicObject                                         │
│                                                         │
│  FOUND? ─── YES ──→ Execute the method, DONE           │
│             │                                           │
│            NO                                           │
│             │                                           │
│             ▼                                           │
│  Look for `method_missing` (same lookup order)          │
│                                                         │
│  FOUND? ─── YES ──→ Call method_missing(name, *args)   │
│             │                                           │
│            NO                                           │
│             │                                           │
│             ▼                                           │
│  BasicObject#method_missing → NoMethodError            │
└─────────────────────────────────────────────────────────┘

CORRECT method_missing + respond_to_missing? PAIR:

  ┌────────────────────────────────────────────────────────┐
  │  method_missing:              respond_to_missing?:     │
  │  "Do I handle X?"             "Tell the world if I    │
  │  → if yes: execute           do handle X"              │
  │  → if no: super              → must MIRROR MM logic   │
  │                                                        │
  │  MISSING respond_to_missing? BREAKS:                  │
  │  - respond_to?(:x) → false (wrong!)                   │
  │  - method(:x)       → NameError                       │
  │  - Duck type checks → fail silently                   │
  └────────────────────────────────────────────────────────┘
```

---

## Diagram 6: Ruby Object Hierarchy

```
INHERITANCE HIERARCHY:

  BasicObject       ← absolute root (~8 methods, no Kernel)
       │
     Object         ← includes Kernel module
       │
    ┌──┴──────────────────────────────────────────┐
    │                                             │
  Numeric          String           Array        Hash ...
    │
  ┌─┴──────────────────┐
  │                    │
Integer              Float
   │
  ...


SINGLETON CLASS (EIGENCLASS) HIERARCHY:

  Every object, including classes, has a singleton class:

  42 (Integer instance)
  └── singleton class of 42
      └── Integer (the class)
          └── singleton class of Integer  ← class methods of Integer live here
              └── Numeric
                  └── singleton class of Numeric
                      └── ...

  Dog (a user class)
  └── singleton class of Dog  ← Dog.bark lives here
      └── singleton class of Animal
          └── singleton class of Object
              └── singleton class of BasicObject
                  └── Class  (all class singleton classes inherit from Class)
                      └── Module
                          └── Object


KERNEL MODULE MIXIN:

  BasicObject  (8 methods)
       │
       │  Kernel (mixes in: puts, require, raise, loop, rand, etc.)
       ▼
    Object     (= BasicObject + Kernel)
       │
    All user classes inherit from Object
    → All Ruby objects have Kernel's methods available
```
