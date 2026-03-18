# Chapter 02 — OOP & Modules: Diagram Explanations

---

## Diagram 1 — Method Resolution Order (MRO) Chain

```
  class Duck < Animal
    include Walkable    # included first
    include Swimmable   # included second (higher priority)
  end

  Duck.ancestors:
  ┌─────────────────────────────────────────────────────────────────┐
  │  Duck → Swimmable → Walkable → Animal → Object → Kernel →      │
  │          BasicObject                                            │
  └─────────────────────────────────────────────────────────────────┘
  ↑ Start here       ↑ Last included     ↑ superclass
                       = higher priority

  Method lookup for duck.move:
  Duck      — defines move? NO
  Swimmable — defines move? YES → execute Swimmable#move, STOP
```

---

## Diagram 2 — include vs extend vs prepend

```
  MODULE M  { def hello = "hello from M" }

  ─────────────────────────────────────────────────────────────────

  INCLUDE:  adds M to instance method lookup AFTER the class

    Ancestors:  [MyClass, M, Object, ...]

    MyClass.new.hello   ✓  (instance has it)
    MyClass.hello       ✗  (class doesn't have it)

  ─────────────────────────────────────────────────────────────────

  EXTEND:  adds M to the CLASS's singleton class

    MyClass's singleton: [M, ...]

    MyClass.new.hello   ✗  (instance doesn't have it)
    MyClass.hello       ✓  (class has it)

  ─────────────────────────────────────────────────────────────────

  PREPEND:  adds M BEFORE the class in the lookup chain

    Ancestors:  [M, MyClass, Object, ...]

    MyClass.new.hello   ✓  (M intercepts the call first)
    M#hello can call super to reach MyClass#hello
```

---

## Diagram 3 — The Full Ancestor Chain

```
  BasicObject         ← absolute root (only 8 methods)
      │
    Object            ← includes Kernel (gives puts, p, require, etc.)
      │
    MyParentClass     ← your app's parent class
      │
    ┌─────────────────────────────────┐
    │  include ModA  (added before MyClass in chain)
    │  include ModB  (added before ModA — closer to class)
    └─────────────────────────────────┘
      │
    MyClass           ← your class
      │
    ┌────────────┐
    │  prepend ModC  (inserted BEFORE MyClass)
    └────────────┘

  Final ancestors: [ModC, MyClass, ModB, ModA, MyParentClass, Object, Kernel, BasicObject]
  Lookup goes left to right
```

---

## Diagram 4 — Method Visibility: Who Can Call What

```
  class MyClass
    def public_method    # anyone can call
    protected
    def shared_method    # same class instances only
    private
    def internal_method  # only self (implicit receiver)
  end

  ┌──────────────────────────────────────────────────────────────┐
  │  Caller                │ public │ protected │ private        │
  ├──────────────────────────────────────────────────────────────┤
  │  External code         │   ✓    │     ✗     │    ✗           │
  │  Same class instance   │   ✓    │     ✓     │    ✓           │
  │  Subclass instance     │   ✓    │     ✓     │    ✗           │
  │  Other instance of     │   ✓    │     ✓     │    ✗           │
  │    same class          │        │           │                │
  └──────────────────────────────────────────────────────────────┘

  Protected is ideal for cross-instance comparison methods:
    def >(other)
      balance > other.balance   # other.balance is protected — works!
    end
```

---

## Diagram 5 — Class vs Instance Variables

```
  class BankAccount
    @@total_accounts = 0          ─── class variable (shared across all)

    def initialize(owner)
      @owner   = owner            ─── instance variable (per object)
      @balance = 0                ─── instance variable (per object)
      @@total_accounts += 1
    end
  end

  MEMORY LAYOUT:
  ┌─────────────────────────────────┐
  │  BankAccount (Class object)     │
  │  @@total_accounts = 2           │  ← one copy for all
  └─────────────────────────────────┘
           ↓ new          ↓ new
  ┌──────────────────┐  ┌──────────────────┐
  │ account1         │  │ account2         │
  │ @owner = "Alice" │  │ @owner = "Bob"   │
  │ @balance = 500   │  │ @balance = 1000  │
  └──────────────────┘  └──────────────────┘
  (each instance has its OWN @variables)
```

---

## Diagram 6 — Class Instance Variable vs Class Variable

```
  CLASS VARIABLE @@  — shared with ALL subclasses:

    Animal  ←──── @@count (shared!)
      ├── Dog   reads/writes the SAME @@count
      └── Cat   reads/writes the SAME @@count


  CLASS INSTANCE VARIABLE @ on class — separate per class:

    Animal  @count = 0  (Animal's own counter)
      ├── Dog   @count = 0  (Dog's own counter)
      └── Cat   @count = 0  (Cat's own counter)

  ┌────────────────────────────────────────────────────────────┐
  │ Animal.count  → @count on Animal class object              │
  │ Dog.count     → @count on Dog class object (different)     │
  │ Cat.count     → @count on Cat class object (different)     │
  └────────────────────────────────────────────────────────────┘
```

---

## Diagram 7 — attr_accessor Internals

```
  attr_accessor :name
  generates:

  ┌─────────────────────────────────────┐
  │  def name          ← getter method  │
  │    @name                            │
  │  end                                │
  │                                     │
  │  def name=(value)  ← setter method  │
  │    @name = value                    │
  │  end                                │
  └─────────────────────────────────────┘

  attr_reader :age  →  generates getter only
  attr_writer :email → generates setter only
```

---

## Diagram 8 — Struct vs Class vs Hash

```
  DATA CONTAINER COMPARISON:

  ┌──────────────┬──────────┬───────────────┬───────────────┐
  │              │  Hash    │  Struct       │  Class        │
  ├──────────────┼──────────┼───────────────┼───────────────┤
  │ Named fields │ No (keys)│ YES           │ YES           │
  │ Type check   │ No       │ No            │ YES (is_a?)   │
  │ Value ==     │ YES      │ YES (built-in)│ Custom needed │
  │ Methods      │ Built-in │ Custom + auto │ Full control  │
  │ Immutable    │ Optional │ Optional      │ Optional      │
  │ Performance  │ Fast     │ Fast          │ Normal        │
  │ Use when     │ Dynamic  │ Simple data   │ Complex logic │
  └──────────────┴──────────┴───────────────┴───────────────┘
```

---

## Diagram 9 — Module Namespace vs Mixin

```
  MODULES SERVE TWO ROLES:

  1. NAMESPACE (organize code, avoid name clashes):

     module Payments
       class CreditCard   ← Payments::CreditCard
         def charge; end
       end

       class BankTransfer  ← Payments::BankTransfer
         def transfer; end
       end
     end

  2. MIXIN (share behavior):

     module Auditable
       def audit_log
         "#{self.class} changed"
       end
     end

     class User
       include Auditable  ← User instances get audit_log
     end

  The same module can be BOTH:
  module Payments
    module Auditable        ← namespace
      def audit_log; end    ← mixin behavior
    end

    class CreditCard
      include Auditable     ← mixing in Payments::Auditable
    end
  end
```
