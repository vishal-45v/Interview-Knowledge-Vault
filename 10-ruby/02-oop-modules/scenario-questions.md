# Chapter 02 — OOP & Modules: Scenario Questions

## Scenario 1 — What does this class print?

```ruby
class Animal
  def speak
    "..."
  end

  def describe
    "I say #{speak}"
  end
end

class Dog < Animal
  def speak
    "Woof"
  end
end

puts Dog.new.describe
```

**Answer:** `I say Woof`. This demonstrates Ruby's late binding (dynamic dispatch). When `describe` (defined in Animal) calls `speak`, it calls `self.speak` on the actual object — which is a Dog. So Dog's `speak` is called even though `describe` is defined in Animal. This is fundamental to how polymorphism works in Ruby.

---

## Scenario 2 — include vs extend

```ruby
module Greetable
  def hello
    "Hello from #{self}"
  end
end

class Person
  include Greetable
end

class Robot
  extend Greetable
end

puts Person.new.hello rescue puts "Person instance: NoMethodError"
puts Person.hello rescue puts "Person class: NoMethodError"
puts Robot.new.hello rescue puts "Robot instance: NoMethodError"
puts Robot.hello rescue puts "Robot class: method found"
```

**Answer:**
```
Hello from #<Person:0x...>  (instance method via include)
Person class: NoMethodError  (include doesn't add class methods)
Robot instance: NoMethodError  (extend doesn't add instance methods)
Hello from Robot  (class method via extend)
```

---

## Scenario 3 — Method Resolution with prepend

```ruby
module Timestamped
  def save
    @saved_at = Time.now
    puts "Timestamping..."
    super
  end
end

class Record
  prepend Timestamped

  def save
    puts "Saving record..."
    "saved"
  end
end

Record.ancestors  # => ?
Record.new.save   # what prints?
```

**Answer:**
```ruby
Record.ancestors
# => [Timestamped, Record, Object, Kernel, BasicObject]
#     ↑ Timestamped is BEFORE Record in the chain
```
Output of `Record.new.save`:
```
Timestamping...
Saving record...
```
`prepend` inserts the module before the class in the MRO. When `save` is called, Timestamped#save runs first, calls `super` which calls Record#save.

---

## Scenario 4 — Class variable inheritance danger

```ruby
class Base
  @@value = "base"

  def self.value
    @@value
  end
end

class Child < Base
  @@value = "child"
end

puts Base.value
puts Child.value
```

**Answer:** Both print `"child"`. Class variables (`@@`) are **shared across the entire inheritance hierarchy**. When Child sets `@@value = "child"`, it modifies the same variable that Base uses. This is why class variables are considered dangerous in Ruby. The fix:

```ruby
# Use class instance variables instead:
class Base
  @value = "base"   # class instance variable (only this class)

  def self.value
    @value
  end
end

class Child < Base
  @value = "child"  # separate from Base's @value
end

Base.value   # => "base"  (not affected by Child)
Child.value  # => "child"
```

---

## Scenario 5 — Private method called with explicit receiver

```ruby
class Person
  def introduce
    puts greet                 # line A
    puts self.greet rescue puts "self.greet failed in old Ruby"   # line B
  end

  private

  def greet
    "Hi, I'm a private method!"
  end
end

Person.new.introduce
Person.new.greet rescue puts "external call failed"
```

**Answer (Ruby 2.7+):**
```
Hi, I'm a private method!   (line A — implicit self, always works)
Hi, I'm a private method!   (line B — self.private in Ruby 2.7+ is allowed)
external call failed         (external callers still cannot call private methods)
```

In Ruby < 2.7, line B would also raise NoMethodError. Ruby 2.7 relaxed this restriction to allow `self.` prefix for private methods within the same class.

---

## Scenario 6 — super without and with arguments

```ruby
class Logger
  def log(message, level: :info)
    puts "[#{level.upcase}] #{message}"
  end
end

class TimestampedLogger < Logger
  def log(message, level: :info)
    super   # What happens?
  end
end

TimestampedLogger.new.log("Server started")
```

**Answer:** `super` without parentheses passes all arguments from the current method call, including keyword arguments. It's equivalent to `super(message, level: level)`. Prints `[INFO] Server started`.

```ruby
# The three super forms:
def log(message, level: :info)
  super              # passes all args: (message, level: level)
  super(message)     # passes only message, drops level keyword
  super()            # passes NO arguments
end
```

---

## Scenario 7 — Comparable with custom class

```ruby
class Box
  include Comparable

  attr_accessor :volume

  def initialize(l, w, h)
    @volume = l * w * h
  end

  def <=>(other)
    volume <=> other.volume
  end
end

boxes = [Box.new(3,3,3), Box.new(2,4,5), Box.new(1,1,10)]
sorted = boxes.sort
puts sorted.map(&:volume).inspect
puts boxes.min.volume
puts boxes.max.volume
puts Box.new(3,3,3).between?(Box.new(1,1,1), Box.new(10,10,10))
```

