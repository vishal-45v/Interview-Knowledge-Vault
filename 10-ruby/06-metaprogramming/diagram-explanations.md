# Chapter 06: Metaprogramming — Diagram Explanations

## Diagram 1: Ruby Method Lookup Chain with `method_missing`

```
obj.some_method  →  Ruby starts dispatch

┌─────────────────────────────────────────────────────────────────┐
│                    METHOD LOOKUP ORDER                          │
│                                                                 │
│  1. obj's singleton class (eigenclass)                         │
│     └── methods defined only for this specific object          │
│                                                                 │
│  2. Prepended modules (in reverse prepend order)               │
│     └── Module prepended LAST appears FIRST                    │
│                                                                 │
│  3. obj.class                                                   │
│     └── regular instance methods defined with def or           │
│         define_method                                           │
│                                                                 │
│  4. Included modules (in reverse include order)                 │
│     └── Module included LAST appears just after class          │
│                                                                 │
│  5. Superclass (repeat steps 2-4 for each ancestor)            │
│     └── walks up to Object → Kernel → BasicObject              │
│                                                                 │
│  ★ NOT FOUND ANYWHERE?                                          │
│     └── method_missing called (follows SAME lookup order)      │
│         └── BasicObject#method_missing raises NoMethodError    │
└─────────────────────────────────────────────────────────────────┘

Example with modules:

  module Loggable     # prepended
  module Serializable # included
  class Animal        # class
  class Dog < Animal  # subclass

  Dog.ancestors:
  ┌─────────────────────────────────────┐
  │  [Loggable,        ← prepend        │
  │   Dog,             ← class itself   │
  │   Serializable,    ← include        │
  │   Animal,          ← superclass     │
  │   Object,                           │
  │   Kernel,          ← included in    │
  │   BasicObject]       Object         │
  └─────────────────────────────────────┘

  dog = Dog.new
  dog.bark?
    │
    ├→ Dog singleton class?      NO
    ├→ Loggable#bark?            NO
    ├→ Dog#bark?                 NO
    ├→ Serializable#bark?        NO
    ├→ Animal#bark?              NO
    ├→ Object#bark?              NO
    ├→ Kernel#bark?              NO
    ├→ BasicObject#bark?         NO
    │
    └→ method_missing called on dog
         └→ if overridden: custom behavior
         └→ if not: BasicObject#method_missing → NoMethodError
```

---

## Diagram 2: Ruby Object Model Internals

```
EVERYTHING IS AN OBJECT IN RUBY

┌─────────────────────────────────────────────────────────────────┐
│                    RUBY OBJECT MODEL                            │
│                                                                 │
│  "hello" (String instance)                                      │
│  ┌───────────────┐                                              │
│  │  RObject      │  ← every Ruby object is an RObject in C     │
│  │  klass ───────┼──→ String class                              │
│  │  ivars table  │    (a Ruby object itself — RClass)           │
│  └───────────────┘                                              │
│                                                                 │
│  String (class = Ruby object of type Class)                     │
│  ┌───────────────────────────┐                                  │
│  │  RClass                   │                                  │
│  │  klass ───────────────────┼──→ Class                         │
│  │  super ───────────────────┼──→ Object                        │
│  │  m_tbl (method table)     │    (String's instance methods)   │
│  │  iv_tbl (class ivars)     │    (@@class_variables)           │
│  └───────────────────────────┘                                  │
│                                                                 │
│  SINGLETON CLASS (eigenclass):                                  │
│  Every object has a hidden singleton class                      │
│  ┌─────────────────────────┐                                    │
│  │ obj                     │                                    │
│  │  klass ─────────────────┼──→ #<Class:obj> (singleton class) │
│  │                         │       klass ──→ String             │
│  │                         │       m_tbl ──→ singleton methods  │
│  └─────────────────────────┘                                    │
└─────────────────────────────────────────────────────────────────┘

CLASS METHOD LOCATION:
  Dog.bark  →  stored in Dog's SINGLETON CLASS, not Dog itself

  ┌──────────────┐      ┌─────────────────────┐
  │   Dog        │      │   #<Class:Dog>       │
  │  (class)     │      │   (singleton class   │
  │              │      │    of Dog)           │
  │  instance    │      │  class methods live  │
  │  methods     │      │  here: Dog.bark      │
  └──────────────┘      └─────────────────────┘
```

