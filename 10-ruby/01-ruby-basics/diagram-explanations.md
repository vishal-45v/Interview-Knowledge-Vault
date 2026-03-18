# Chapter 01 — Ruby Basics: Diagram Explanations

---

## Diagram 1 — Ruby Object Hierarchy

Every Ruby value is an instance of a class, and every class ultimately inherits from BasicObject.

```
BasicObject         <- Root of everything; minimal interface
    |
  Object            <- Most objects inherit from here
    |
  Numeric           <- Parent of numeric types
   / \
Integer  Float

  Object
    |
  String
  Symbol
  Array
  Hash
  NilClass          <- nil's class
  TrueClass         <- true's class
  FalseClass        <- false's class
```

```ruby
# Verify the hierarchy:
42.class            # => Integer
42.class.superclass # => Numeric
Numeric.superclass  # => Object
Object.superclass   # => BasicObject
BasicObject.superclass  # => nil  (top of the chain)

nil.class           # => NilClass
NilClass.superclass # => Object
```

---

## Diagram 2 — Variable Scope Visualization

```
 ┌─────────────────────────────────────────────────────────────┐
 │  GLOBAL SCOPE ($global, constants)                          │
 │                                                             │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │  CLASS SCOPE (@@class_var, CONSTANT)                  │  │
 │  │                                                       │  │
 │  │  ┌─────────────────────────────────────────────────┐  │  │
 │  │  │  INSTANCE SCOPE (@instance_var)                 │  │  │
 │  │  │                                                 │  │  │
 │  │  │  ┌───────────────────────────────────────────┐  │  │  │
 │  │  │  │  METHOD SCOPE (local_var)                 │  │  │  │
 │  │  │  │                                           │  │  │  │
 │  │  │  │  ┌─────────────────────────────────────┐  │  │  │  │
 │  │  │  │  │  BLOCK SCOPE (block_var, local_var) │  │  │  │  │
 │  │  │  │  │  Shares outer local scope           │  │  │  │  │
 │  │  │  │  └─────────────────────────────────────┘  │  │  │  │
 │  │  │  └───────────────────────────────────────────┘  │  │  │
 │  │  └─────────────────────────────────────────────────┘  │  │
 │  └───────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────┘
```

```ruby
$global = "everywhere"     # accessible at all scopes

class MyClass
  @@shared = "class-wide"  # accessible to all instances

  def initialize
    @own = "mine alone"    # per-instance
  end

  def work
    local = "method only"  # dies when method returns

    [1,2,3].each do |n|
      block_local = n      # also in block scope
      local                # CAN access outer local_var
      @own                 # CAN access instance var
    end

    block_local            # => NameError! died with the block
  end
end
```

---

## Diagram 3 — Truthiness Map

```
  Ruby Value         Truthy?    Notes
  ─────────────────────────────────────────────────────────
  nil                FALSY      "no value"
  false              FALSY      "explicit false"
  ─────────────────────────────────────────────────────────
  true               truthy
  0                  truthy     ← Different from C/Python!
  ""                 truthy     ← Different from many languages!
  []                 truthy     ← Empty array is truthy!
  {}                 truthy     ← Empty hash is truthy!
  0.0                truthy
  "false"            truthy     ← The STRING "false" is truthy!
  :anything          truthy
  any object         truthy
```

---

## Diagram 4 — String vs Symbol Memory Model

```
 STRING LITERALS — each creates a new object:

  "status"         "status"          "status"
     │                │                  │
  obj_id:21        obj_id:23          obj_id:25
  (different objects every time)


 SYMBOL LITERALS — always the same object:

  :status          :status           :status
     │                │                  │
     └────────────────┴──────────────────┘
                       │
                   obj_id:708988
            (same object, always, forever)
```

---

## Diagram 5 — Array Methods: Adding and Removing

