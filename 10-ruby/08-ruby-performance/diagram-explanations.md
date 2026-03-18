# Chapter 08: Ruby Performance — Diagram Explanations

## Diagram 1: Ruby Generational GC — Object Lifecycle

```
OBJECT ALLOCATION AND GC LIFECYCLE

  Object.new  →  Allocates on Eden heap (young generation)
                 ↓
  ┌─────────────────────────────────────────────────────────────┐
  │                    RUBY HEAP                                │
  │                                                             │
  │  ┌─────────────────────────┐  ┌─────────────────────────┐  │
  │  │   YOUNG GENERATION       │  │   OLD GENERATION        │  │
  │  │   (Eden + Survivor)      │  │   (Tenured)             │  │
  │  │                          │  │                         │  │
  │  │  [obj1][obj2][obj3]...   │  │  [ClassDef][Config]     │  │
  │  │  [temp][temp][temp]...   │  │  [CachedData][Routes]   │  │
  │  │                          │  │                         │  │
  │  │  Most objects die here!  │  │  Long-lived objects     │  │
  │  │  (short-lived temps)     │  │  promoted from young    │  │
  │  └──────────────────────────┘  └─────────────────────────┘  │
  │                                                             │
  │  Minor GC:  scans YOUNG only (fast, frequent)              │
  │  Major GC:  scans BOTH (slow, infrequent)                  │
  └─────────────────────────────────────────────────────────────┘

OBJECT PROMOTION FLOW:

  new object
      │
      ▼
  Eden heap ──[survived Minor GC 1]──→ Survivor space
                                            │
                                   [survived Minor GC 2]
                                            │
                                            ▼
                                       OLD generation
                                       (lives until Major GC
                                        or explicit dereference)

WHAT CAUSES GC RUNS:

  Minor GC triggers when:
  ├── Eden heap is full
  └── RUBY_GC_MALLOC_LIMIT reached

  Major GC triggers when:
  ├── Old generation exceeds RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR
  ├── After N minor GC runs
  └── malloc limit exceeded for large objects

GC TUNING ENVIRONMENT VARIABLES:

  RUBY_GC_HEAP_INIT_SLOTS=600000       # Start with larger heap → fewer early GCs
  RUBY_GC_HEAP_GROWTH_FACTOR=1.25      # Heap grows 25% when full (default)
  RUBY_GC_HEAP_GROWTH_MAX_SLOTS=0      # Max slots to grow (0 = unlimited)
  RUBY_GC_HEAP_FREE_SLOTS=4096         # Minimum free slots before GC
  RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR=2.0  # Major GC threshold
  RUBY_GC_MALLOC_LIMIT=67108864        # 64MB malloc limit
```

---

## Diagram 2: Object Allocation Flow

```
Ruby code: result = User.new(name: "Alice")
                │
                ▼
    ┌─────────────────────────────────┐
    │   Ruby interpreter sees:        │
    │   1. Constant User (class)      │
    │   2. Call .new                  │
    │   3. String literal "Alice"     │
    └──────────────┬──────────────────┘
                   │
                   ▼
    ┌─────────────────────────────────────────────┐
    │   OBJECT ALLOCATION                         │
    │                                             │
    │   String "Alice" → allocate RString         │
    │   User instance  → allocate RObject         │
    │                                             │
    │   Each allocation:                          │
    │   1. Check free list in Eden heap           │
    │   2. If available: use it (fast path)       │
    │   3. If Eden full: trigger Minor GC          │
    │   4. After GC: if still full → grow heap    │
    └──────────────┬──────────────────────────────┘
                   │
                   ▼
    ┌─────────────────────────────────────────────┐
    │   REFERENCE COUNT ≠ Ruby (uses tracing GC)  │
    │                                             │
    │   When result goes out of scope:            │
    │   - Ruby does NOT immediately free          │
    │   - Object sits until next GC run           │
    │   - GC marks all REACHABLE objects          │
    │   - Unmarked objects are reclaimed          │
    └─────────────────────────────────────────────┘

ALLOCATION HOTSPOT IDENTIFICATION:

  require 'memory_profiler'
  MemoryProfiler.report { your_code }.pretty_print

  Output shows:
  ┌──────────────────────────────────────────────┐
  │ Allocated objects by class                   │
  │                                              │
  │   String         5234  (53% of allocations)  │
  │   Array          1023                        │
  │   Hash            456                        │
  │   User            100                        │
  │                                              │
  │ Allocated objects by location                │
  │                                              │
  │   app/services/processor.rb:45  → 2300 objs │
  │   app/models/user.rb:12         →  800 objs  │
  └──────────────────────────────────────────────┘
```

---

## Diagram 3: `||=` Memoization Trap — Visual

```
CORRECT memoization (truthy value):
  @result = nil          (initial state)
  @result ||= compute()  (checks: nil? → yes → compute → store 42)
  @result = 42           (stored!)
  @result ||= compute()  (checks: 42? → truthy → skip compute)
  @result = 42           ✓ (cache hit!)

BROKEN memoization (falsy value):
  @result = nil          (initial state)
  @result ||= compute()  (checks: nil? → yes → compute → false)
  @result = false        (stored!)
  @result ||= compute()  (checks: false? → FALSY → compute again!)
  @result = false        ← re-computed! cache miss every time

  ┌────────────────────────────────────────────────────────────┐
  │  FALSY values that defeat ||=:                             │
  │    nil   → returned by methods that find nothing          │
  │    false → returned by boolean checks, feature flags      │
  │    0     → NOT falsy in Ruby (0 is truthy!) ← safe        │
  │    ""    → NOT falsy in Ruby ("" is truthy!) ← safe       │
  └────────────────────────────────────────────────────────────┘

FIX — use key? or defined?:

  Hash cache:
  @cache ||= {}
  unless @cache.key?(key)        ← "was this slot ever written?"
    @cache[key] = compute(key)
  end
  @cache[key]

  Instance variable:
  return @result if defined?(@result)  ← "was this ivar ever assigned?"
  @result = compute

  SAFE for all values:      nil, false, 0, "", true, 42, Object.new
  ||= SAFE for:             only truthy values
```