---

## Diagram 3: `define_method` vs `def` — Closure Behavior

```
┌───────────────────────────────────────────────────────────┐
│                  def CREATES A NEW SCOPE                  │
│                                                           │
│  x = 10                                                   │
│  class Foo                                                │
│    def bar                                                │
│      x  ← NameError! def creates new scope, x is gone    │
│    end                                                    │
│  end                                                      │
└───────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────┐
│           define_method CLOSES OVER ENCLOSING SCOPE       │
│                                                           │
│  x = 10                                                   │
│  class Foo                                                │
│    define_method(:bar) do                                 │
│      x  ← works! x is captured from enclosing scope      │
│    end                                                    │
│  end                                                      │
│  Foo.new.bar  # => 10                                     │
└───────────────────────────────────────────────────────────┘

PRACTICAL IMPACT IN LOOPS:

  PREFIX = "greet"
  ['hello', 'bye'].each do |word|    ← each creates new binding per iter
    define_method("#{PREFIX}_#{word}") do
      word    ← each iteration captures its own 'word'
    end
  end

  obj.greet_hello  # => "hello"  ✓
  obj.greet_bye    # => "bye"    ✓

  vs. for loop (NO new binding per iteration):

  for word in ['hello', 'bye']       ← for leaks variable
    define_method("say_#{word}") { word }
  end

  obj.say_hello  # => "bye"  ✗ (last value of loop variable)
  obj.say_bye    # => "bye"  ✓
```

---

## Diagram 4: `include` vs `extend` vs `prepend` Visual

```
module M
  def hello = "M"
end

class C
  def hello = "C"
end

─────────────── INCLUDE ───────────────
C.include(M)

Ancestors: [C, M, Object, ...]
Lookup:    C → M → Object

c = C.new
c.hello  # → checks C first → "C" (C wins, M is fallback)

If C doesn't define hello:
c.hello  # → checks C: no → checks M: yes → "M"

─────────────── PREPEND ───────────────
C.prepend(M)

Ancestors: [M, C, Object, ...]
Lookup:    M → C → Object

c = C.new
c.hello  # → checks M first → "M" (M wins, even though C defines it)

M can call super to reach C:
module M
  def hello
    "M wrapping #{super}"   # super calls C#hello
  end
end
c.hello  # => "M wrapping C"

─────────────── EXTEND ────────────────
C.extend(M)

Adds M to C's SINGLETON CLASS ancestors
C's own ancestors: [C, Object, ...]  ← unchanged
C.singleton_class.ancestors: [#<Class:C>, M, #<Class:Object>, ...]

C.hello   # => "M"  (class-level method)
C.new.hello  # NoMethodError (instance method not affected)

─────────────── VISUAL SUMMARY ────────

                ┌─────┐
include(M):     │  C  │ ← instances call C first, then M
                └──┬──┘
                   │
                ┌──▼──┐
                │  M  │
                └─────┘

                ┌─────┐
prepend(M):     │  M  │ ← M is BEFORE C
                └──┬──┘
                   │
                ┌──▼──┐
                │  C  │
                └─────┘

                ┌───────────────────┐
extend(M):      │ C (class object)  │
                │ singleton class   │ ← M added here
                │ has M's methods   │
                └───────────────────┘
```

---

## Diagram 5: `method_missing` Dispatch Flow

