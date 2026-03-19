# Chapter 02 — OOP & Modules: Theory Questions

## Class Definition and Initialize

**Q1. How do you define a class in Ruby and what does initialize do?**

```ruby
class BankAccount
  def initialize(owner, balance = 0)
    @owner   = owner
    @balance = balance
  end

  def deposit(amount)
    @balance += amount
  end

  def withdraw(amount)
    raise "Insufficient funds" if amount > @balance
    @balance -= amount
  end

  def to_s
    "#{@owner}'s account: $#{@balance}"
  end
end

account = BankAccount.new("Alice", 1000)
account.deposit(500)
puts account   # => Alice's account: $1500
```

`initialize` is a special method called automatically by `new`. It sets up the initial state of the object (instance variables). It's always `private` — you cannot call `.initialize` directly.

---

## attr_accessor / attr_reader / attr_writer

**Q2. What do attr_accessor, attr_reader, and attr_writer generate?**

```ruby
class Person
  attr_accessor :name    # generates getter AND setter
  attr_reader   :age     # generates getter only
  attr_writer   :email   # generates setter only

  def initialize(name, age, email)
    @name  = name
    @age   = age
    @email = email
  end
end

p = Person.new("Alice", 30, "alice@example.com")

# attr_accessor :name generates:
p.name          # getter: returns @name
p.name = "Bob"  # setter: @name = "Bob"

# attr_reader :age generates:
p.age           # getter only
p.age = 31      # => NoMethodError (no setter generated)

# attr_writer :email generates:
p.email = "bob@example.com"  # setter only
p.email                      # => NoMethodError (no getter generated)
```

Internally, `attr_accessor :name` is equivalent to:
```ruby
def name
  @name
end

def name=(value)
  @name = value
end
```

---

## Inheritance

**Q3. How does inheritance work in Ruby?**

```ruby
class Animal
  attr_reader :name

  def initialize(name)
    @name = name
  end

  def speak
    "..."
  end

  def describe
    "I am #{name} and I say: #{speak}"
  end
end

class Dog < Animal
  def speak
    "Woof!"
  end
end

class Cat < Animal
  def speak
    "Meow!"
  end

  def purr
    "Purrr..."
  end
end

dog = Dog.new("Rex")
dog.describe  # => "I am Rex and I say: Woof!"

cat = Cat.new("Whiskers")
cat.describe  # => "I am Whiskers and I say: Meow!"
cat.purr      # => "Purrr..."
dog.purr      # => NoMethodError
```

**Q4. What does the super keyword do?**

```ruby
class Vehicle
  def initialize(make, model)
    @make  = make
    @model = model
  end

  def info
    "#{@make} #{@model}"
  end
end

class Car < Vehicle
  def initialize(make, model, year)
    super(make, model)    # calls Vehicle#initialize with these args
    @year = year
  end

  def info
    "#{@year} #{super}"   # calls Vehicle#info, prepends year
  end
end

car = Car.new("Toyota", "Camry", 2023)
car.info  # => "2023 Toyota Camry"

# super without arguments passes ALL arguments from current method:
class Child < Parent
  def greet(name)
    super      # passes name automatically to Parent#greet
    super()    # calls Parent#greet with NO arguments
    super(name)  # explicit — same as super without parens
  end
end
```

---

## Method Lookup Chain (MRO)

**Q5. Explain Ruby's Method Resolution Order (MRO)**

```ruby
module Swimmable
  def swim
    "#{self.class} is swimming"
  end
end

module Flyable
  def fly
    "#{self.class} is flying"
  end
end

class Animal
  def breathe
    "breathing"
  end
end

class Duck < Animal
  include Swimmable
  include Flyable

  def quack
    "Quack!"
  end
end

Duck.ancestors
# => [Duck, Flyable, Swimmable, Animal, Object, Kernel, BasicObject]
#     ↑ own  ↑ last included first  ↑ superclass ...

# Method lookup: Ruby walks ancestors left to right until it finds the method
duck = Duck.new
duck.fly      # found in Flyable
duck.swim     # found in Swimmable
duck.breathe  # found in Animal
duck.class    # found in Object
```

