# Chapter 02 — OOP & Modules: Structured Answers

---

## Answer 1 — "Explain Ruby's Method Resolution Order (MRO) in detail"

Ruby's MRO determines which method implementation is used when the same method name exists in multiple places. Ruby uses C3 linearization (same as Python 3) to flatten the ancestor hierarchy into an ordered list.

```ruby
module Walkable
  def move
    "walking"
  end
end

module Swimmable
  def move
    "swimming"
  end
end

class Animal
  def breathe
    "breathing"
  end
end

class Duck < Animal
  include Walkable
  include Swimmable  # included LAST = checked FIRST among modules

  def quack
    "Quack"
  end
end

Duck.ancestors
# => [Duck, Swimmable, Walkable, Animal, Object, Kernel, BasicObject]

# Rule: modules are inserted in reverse include order
# Last included = closest to the class = highest priority
Duck.new.move  # => "swimming" (Swimmable wins because it was included last)
```

**The algorithm:** When a method is called on an object:
1. Start with the object's own class (Duck)
2. Check each module mixed into Duck (in reverse include order)
3. Move up to the superclass (Animal)
4. Repeat for each level up to BasicObject

```ruby
# prepend changes the order — module goes BEFORE the class:
module Logging
  def save
    puts "before save"
    super
    puts "after save"
  end
end

class Record
  prepend Logging
  def save; puts "saving"; end
end

Record.ancestors  # => [Logging, Record, Object, ...]
Record.new.save
# before save
# saving
# after save
```

**Interview insight:** "Understanding MRO lets you predict which method wins and design module hierarchies intentionally — it's essential for Rails concerns, ActiveSupport::Concern, and any plugin architecture."

---

## Answer 2 — "Explain the difference between include, extend, and prepend with real use cases"

```ruby
# include — adds module methods as INSTANCE methods
# Use case: shared behavior across multiple classes

module Serializable
  def to_json
    require 'json'
    instance_variables.each_with_object({}) { |v, h|
      h[v.to_s.sub('@', '')] = instance_variable_get(v)
    }.to_json
  end
end

class User
  include Serializable
  def initialize(name); @name = name; end
end

User.new("Alice").to_json  # => '{"name":"Alice"}'


# extend — adds module methods as CLASS (singleton) methods
# Use case: class-level functionality like finders, factories

module Findable
  def find(id)
    all.detect { |record| record.id == id }
  end

  def where(**conditions)
    all.select { |r| conditions.all? { |k, v| r.send(k) == v } }
  end
end

class User
  extend Findable
  # User.find(1), User.where(role: "admin") now available
end


# prepend — inserts module BEFORE the class in the MRO
# Use case: transparent method decoration (logging, caching, validation)

module Cacheable
  def find(id)
    cache_key = "#{self.class.name.downcase}:#{id}"
    @cache    ||= {}
    @cache[cache_key] ||= super
  end
end

class UserRepository
  prepend Cacheable

  def find(id)
    puts "Hitting database for id=#{id}"
    User.new(id)  # simulate DB call
  end
end

repo = UserRepository.new
repo.find(1)  # hits DB
repo.find(1)  # returns cached result
```

---

## Answer 3 — "What is duck typing and why is it important in Ruby?"

Duck typing is a programming philosophy where an object's capabilities matter more than its type. The name comes from "if it walks like a duck and quacks like a duck, it is a duck."

```ruby
# Type-checking approach (un-Rubyish):
def process_payment(payment)
  case payment
  when CreditCard
    payment.charge
  when BankTransfer
    payment.transfer
  when Cryptocurrency
    payment.broadcast
  end
end

# Duck typing approach (Rubyish):
def process_payment(payment)
  # Just call the method — the object knows how to handle it
  payment.execute
end

# Now ANY object that responds to execute works:
class CreditCard
  def execute; charge_card; end
end

class PayPal
  def execute; api_transfer; end
end

class FuturePaymentMethod
  def execute; do_something_new; end
end
# No changes needed to process_payment!
```

**Practical Ruby duck typing:**
```ruby
# IO duck typing — write to anything that responds to <<
def log(output, message)
  output << "#{Time.now}: #{message}\n"
end

log($stdout, "Server started")    # writes to stdout
log(File.open("app.log", "a"), "Server started")  # writes to file
log(StringIO.new, "Server started")  # writes to string buffer

# Enumerable duck typing
def sum_all(collection)
  collection.sum   # works for Array, Range, Set, custom Enumerable
end

sum_all([1,2,3])   # => 6
sum_all(1..10)     # => 55
```

---

## Answer 4 — "How do you implement a proper value object in Ruby?"

A value object is an object defined by its attributes, not its identity. Two value objects with the same attributes should be equal.