```
obj.unknown_method(arg)
         │
         ▼
┌─────────────────────────┐
│   Ruby Method Dispatch   │
│   Walks ancestor chain   │
│   Finds nothing          │
└────────────┬────────────┘
             │ NOT FOUND
             ▼
┌─────────────────────────────────────────────────────┐
│           method_missing lookup begins               │
│                                                      │
│  Walks ancestor chain again, but looking for         │
│  method_missing instead of unknown_method            │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  Did you override method_missing in your class?│  │
│  └───────────────────┬────────────────────────────┘  │
│                      │                                │
│          YES ◄────── ┤ ──────► NO                    │
│           │          │          │                     │
│           ▼          │          ▼                     │
│  Your method_missing │   BasicObject#method_missing   │
│  is called           │   raises NoMethodError         │
│  (name, *args, block)│                                │
└─────────────────────────────────────────────────────┘

CORRECT IMPLEMENTATION:

  def method_missing(name, *args, &block)
    if handles?(name)                    ← check if we handle this
      # custom logic
    else
      super                             ← MUST call super, not raise manually
    end                                  ← super propagates to next in chain
  end

  def respond_to_missing?(name, include_private = false)
    handles?(name) || super            ← MUST match method_missing logic
  end

MISSING respond_to_missing? CONSEQUENCES:

  obj.unknown_method          # works (method_missing handles it)
  obj.respond_to?(:unknown_method)    # false  ← WRONG
  obj.method(:unknown_method)         # NameError
  obj.respond_to_missing?(:unknown_method)  # uses Object's default → false
```

---

## Diagram 6: `send` / `public_send` / `__send__` Decision Tree

```
You want to call a method dynamically:

                ┌──────────────────────────────┐
                │  Is the method name from      │
                │  user/external input?         │
                └──────────────┬───────────────┘
                               │
              YES ◄────────────┤──────────────► NO
               │               │                │
               ▼               │                ▼
  ┌────────────────────┐       │   ┌─────────────────────────┐
  │ Use public_send    │       │   │ Is this inside a proxy   │
  │ + whitelist check  │       │   │ or delegation class?     │
  └────────────────────┘       │   └──────────┬──────────────┘
                               │              │
                               │   YES ◄──────┤──────────────► NO
                               │    │         │                │
                               │    ▼         │                ▼
                               │  __send__    │         send (or public_send
                               │  (proxy-safe)│         if visibility matters)
                               │              │

VISIBILITY RULES:

  ┌─────────────┬──────────────┬────────────────────────────┐
  │  Method     │  Private OK? │  Notes                     │
  ├─────────────┼──────────────┼────────────────────────────┤
  │  send       │  YES         │  Bypasses all visibility   │
  │  __send__   │  YES         │  Cannot be overridden      │
  │  public_send│  NO          │  Respects private/protected│
  │  call via . │  NO          │  obj.method_name           │
  └─────────────┴──────────────┴────────────────────────────┘
```

---

## Diagram 7: Refinements Lexical Scope

```
WITHOUT REFINEMENTS (global monkey patch):

  ┌─────────────────────────────────────────────────────────┐
  │  ENTIRE RUBY PROCESS                                    │
  │                                                         │
  │  class String                                           │
  │    def double = self * 2  ← affects ALL strings        │
  │  end                         everywhere, forever        │
  │                                                         │
  │  GemA: "hello".double  # => "hellohello"               │
  │  GemB: "hello".double  # => "hellohello" (unexpected!) │
  │  YourCode: "hello".double  # => "hellohello"           │
  └─────────────────────────────────────────────────────────┘

WITH REFINEMENTS (lexically scoped):

  ┌─────────────────────────────────────────────────────────┐
  │  ENTIRE RUBY PROCESS                                    │
  │                                                         │
  │  ┌─────────────────────────────────────────────────┐   │
  │  │  module MyLib          using StringDouble        │   │
  │  │    using StringDoubleRefinement                  │   │
  │  │                                                  │   │
  │  │    "hello".double  # => "hellohello" ✓           │   │
  │  │    (refinement ACTIVE in this lexical scope)     │   │
  │  └─────────────────────────────────────────────────┘   │
  │                                                         │
  │  GemA: "hello".double  # NoMethodError ✓ (isolated)    │
  │  GemB: "hello".double  # NoMethodError ✓ (isolated)    │
  │                                                         │
  └─────────────────────────────────────────────────────────┘

  ACTIVATION RULES:
  ┌─────────────────────────────────────────────────────────┐
  │  using SomeRefinement  must appear at:                  │
  │    - Top level of a file                                │
  │    - Inside a module body                               │
  │    - NOT inside a method body (SyntaxError)             │
  │    - NOT inside a class body (works, but not in method) │
  │                                                         │
  │  send bypasses refinements (dynamic dispatch ignored)   │
  │  Direct calls use refinements (static dispatch)         │
  └─────────────────────────────────────────────────────────┘
```
