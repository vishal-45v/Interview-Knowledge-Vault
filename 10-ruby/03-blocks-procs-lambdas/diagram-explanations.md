# Chapter 03 — Blocks, Procs & Lambdas: Diagram Explanations

---

## Diagram 1 — Proc vs Lambda Comparison

```
  ┌─────────────────────────────────┬──────────────────────────────────┐
  │           PROC                  │           LAMBDA                 │
  ├─────────────────────────────────┼──────────────────────────────────┤
  │ Created with:                   │ Created with:                    │
  │   Proc.new { }                  │   lambda { }                     │
  │   proc { }                      │   -> { }                         │
  │   &block param                  │                                  │
  ├─────────────────────────────────┼──────────────────────────────────┤
  │ lambda? => false                │ lambda? => true                  │
  ├─────────────────────────────────┼──────────────────────────────────┤
  │ ARITY: lenient                  │ ARITY: strict                    │
  │   Too few  → missing = nil      │   Too few  → ArgumentError       │
  │   Too many → extras ignored     │   Too many → ArgumentError       │
  ├─────────────────────────────────┼──────────────────────────────────┤
  │ RETURN: exits enclosing method  │ RETURN: exits lambda only        │
  │   (like return in a block)      │   (like return in a method)      │
  ├─────────────────────────────────┼──────────────────────────────────┤
  │ BREAK: raises LocalJumpError    │ BREAK: exits lambda (like return)│
  │   if proc called outside method │                                  │
  └─────────────────────────────────┴──────────────────────────────────┘
```

---

## Diagram 2 — Block Scope vs Method Scope

```
  outer_var = "I'm outside"

  def method_scope
    # outer_var is NOT accessible here (methods create new scope)
    outer_var  # => NameError: undefined local variable
  end

  [1,2,3].each do |n|
    # Blocks SHARE the outer scope
    puts outer_var  # => "I'm outside"  (accessible!)
  end

  ─────────────────────────────────────────────────────────

  SCOPE GATES (things that CREATE new scope):
    def ... end         ← NEW scope (can't see outer locals)
    class ... end       ← NEW scope
    module ... end      ← NEW scope

  SCOPE BRIDGES (things that SHARE outer scope):
    do...end blocks     ← SHARED scope
    { } blocks          ← SHARED scope
    Proc.new { }        ← SHARED scope (closure)
    lambda { }          ← SHARED scope (closure)
```

---

## Diagram 3 — yield Execution Flow

```
  def wrap(value)          ← method begins
    puts "before"
    result = yield(value)  ← PAUSES here, passes control to block
    puts "after"           ← RESUMES here after block returns
    result
  end

  wrap(10) { |n| n * 2 }

  EXECUTION FLOW:
  1. wrap(10) called
  2. puts "before"   → "before"
  3. yield(10)  ──────────────────→  block receives n=10
                                      evaluates: 10 * 2
                                      returns: 20
  4. ←──────────────────────────────  block returns 20 to yield
  5. result = 20
  6. puts "after"    → "after"
  7. returns 20
```

---

## Diagram 4 — Closure: Variable Capture

```
  def make_counter
    count = 0         ← local var in make_counter

    increment = -> { count += 1 }   ─┐
    decrement = -> { count -= 1 }   ─┤── all three lambdas
    current   = -> { count }        ─┘   capture the SAME `count`

    [increment, decrement, current]
  end   ← make_counter returns, but count lives on in the lambdas

  inc, dec, cur = make_counter

  inc.call  # count becomes 1
  inc.call  # count becomes 2
  dec.call  # count becomes 1
  cur.call  # => 1

  MEMORY:
  ┌─────────────────────────────────────────────────────┐
  │   count [binding]  ←──────────────────────────────┐ │
  │                                                   │ │
  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │ │
  │  │  increment  │  │  decrement  │  │ current  │  │ │
  │  │  -> { count─┼──┼─────────────┼──┼──────────┼──┘ │
  │  │    += 1  }  │  │  ...        │  │ ...      │    │
  │  └─────────────┘  └─────────────┘  └──────────┘    │
  └─────────────────────────────────────────────────────┘
```

---

## Diagram 5 — &:symbol Transformation Chain

```
  ["hello", "world"].map(&:upcase)

  STEPS:
  1. :upcase               ← a Symbol
  2. &:upcase              ← & calls :upcase.to_proc
  3. :upcase.to_proc       ← returns Proc { |obj| obj.send(:upcase) }
  4. & converts it         ← to a block for map
  5. map iterates:
       "hello".send(:upcase)  → "HELLO"
       "world".send(:upcase)  → "WORLD"
  6. Result: ["HELLO", "WORLD"]

  EQUIVALENT LONGHAND:
  ["hello", "world"].map { |obj| obj.upcase }
```

---

## Diagram 6 — Lazy vs Eager Evaluation Pipeline

```
  EAGER (processes all at each step):

  [1..1000000]
       │
       ▼ map { |n| n * 2 }     ← creates 1,000,000 element array
       │
       ▼ select { |n| n > 100 } ← creates another large array
       │
       ▼ first(5)              ← takes 5 from big array
       └→ [102, 104, 106, 108, 110]

  LAZY (processes element by element, stops at first(5)):

  1..1000000
       │
       ▼ .lazy
       │   ↕
       │  1 → map (→2)  → select? no  → continue
       │  2 → map (→4)  → select? no  → continue
       │  ...
       │ 51 → map (→102) → select? yes → count=1
       │ 52 → map (→104) → select? yes → count=2
       │ 53 → map (→106) → select? yes → count=3
       │ 54 → map (→108) → select? yes → count=4
       │ 55 → map (→110) → select? yes → count=5  ← STOP
       │
       └→ [102, 104, 106, 108, 110]
```

---

## Diagram 7 — Proc/Lambda Composition with >> and <<

```
  double  = ->(n) { n * 2 }
  add_one = ->(n) { n + 1 }
  to_str  = ->(n) { n.to_s }

  PIPELINE with >>  (left to right):
  pipeline = double >> add_one >> to_str

  Input: 5
  5 ──→ double ──→ 10 ──→ add_one ──→ 11 ──→ to_str ──→ "11"

  COMPOSITION with <<  (right to left, math notation f∘g):
  composed = to_str << add_one << double

  Input: 5
  5 ──→ double ──→ 10 ──→ add_one ──→ 11 ──→ to_str ──→ "11"
  (same result here — different notation, not different order)

  ORDER MATTERS when different functions:
  f = add_one >> double   → (5+1)*2 = 12
  g = double >> add_one   → (5*2)+1 = 11
```

---

## Diagram 8 — Currying: Partial Application

```
  ORIGINAL lambda:
  multiply = ->(a, b) { a * b }
  multiply.call(3, 4)  → 12

  CURRIED:
  curried = multiply.curry

  PARTIAL APPLICATION:
  triple = curried.call(3)   ← pre-fill a=3, returns new lambda waiting for b
  double = curried.call(2)   ← pre-fill a=2, returns new lambda waiting for b

  FINAL CALL:
  triple.call(4)   → 12   (3 * 4)
  double.call(5)   → 10   (2 * 5)

  VISUAL:
  multiply(a, b)
      │
      ▼ .curry.call(3)
  ->(b) { 3 * b }   ← triple — waiting for b
      │
      ▼ .call(4)
  12

  [1,2,3,4].map(&triple)  → [3, 6, 9, 12]
```