---

## Diagram 4: String Concatenation Memory Comparison

```
SCENARIO: Building a string with 5 concatenations

String#+ (creates new objects):

  step 0: ""          [obj A, size 0]
  step 1: "a"         [obj B, size 1]  → obj A abandoned
  step 2: "ab"        [obj C, size 2]  → obj B abandoned
  step 3: "abc"       [obj D, size 3]  → obj C abandoned
  step 4: "abcd"      [obj E, size 4]  → obj D abandoned
  step 5: "abcde"     [obj F, size 5]  → obj E abandoned

  Total objects created: 6
  Abandoned objects (GC pressure): 5
  Final: obj F

String#<< (mutates in place):

  step 0: ""          [obj A, size 0]
  step 1: "a"         [obj A, size 1]  ← same object!
  step 2: "ab"        [obj A, size 2]  ← same object!
  step 3: "abc"       [obj A, size 3]  ← same object!
  step 4: "abcd"      [obj A, size 4]  ← same object!
  step 5: "abcde"     [obj A, size 5]  ← same object!

  Total objects created: 1
  Abandoned objects (GC pressure): 0
  Final: obj A (mutated)

SCALING: for 10,000 concatenations
  String#+  → 10,001 objects, 10,000 GC candidates
  String#<< → 1 object, 0 GC candidates

FROZEN STRING OPTIMIZATION:
  # frozen_string_literal: true

  CONSTANT = "prefix"    ← 1 object, shared across all references
  CONSTANT.frozen?       ← true

  str = +"prefix"        ← unfrozen copy for mutation (+ unary operator)
  str << " suffix"       ← ok: unfrozen
```

---

## Diagram 5: YJIT Compilation Pipeline

```
WITHOUT YJIT (pure interpreter):
  Ruby source code
        │
        ▼
  Parser (Prism in Ruby 3.3+)
        │
        ▼
  AST (Abstract Syntax Tree)
        │
        ▼
  YARV bytecode instructions
        │ (interpreted on every call)
        ▼
  Ruby bytecode interpreter
  (pure software — slow)

WITH YJIT ENABLED:
  First call (cold):
  Ruby source → AST → YARV bytecode → Interpreter
                                             │
                                      YJIT observes
                                      types, profiles
                                             │
                                      [threshold reached]

  Subsequent calls (hot method):
  Ruby source → AST → YARV bytecode → YJIT compiler
                                             │
                                      Native machine code
                                      (stored in code cache)
                                             │
                                      Direct CPU execution
                                      (no interpreter overhead)

YJIT EFFECTIVENESS:
  ┌────────────────────────────────────────────────────────┐
  │  Code Type          │ YJIT Benefit │ Why               │
  ├────────────────────────────────────────────────────────┤
  │  Integer math loops  │ 2-3x faster  │ Native arithmetic │
  │  Method-heavy code   │ 1.3-2x       │ Inlined calls     │
  │  String processing   │ 1.1-1.3x     │ Partial benefit   │
  │  Hash/Array access   │ 1.1-1.5x     │ Type specialization│
  │  DB query wait       │ 0% faster    │ Blocked on I/O    │
  │  Sleep/network wait  │ 0% faster    │ Not CPU-bound     │
  │  method_missing heavy│ < 5% faster  │ Hard to specialize│
  └────────────────────────────────────────────────────────┘
```

---

## Diagram 6: Profiling Workflow Decision Tree

```
APPLICATION IS SLOW
        │
        ▼
  ┌─────────────────────────────┐
  │  Measure first!             │
  │  Is it CPU-bound or I/O?    │
  └─────────────┬───────────────┘
                │
   utime ≈ real │               real >> utime
  (CPU bound)   │               (I/O bound)
                │
    ┌───────────▼───────────┐   ┌───────────────────────────┐
    │   CPU PROFILING        │   │   I/O PROFILING            │
    │                        │   │                            │
    │ stackprof --mode=cpu   │   │ stackprof --mode=wall      │
    │ ruby-prof (detailed)   │   │ rack-mini-profiler (SQL)   │
    │ benchmark-ips          │   │ bullet gem (N+1)           │
    │ YJIT stats             │   │ EXPLAIN ANALYZE            │
    └───────────┬────────────┘   └───────────┬────────────────┘
                │                            │
    ┌───────────▼────────────┐   ┌───────────▼────────────────┐
    │  OPTIMIZE CPU          │   │  OPTIMIZE I/O              │
    │                        │   │                            │
    │ - Better algorithm     │   │ - Batch queries (avoid N+1)│
    │ - flat_map vs map+flat │   │ - Connection pooling       │
    │ - freeze string lits   │   │ - Caching (Redis/Memcache) │
    │ - Avoid allocations    │   │ - Async/concurrent I/O     │
    │ - YJIT                 │   │ - DB indexes               │
    │ - Ractors (parallel)   │   │ - Reduce payload size      │
    └────────────────────────┘   └────────────────────────────┘

OPTIMIZATION PRIORITY (always in this order):
  1. Profile first (know your bottleneck)
  2. Fix algorithms (O(n²) → O(n log n))
  3. Fix data structures (Array → Set, Hash)
  4. Fix I/O (batch, cache, async)
  5. Reduce allocations (freeze, reuse)
  6. Low-level tuning (YJIT, GC params)
  7. C extensions (last resort)
```
