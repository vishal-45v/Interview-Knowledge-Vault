# Chapter 02 — OOP & Modules: Follow-Up Traps

---

## Trap 1 — private methods cannot be called with explicit receiver (pre-2.7)

```ruby
class Example
  def public_method
    private_method       # OK — implicit self
    self.private_method  # Ruby < 2.7: NoMethodError
                         # Ruby 2.7+: OK (intentional change)
  end

  private

  def private_method
    "secret"
  end
end

# External callers ALWAYS get NoMethodError:
obj = Example.new
obj.private_method       # => NoMethodError always

# Private methods CAN be called via send (bypass for tests):
obj.send(:private_method)  # => "secret" (use sparingly)
```

**Interviewer trap:** "Can you call a private method using send?"
**Answer:** Yes, `send` bypasses visibility. But `public_send` respects it:
```ruby
obj.public_send(:private_method)  # => NoMethodError (respects visibility)
obj.send(:private_method)         # => "secret" (bypasses visibility)
```

---

## Trap 2 — include vs prepend: position in the ancestors chain

```ruby
module M
  def greet
    "M says: #{super}"
  end
end

class C
  def greet
    "C"
  end
end

# With include:
class WithInclude < C; include M; end
WithInclude.ancestors  # => [WithInclude, M, C, ...]
# M is AFTER WithInclude but BEFORE C
# WithInclude#greet would call M#greet first if WithInclude has none
# but WithInclude doesn't define greet, so M#greet is found → calls super → C#greet
WithInclude.new.greet  # => "M says: C"

# With prepend:
class WithPrepend < C; prepend M; end
WithPrepend.ancestors  # => [M, WithPrepend, C, ...]
# M is BEFORE WithPrepend — M#greet wraps WithPrepend#greet
# Even though WithPrepend doesn't define greet, the chain differs
WithPrepend.new.greet  # => "M says: C"

# The REAL difference shows when the class defines the method:
class ConcreteWithPrepend
  prepend M

  def greet
    "ConcreteWithPrepend"
  end
end

ConcreteWithPrepend.ancestors  # => [M, ConcreteWithPrepend, ...]
ConcreteWithPrepend.new.greet  # => "M says: ConcreteWithPrepend"
# M intercepts the call before ConcreteWithPrepend gets it
```

---

## Trap 3 — super without arguments passes ALL arguments automatically

```ruby
class Parent
  def greet(name, greeting: "Hello")
    "#{greeting}, #{name}!"
  end
end

class Child < Parent
  def greet(name, greeting: "Hi")
    super          # passes (name, greeting: greeting) — the CHILD's greeting value
    super(name)    # passes only name, uses Parent's default: "Hello"
    super()        # passes nothing — uses Parent's defaults for all params
  end
end

# The trap: you might intend to call parent with "Hello" default
# but super passes whatever Child received
c = Child.new
c.greet("Alice")  # super sends ("Alice", greeting: "Hi") to Parent
```

**Interviewer follow-up:** "How do you call super with no arguments when the method accepts arguments?"
**Answer:** Use `super()` with explicit empty parens. Plain `super` (no parens) always forwards all current arguments.

---

## Trap 4 — Class variables (@@) are shared across ALL subclasses

```ruby
class Animal
  @@count = 0

  def initialize
    @@count += 1
  end

  def self.count
    @@count
  end
end

class Dog < Animal; end
class Cat < Animal; end

Dog.new
Dog.new
Cat.new
Animal.new

puts Animal.count  # => 4
puts Dog.count     # => 4  (same @@count!)
puts Cat.count     # => 4  (same @@count!)

# Fix: use class instance variables
class Animal
  @count = 0

  def initialize
    self.class.increment_count  # increments the specific class's counter
  end

  def self.increment_count
    @count += 1
  end

  def self.count
    @count
  end
end

class Dog < Animal
  @count = 0
end

class Cat < Animal
  @count = 0
end

Dog.new; Dog.new
Cat.new

Animal.count  # => 0  (Animal's own @count, only direct Animal.new increments it)
Dog.count     # => 2
Cat.count     # => 1
```

---

## Trap 5 — attr_accessor creates instance variables lazily (not eagerly)

```ruby
class User
  attr_accessor :name, :email
  # Does NOT create @name or @email immediately
  # They are created when first ASSIGNED
end

u = User.new
puts u.instance_variables.inspect  # => []  (empty! no vars yet)

u.name = "Alice"
puts u.instance_variables.inspect  # => [:@name]  (now it exists)

# Implications for serialization:
u2 = User.new
u2.to_h rescue nil  # custom to_h might miss unset variables

# Always initialize in initialize for guaranteed state:
class User
  attr_accessor :name, :email

  def initialize(name: nil, email: nil)
    @name  = name
    @email = email
  end
end

User.new.instance_variables  # => [:@name, :@email]  (both exist, even if nil)
```