```
  arr = [1, 2, 3, 4, 5]

  FRONT                                              BACK
  ─────────────────────────────────────────────────────────
  unshift(0)  →  [0, 1, 2, 3, 4, 5]    ←  push(6) / << 6
  shift       →  removes [0]            ←  pop     (removes 5)

  arr = [1, 2, 3, 4, 5]
  arr.unshift(0)   → [0, 1, 2, 3, 4, 5]
  arr.shift        → returns 0, arr = [1, 2, 3, 4, 5]
  arr.push(6)      → [1, 2, 3, 4, 5, 6]
  arr << 7         → [1, 2, 3, 4, 5, 6, 7]
  arr.pop          → returns 7, arr = [1, 2, 3, 4, 5, 6]
```

---

## Diagram 6 — Hash Merge: Who Wins?

```
  base.merge(override)  →  override wins on conflict
  override.merge(base)  →  base wins on conflict

  base     = { a: 1, b: 2, c: 3 }
  override = {        b: 99,      d: 4 }

  base.merge(override):
  { a: 1,  b: 99,  c: 3,  d: 4 }
            ↑ override wins

  override.merge(base):
  { b: 2,  d: 4,  a: 1,  c: 3 }
    ↑ base wins

  With block:
  base.merge(override) { |key, old, new_val| old }
  { a: 1,  b: 2,  c: 3,  d: 4 }
            ↑ base wins (block returns old value)
```

---

## Diagram 7 — Ranges: Inclusive vs Exclusive

```
  Number line:   1   2   3   4   5   6   7   8   9   10

  (1..10)        ●───────────────────────────────────●
                 inclusive both ends (includes 10)

  (1...10)       ●──────────────────────────────────○
                 excludes right end (includes 9, not 10)

  (1..)          ●───────────────────────────────────→ ∞
                 endless range (Ruby 2.6+)

  (..10)         ∞ ──────────────────────────────────●
                 beginless range (Ruby 2.7+)
```

---

## Diagram 8 — Object Equality: ==, eql?, equal?

```
  a = "hello"
  b = "hello"
  c = a

  MEMORY LAYOUT:
  ┌─────────┐        ┌──────────────────┐
  │  a ─────┼───────→│ String "hello"   │ object_id: 100
  └─────────┘        └──────────────────┘
                              ↑
  ┌─────────┐                 │  (c is alias for same object)
  │  c ─────┼─────────────────┘
  └─────────┘

  ┌─────────┐        ┌──────────────────┐
  │  b ─────┼───────→│ String "hello"   │ object_id: 102
  └─────────┘        └──────────────────┘
                      (different object, same content)

  EQUALITY CHECKS:
  a == b         → true   (same value "hello")
  a.eql?(b)      → true   (same value, same type)
  a.equal?(b)    → false  (different objects)
  a.equal?(c)    → true   (c points to same object as a)
  1 == 1.0       → true   (value equality across types)
  1.eql?(1.0)    → false  (different types: Integer vs Float)
```

---

## Diagram 9 — Method Call: puts vs p vs print

```
  Input:    nil     |  "hello"   |  42    |  [1, nil, 3]
  ─────────────────────────────────────────────────────────
  puts:     (blank) │  hello     │  42    │  1
                    │            │        │  (blank)
                    │            │        │  3
  ─────────────────────────────────────────────────────────
  p:        nil     │  "hello"   │  42    │  [1, nil, 3]
  ─────────────────────────────────────────────────────────
  print:    (empty) │  hello     │  42    │  [1, nil, 3]
                    │ (no \n)    │        │  (no \n)
```

---

## Diagram 10 — Integer Division and Modulo

```
  7 / 2  = 3   (integer result, truncates .5)
  7 % 2  = 1   (remainder)

  Visual:
  ┌─┬─┬─┬─┬─┬─┬─┐
  │1│2│3│4│5│6│7│
  └─┴─┴─┴─┴─┴─┴─┘
  └──────┘└──────┘ └──┘
    group 1  group 2  remainder = 1

  7 = (2 * 3) + 1
       ↑   ↑    ↑
      div  q  remainder

  Float division:
  7.0 / 2  = 3.5  (one operand float = float result)
  7.fdiv(2) = 3.5  (explicit float division method)
```