```ruby
class Money
  include Comparable

  attr_reader :amount, :currency

  def initialize(amount, currency = "USD")
    @amount   = BigDecimal(amount.to_s)
    @currency = currency.upcase.freeze
    freeze  # value objects should be immutable
  end

  # Value equality based on amount and currency
  def ==(other)
    other.is_a?(Money) &&
      amount == other.amount &&
      currency == other.currency
  end

  alias eql? ==

  # hash must be consistent with eql? for Hash/Set correctness
  def hash
    [amount, currency].hash
  end

  # Comparable: only compare same currency
  def <=>(other)
    raise TypeError, "Cannot compare #{currency} with #{other.currency}" \
      unless currency == other.currency
    amount <=> other.amount
  end

  def +(other)
    raise TypeError, "Currency mismatch" unless currency == other.currency
    Money.new(amount + other.amount, currency)
  end

  def to_s
    "#{currency} #{amount.to_f.round(2)}"
  end

  def inspect
    "#<Money #{self}>"
  end
end

price    = Money.new(9.99, "USD")
tax      = Money.new(0.80, "USD")
total    = price + tax
puts total   # => USD 10.79

# Value equality:
Money.new(10, "USD") == Money.new(10, "USD")  # => true
Money.new(10, "USD").equal?(Money.new(10, "USD"))  # => false (different objects)

# Works in sets/hashes correctly:
require 'set'
prices = Set.new([Money.new(10), Money.new(10), Money.new(20)])
prices.size  # => 2 (duplicates removed correctly)
```

---

## Answer 5 — "Explain the Comparable module and the spaceship operator"

```ruby
# The spaceship operator <=> is the foundation of all comparison
# It should return:
#  -1 (or negative) if self < other
#   0 if self == other
#   1 (or positive) if self > other
#  nil if comparison is not possible

class Version
  include Comparable

  attr_reader :major, :minor, :patch

  def initialize(version_string)
    parts = version_string.split('.').map(&:to_i)
    @major = parts[0] || 0
    @minor = parts[1] || 0
    @patch = parts[2] || 0
  end

  def <=>(other)
    return nil unless other.is_a?(Version)
    result = major <=> other.major
    result = minor <=> other.minor if result == 0
    result = patch <=> other.patch if result == 0
    result
  end

  def to_s
    "#{major}.#{minor}.#{patch}"
  end
end

versions = ["2.1.0", "1.9.3", "3.0.1", "2.7.5"].map { |v| Version.new(v) }
versions.sort.map(&:to_s)
# => ["1.9.3", "2.1.0", "2.7.5", "3.0.1"]

v = Version.new("2.7.0")
v > Version.new("2.6.9")   # => true
v < Version.new("3.0.0")   # => true
v.between?(Version.new("2.0.0"), Version.new("3.0.0"))  # => true
```

---

## Answer 6 — "When would you use a Struct instead of a plain class?"

```ruby
# Use Struct for:
# 1. Simple data containers with no complex behavior
# 2. Value objects where == compares attributes
# 3. Quick prototyping

# Plain class (when you need complex initialization/validation):
class User
  attr_reader :name, :email

  def initialize(name:, email:)
    raise ArgumentError, "Invalid email" unless email.include?("@")
    @name  = name.strip
    @email = email.downcase
  end
end

# Struct (when it's just data):
Coordinate = Struct.new(:lat, :lng) do
  def distance_to(other)
    # Haversine formula simplified
    Math.sqrt((lat - other.lat)**2 + (lng - other.lng)**2) * 111  # km approx
  end

  def to_s
    "(#{lat}, #{lng})"
  end
end

NYC   = Coordinate.new(40.7128, -74.0060)
LA    = Coordinate.new(34.0522, -118.2437)
dist  = NYC.distance_to(LA)

# Struct gives you for free:
# - Attribute readers
# - == based on attribute values
# - to_a (array of values)
# - to_h (hash of name => value)
# - members (list of attribute names)
NYC.to_h     # => {lat: 40.7128, lng: -74.006}
NYC == Coordinate.new(40.7128, -74.0060)  # => true

# Ruby 3.2+: Data class (immutable Struct)
Point = Data.define(:x, :y)
p = Point.new(x: 3, y: 4)
p.x          # => 3
p.with(x: 5) # => Point(x: 5, y: 4)  creates new with updated field
```

---

## Answer 7 — "How do you design a module-based mixin strategy?"

```ruby
# Good mixin design: small, focused, composable

module Persistable
  def save
    store = self.class.store
    store[id] = self
    true
  end

  def delete
    self.class.store.delete(id)
  end

  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@store, {})
  end

  module ClassMethods
    def store
      @store
    end

    def find(id)
      @store[id]
    end

    def all
      @store.values
    end
  end
end

module Validatable
  def valid?
    errors.empty?
  end

  def errors
    @errors ||= []
  end

  def validate!
    raise "Invalid: #{errors.join(', ')}" unless valid?
    self
  end
end

module Auditable
  def created_at
    @created_at ||= Time.now
  end

  def updated_at
    @updated_at
  end

  def touch
    @updated_at = Time.now
  end
end

# Compose capabilities with include:
class User
  include Persistable
  include Validatable
  include Auditable

  attr_accessor :name, :email

  def initialize(id:, name:, email:)
    @id    = id
    @name  = name
    @email = email
  end

  def id; @id; end

  def valid?
    errors.clear
    errors << "Name is required"  if name.nil? || name.empty?
    errors << "Email is required" if email.nil? || email.empty?
    super  # calls Validatable's valid? which checks errors
  end
end
```