---

## Trap 6 — protected methods CAN be called by other instances of same class

```ruby
class Money
  def initialize(amount)
    @amount = amount
  end

  def >(other)
    amount > other.amount   # calling protected method on other instance
  end

  def ==(other)
    other.is_a?(Money) && amount == other.amount
  end

  protected

  def amount
    @amount
  end
end

m1 = Money.new(100)
m2 = Money.new(50)

m1 > m2          # => true   (protected: OK from same class)
m1.amount        # => NoMethodError (protected: not from outside)

# Common misconception: protected == "subclass accessible only"
# In Ruby, protected means:
# - Cannot be called from outside
# - CAN be called from same class instance (cross-instance)
# - CAN be called from subclass instances
```

---

## Trap 7 — Module.include? vs Module.included_modules

```ruby
module M; end
module N; end

class A
  include M
end

class B < A
  include N
end

A.include?(M)   # => true
A.include?(N)   # => false  (N is not in A's ancestors)
B.include?(M)   # => true   (inherited through A)
B.include?(N)   # => true

A.included_modules  # => [M, Kernel]  (modules directly included in hierarchy)
B.included_modules  # => [N, M, Kernel]

# include? checks the entire ancestor chain:
B.ancestors  # => [B, N, A, M, Object, Kernel, BasicObject]

# Trap: include? and ancestor?
B.ancestors.include?(A)   # => true  (class in ancestor chain)
B < A                     # => true  (cleaner way to check subclass)
```

---

## Trap 8 — method(:name) returns a bound Method object

```ruby
class Calculator
  def double(n)
    n * 2
  end
end

calc = Calculator.new
m = calc.method(:double)

# m is a Method object, bound to calc
m.call(5)       # => 10
m.(5)           # => 10  (syntactic sugar for .call)
m[5]            # => 10  (also works)

# Convert to Proc (for passing as block):
[1, 2, 3, 4].map(&m)         # => [2, 4, 6, 8]
[1, 2, 3, 4].map(&calc.method(:double))  # same

# Unbound method — not tied to an instance
unbound = Calculator.instance_method(:double)
unbound.call(5)   # => NoMethodError — must bind first!

bound = unbound.bind(Calculator.new)
bound.call(5)     # => 10

# Trap: UnboundMethod is rarely needed directly
# It's useful for decorating/wrapping methods
```

---

## Trap 9 — Modules cannot be instantiated

```ruby
module MyModule
  def useful_method
    "useful"
  end
end

MyModule.new  # => NoMethodError: undefined method 'new' for MyModule:Module

# Modules are for:
# 1. Mixins (include/extend/prepend)
# 2. Namespacing
MyModule::SomeClass  # nested class in module namespace
```

---

## Trap 10 — initialize is always private

```ruby
class MyClass
  def initialize
    @value = 42
  end
end

obj = MyClass.new
obj.initialize   # => NoMethodError: private method 'initialize' called

# Even if you try to make it public:
class MyClass
  public :initialize
  def initialize
    @value = 42
  end
end

# Ruby resets initialize to private — you cannot make it public
# This is a Ruby convention enforced by the language
```

---

## Trap 11 — Inheriting from a class that has a custom new

```ruby
class Singleton
  @instance = nil

  def self.new
    @instance ||= super
  end

  private_class_method :new   # makes MyClass.new private from outside
end

class Child < Singleton; end

# Trap: private_class_method :new is also inherited
Child.new  # => NoMethodError (inherited the private new)

# Fix: define a factory method
class Singleton
  def self.instance
    @instance ||= super()  # super() calls the original Class#new
  end

  private_class_method :new
end
```

---

## Trap 12 — Method defined? vs respond_to? — private methods

```ruby
class Example
  def public_method; end

  private

  def private_method; end
end

e = Example.new

# respond_to? checks PUBLIC methods by default
e.respond_to?(:public_method)   # => true
e.respond_to?(:private_method)  # => false  (private is hidden)

# Second argument true: include private methods
e.respond_to?(:private_method, true)  # => true

# method_defined? on the CLASS checks all methods including private
Example.method_defined?(:private_method)  # => true
Example.public_method_defined?(:private_method)  # => false
Example.private_method_defined?(:private_method) # => true
```