**Answer:**
```
[10, 27, 40]     # sorted by volume ascending
10               # min volume
40               # max volume
true             # 27 is between 1 and 1000
```

---

## Scenario 8 — Protected methods and cross-instance comparison

```ruby
class Account
  def initialize(balance)
    @balance = balance
  end

  def richer_than?(other)
    balance > other.balance   # can we call other.balance?
  end

  protected

  def balance
    @balance
  end
end

a = Account.new(1000)
b = Account.new(500)
puts a.richer_than?(b)
puts a.balance rescue puts "external access denied"
```

**Answer:**
```
true
external access denied
```

Protected methods can be called from instances of the same class or subclass, but not from external code. This makes them perfect for comparison methods where two instances of the same class need to access each other's internals.

---

## Scenario 9 — Struct equality

```ruby
Point = Struct.new(:x, :y)

p1 = Point.new(1, 2)
p2 = Point.new(1, 2)
p3 = Point.new(3, 4)

puts p1 == p2          # ?
puts p1.equal?(p2)     # ?
puts p1 == p3          # ?
puts p1.class          # ?
puts p1.members.inspect # ?
```

**Answer:**
```
true           # Struct implements == as value equality
false          # different objects in memory
false          # different values
Point          # it's an instance of the Point Struct
[:x, :y]       # members returns the attribute names
```

Struct provides value equality (compares all attributes) for free, which is a key reason to use it for simple data containers.

---

## Scenario 10 — Module method available on instance or class?

```ruby
module Findable
  def find(id)
    "Finding #{self} with id #{id}"
  end
end

class User
  include Findable
end

class Post
  extend Findable
end

puts User.new.find(1) rescue puts "User instance: error"
puts User.find(1) rescue puts "User class: error"
puts Post.new.find(1) rescue puts "Post instance: error"
puts Post.find(1) rescue puts "Post class: error"
```

**Answer:**
```
Finding #<User:0x...> with id 1    (include → instance method)
User class: error                   (include doesn't create class methods)
Post instance: error                (extend doesn't create instance methods)
Finding Post with id 1              (extend → class method, self is Post)
```

---

## Scenario 11 — Reopening a class

```ruby
class String
  def palindrome?
    self == self.reverse
  end
end

puts "racecar".palindrome?
puts "hello".palindrome?
puts "".palindrome?
```

**Answer:**
```
true
false
true
```

Ruby allows "monkey patching" — reopening any class including built-in ones to add methods. This is powerful but can cause compatibility issues. The safer alternative is `Refinements` (Ruby 2.0+) which scope changes to a module.

---

## Scenario 12 — ancestors chain with multiple modules

```ruby
module A
  def who
    "A"
  end
end

module B
  def who
    "B"
  end
end

module C
  def who
    "C"
  end
end

class Base
  include A
  include B
  include C
end

puts Base.new.who
puts Base.ancestors.inspect
```

**Answer:** Prints `C` and:
```
[Base, C, B, A, Object, Kernel, BasicObject]
```

Modules are inserted in reverse include order — last included is checked first. Since C was included last, it appears earliest in the ancestors array (after Base itself) and its `who` method wins.

---

## Scenario 13 — attr_accessor and instance variable laziness

```ruby
class Config
  attr_accessor :timeout

  def initialize
    # Note: @timeout is NOT set here
  end

  def effective_timeout
    @timeout || 30
  end
end

c = Config.new
puts c.timeout.inspect     # ?
puts c.effective_timeout   # ?
c.timeout = 60
puts c.timeout             # ?
puts c.effective_timeout   # ?
```

**Answer:**
```
nil    # @timeout was never set — uninitialized instance var is nil
30     # nil || 30 => 30
60     # set via attr_writer
60     # 60 || 30 => 60 (60 is truthy)
```

---

## Scenario 14 — method(:name) and unbound methods

```ruby
class Greeter
  def hello(name)
    "Hello, #{name}!"
  end
end

g = Greeter.new
m = g.method(:hello)

puts m.call("Alice")
puts m.arity
puts m.class

arr = ["Alice", "Bob", "Carol"]
puts arr.map(&m).inspect
```

**Answer:**
```
Hello, Alice!
1
Method
["Hello, Alice!", "Hello, Bob!", "Hello, Carol!"]
```

`method(:name)` returns a Method object (a first-class function). It can be called with `.call`, converted to a proc with `&`, and passed around. `arity` returns the number of required parameters.

---

## Scenario 15 — Checking module inclusion dynamically

```ruby
module Exportable
  def export
    "Exporting #{self.class}"
  end
end

class Report
  include Exportable
end

class Dashboard; end

objects = [Report.new, Dashboard.new, Report.new]

objects.select { |obj| obj.class.include?(Exportable) }
       .each   { |obj| puts obj.export }
```

**Answer:**
```
Exporting Report
Exporting Report
```

`Class#include?` checks if the module is in the class's ancestors. This is the type-safe way to check capability before calling module-specific methods. The Duck typing alternative is `obj.respond_to?(:export)`.