---

## Modules: include vs extend vs prepend

**Q6. What is the difference between include, extend, and prepend?**

```ruby
module Greetable
  def greet
    "Hello from #{self}"
  end
end

# include — adds module methods as INSTANCE methods
class Person
  include Greetable
end
Person.new.greet    # => "Hello from #<Person:...>"
Person.greet        # => NoMethodError (not a class method)

# extend — adds module methods as CLASS methods (singleton methods)
class Robot
  extend Greetable
end
Robot.greet         # => "Hello from Robot"
Robot.new.greet     # => NoMethodError (not an instance method)

# Also: extend on an instance adds methods to just that one object
alice = Person.new
alice.extend(Greetable)
alice.greet  # works for alice only

# prepend — inserts module BEFORE the class in the ancestors chain
module Logging
  def greet
    puts "Calling greet..."
    result = super    # calls the class's greet
    puts "Done."
    result
  end
end

class Employee
  prepend Logging

  def greet
    "Hello, I'm an employee"
  end
end

Employee.ancestors
# => [Logging, Employee, Object, ...]
#     ↑ prepended BEFORE the class

Employee.new.greet
# Calling greet...
# Done.
# => "Hello, I'm an employee"
```

**Q7. What is a mixin and how does it differ from inheritance?**

```ruby
# Inheritance models IS-A relationships
class Dog < Animal   # Dog IS-A Animal

# Mixins (include) model HAS-A / CAN-DO capabilities
module Serializable
  def to_json
    require 'json'
    instance_variables.each_with_object({}) { |var, h|
      h[var.to_s.delete('@')] = instance_variable_get(var)
    }.to_json
  end
end

module Auditable
  def audit_log
    "#{self.class} changed at #{Time.now}"
  end
end

class User < ActiveRecord::Base
  include Serializable
  include Auditable
end

class Product < ActiveRecord::Base
  include Serializable  # can reuse without copying code
end
```

Mixins solve the "composition over inheritance" problem. Ruby only allows single inheritance (`<`) but multiple module inclusion.

---

## self in Different Contexts

**Q8. What does self refer to in different contexts?**

```ruby
class Counter
  @@total = 0

  def initialize
    @@total += 1
    @id = @@total
  end

  # Inside an instance method, self is the instance
  def describe
    self.class   # => Counter
    self         # => the Counter object
    self.object_id
  end

  # Inside a class method, self is the class itself
  def self.total
    @@total   # self is Counter here
  end

  # Alternative class method definition using self
  class << self
    def reset
      @@total = 0
    end
  end
end

Counter.new
Counter.new
Counter.total  # => 2 (self.total is called on the class)
```

---

## Method Visibility

**Q9. What are public, protected, and private methods?**

```ruby
class BankAccount
  def initialize(balance)
    @balance = balance
  end

  # Public — accessible from anywhere (default)
  def deposit(amount)
    @balance += validate_amount(amount)
  end

  # Protected — accessible from instances of same class/subclass
  # Used for internal comparisons between instances
  def >(other)
    balance > other.balance
  end

  # Private — only callable without explicit receiver (within the object)
  private

  def validate_amount(amount)
    raise ArgumentError, "Amount must be positive" unless amount.positive?
    amount
  end

  protected

  def balance
    @balance
  end
end

a1 = BankAccount.new(1000)
a2 = BankAccount.new(500)

a1.deposit(100)         # public — works
a1 > a2                 # protected — works (called on instance)
a1.balance              # => NoMethodError (protected, called from outside)
a1.send(:validate_amount, 50)  # private — bypassed with send (but bad practice)
```

**Q10. Why can't private methods be called with an explicit receiver?**

```ruby
class Example
  private

  def secret
    "shh"
  end

  def reveal
    secret        # works — implicit self
    self.secret   # => NoMethodError in Ruby < 2.7
                  # => works in Ruby 2.7+ for private methods
  end
end

# Ruby 2.7+ changed this: self.private_method is allowed
# but external callers still cannot call it
obj = Example.new
obj.secret        # => NoMethodError always
```

---

## Duck Typing

**Q11. What is duck typing and how does Ruby support it?**

```ruby
# Duck typing: "If it walks like a duck and quacks like a duck, it's a duck"
# Don't check what type an object IS — check what it CAN DO

# Bad: type checking
def process(collection)
  raise TypeError unless collection.is_a?(Array)
  collection.each { |item| puts item }
end

# Good: duck typing
def process(collection)
  collection.each { |item| puts item }
  # works for Array, Range, Hash, any Enumerable!
end

# Even better: explicit capability check
def process(collection)
  raise ArgumentError, "must respond to each" unless collection.respond_to?(:each)
  collection.each { |item| puts item }
end

process([1, 2, 3])      # works
process(1..10)           # works
process({ a: 1, b: 2 }) # works — hashes respond to each

# Real-world duck typing in Ruby standard library:
def write_to(output, data)
  output.puts(data)   # works on $stdout, StringIO, File, any IO
end
```

---

## Comparable Module

**Q12. How do you use the Comparable module?**

```ruby
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  # You MUST define <=> to use Comparable
  def <=>(other)
    degrees <=> other.degrees
  end

  def to_s
    "#{degrees}°"
  end
end

temps = [Temperature.new(100), Temperature.new(37), Temperature.new(0)]
temps.sort              # => [0°, 37°, 100°]
temps.min               # => 0°
temps.max               # => 100°

t1 = Temperature.new(37)
t2 = Temperature.new(100)
t1 < t2                 # => true   (from Comparable)
t1 > t2                 # => false
t1 <= t2                # => true
t1.between?(Temperature.new(0), Temperature.new(50))  # => true
temps.sort.first        # => 0°
```

---

## Enumerable Module

**Q13. What does including Enumerable in a class give you?**

```ruby
class WordCollection
  include Enumerable

  def initialize
    @words = []
  end

  def add(word)
    @words << word
    self
  end

  # You MUST define each to use Enumerable
  def each(&block)
    @words.each(&block)
  end
end

wc = WordCollection.new
wc.add("hello").add("world").add("ruby").add("programming")

# Enumerable provides ALL these for free:
wc.map(&:upcase)          # => ["HELLO", "WORLD", "RUBY", "PROGRAMMING"]
wc.select { |w| w.length > 4 }  # => ["hello", "world", "programming"]
wc.sort                   # => ["hello", "programming", "ruby", "world"]
wc.include?("ruby")       # => true
wc.count                  # => 4
wc.min                    # => "hello"  (alphabetical)
wc.max                    # => "world"
wc.first                  # => "hello"
wc.to_a                   # => ["hello", "world", "ruby", "programming"]
```

---

## Struct

**Q14. What is a Struct and when would you use it?**

```ruby
# Struct — lightweight value object with named attributes
Point = Struct.new(:x, :y)

p = Point.new(3, 4)
p.x         # => 3
p.y         # => 4
p.to_a      # => [3, 4]
p == Point.new(3, 4)  # => true (value equality built-in)
p.members   # => [:x, :y]

# Struct with custom methods
Point = Struct.new(:x, :y) do
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end

  def to_s
    "(#{x}, #{y})"
  end
end

a = Point.new(0, 0)
b = Point.new(3, 4)
a.distance_to(b)  # => 5.0

# Struct vs plain class:
# Use Struct for simple value objects (coordinates, RGB colors, etc.)
# Use a full class when you need more complex initialization or behavior

# OpenStruct — dynamic attributes (slower, avoid in hot paths)
require 'ostruct'
person = OpenStruct.new(name: "Alice", age: 30)
person.name    # => "Alice"
person.city    # => nil   (new attributes can be added dynamically)
person.city = "NYC"
person.city    # => "NYC"
```

---

## Object Equality

**Q15. How does Ruby determine if two objects are equal?**

```ruby
# For custom classes, define == for value equality
class Money
  attr_reader :amount, :currency

  def initialize(amount, currency)
    @amount   = amount
    @currency = currency
  end

  def ==(other)
    other.is_a?(Money) &&
      amount == other.amount &&
      currency == other.currency
  end

  # If you define ==, also define eql? and hash for Hash/Set use
  def eql?(other)
    self == other
  end

  def hash
    [amount, currency].hash
  end
end

m1 = Money.new(100, "USD")
m2 = Money.new(100, "USD")
m3 = Money.new(200, "USD")

m1 == m2   # => true   (same value)
m1 == m3   # => false
m1.equal?(m2)  # => false  (different objects)

# Hash/Set correctness requires eql? and hash:
require 'set'
s = Set.new([m1, m2])
s.size  # => 1  (m1 and m2 are considered equal if eql? and hash agree)
```

---

## Module Introspection

**Q16. How do you inspect a class's module hierarchy?**

```ruby
module Walkable; end
module Swimmable; end

class Animal; end
class Duck < Animal
  include Walkable
  include Swimmable
end

Duck.ancestors
# => [Duck, Swimmable, Walkable, Animal, Object, Kernel, BasicObject]

Duck.include?(Swimmable)   # => true
Duck.include?(Walkable)    # => true
Duck.include?(Comparable)  # => false

Duck.instance_methods(false)   # methods defined on Duck only
Duck.instance_methods(true)    # all methods including inherited

Duck.method_defined?(:swim)    # => true (instance method)
Duck.respond_to?(:new)         # => true (class method)

# Check ancestors:
Duck < Animal    # => true  (Duck is a subclass of Animal)
Animal < Duck    # => false
Duck < Duck      # => false  (not strictly less than itself)
Duck <= Duck     # => true   (subclass or same)
```

---

## class << self (Singleton Class)

**Q17. What is the singleton class (eigenclass) in Ruby?**

```ruby
# Every Ruby object has a hidden singleton class
# It holds methods defined on that specific object only

class Dog
  # Class methods can be defined three ways:

  # Way 1: def self.method_name
  def self.description
    "Dogs are loyal companions"
  end

  # Way 2: class << self block (opens the singleton class)
  class << self
    def count
      @@count ||= 0
    end

    def bark_all
      "All dogs: Woof!"
    end
  end
end

# Way 3: define on the class object after definition
def Dog.fetch
  "#{self} fetches!"
end

Dog.description   # => "Dogs are loyal companions"
Dog.count         # => 0
Dog.bark_all      # => "All dogs: Woof!"

# Adding a method to a specific INSTANCE only:
spot = Dog.new
fido = Dog.new

def spot.special_trick
  "Spot can skateboard!"
end

spot.special_trick  # => "Spot can skateboard!"
fido.special_trick  # => NoMethodError
```

---

## method_defined? and instance_methods

**Q18. How do you check if a method exists on a class?**

```ruby
class Calculator
  def add(a, b)
    a + b
  end

  private

  def validate(n)
    raise unless n.is_a?(Numeric)
  end
end

Calculator.method_defined?(:add)       # => true  (public instance method)
Calculator.method_defined?(:validate)  # => true  (private also counts)
Calculator.method_defined?(:divide)    # => false

Calculator.public_method_defined?(:add)       # => true
Calculator.private_method_defined?(:validate) # => true
Calculator.public_method_defined?(:validate)  # => false

# instance_methods
Calculator.instance_methods(false)   # => [:add]  (defined on Calculator only)
Calculator.private_instance_methods(false)  # => [:validate]
```
