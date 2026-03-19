# Chapter 06: Metaprogramming — Theory Questions

## Core Metaprogramming Mechanics

**Q1. What is metaprogramming in Ruby, and why does Ruby excel at it compared to other languages?**

Metaprogramming is writing code that writes or modifies code at runtime. Ruby excels because everything is an object (including classes), methods can be defined dynamically, the object model is open and mutable, and the language exposes its own internals via reflection APIs. Unlike Java or C#, Ruby has no compile-time finality — classes can be reopened, methods added or removed, and the method dispatch chain can be intercepted.

---

**Q2. Explain `define_method` and contrast it with `def`.**

`define_method` is a method on `Module` that creates an instance method programmatically. Unlike `def`, it closes over its surrounding scope (it is a closure), allowing you to capture local variables at definition time.

```ruby
class Greeter
  ['hello', 'goodbye'].each do |word|
    define_method("say_#{word}") do |name|
      "#{word.capitalize}, #{name}!"
    end
  end
end

g = Greeter.new
g.say_hello('Alice')   # => "Hello, Alice!"
g.say_goodbye('Alice') # => "Goodbye, Alice!"
```

`def` creates a new scope; `define_method` captures the enclosing local scope.

---

**Q3. What is `method_missing` and when should you use it?**

`method_missing` is called by Ruby's method dispatch when no matching method is found in the object's class hierarchy. You override it to intercept undefined method calls and provide dynamic behavior.

```ruby
class DynamicProxy
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    if @target.respond_to?(name)
      @target.send(name, *args, &block)
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private) || super
  end
end

proxy = DynamicProxy.new([1, 2, 3])
proxy.length  # => 3
proxy.map { |x| x * 2 }  # => [2, 4, 6]
```

Use it for proxies, DSLs, and dynamic finders. Always pair with `respond_to_missing?`.

---

**Q4. Why must `respond_to_missing?` always accompany `method_missing`?**

Without `respond_to_missing?`, `respond_to?` returns `false` for dynamically handled methods, breaking duck typing, `method(:name)`, and any code that checks interface compatibility before calling.

```ruby
class Flexible
  def method_missing(name, *args)
    return "handled: #{name}" if name.to_s.start_with?('magic_')
    super
  end

  def respond_to_missing?(name, include_private = false)
    name.to_s.start_with?('magic_') || super
  end
end

f = Flexible.new
f.magic_foo          # => "handled: magic_foo"
f.respond_to?(:magic_foo)  # => true  (correct only with respond_to_missing?)
f.method(:magic_foo)       # => #<Method: Flexible#magic_foo>
```

---

**Q5. Explain the difference between `send`, `public_send`, and `__send__`.**

- `send` — calls any method including private and protected ones.
- `public_send` — only calls public methods; raises `NoMethodError` for private/protected.
- `__send__` — same as `send` but immune to being overridden (safe in proxies).

```ruby
class Vault
  private

  def secret
    "classified"
  end
end

v = Vault.new
v.send(:secret)         # => "classified"   (bypasses private)
v.public_send(:secret)  # => NoMethodError: private method 'secret' called
v.__send__(:secret)     # => "classified"   (like send, but cannot be redefined)
```

Use `public_send` when the method name comes from user input to avoid security bypasses.

---

**Q6. What is `class_eval` (alias `module_eval`) and how does it differ from `instance_eval`?**

- `class_eval` opens the class/module and evaluates a block in its context — methods defined inside become **instance methods** of that class.
- `instance_eval` evaluates in the context of a specific object — methods defined inside become **singleton methods** of that object.

```ruby
# class_eval — defines instance methods
String.class_eval do
  def shout
    upcase + '!!!'
  end
end
"hello".shout  # => "HELLO!!!"

# instance_eval — defines singleton method on one object
obj = Object.new
obj.instance_eval do
  def greet
    "I am a singleton method"
  end
end
obj.greet  # => "I am a singleton method"
Object.new.greet  # => NoMethodError
```

---

**Q7. What are Ruby open classes (monkey patching) and what are the risks?**

Open classes mean any class can be reopened at any time, in any file, and methods added, removed, or replaced.

```ruby
class Integer
  def factorial
    return 1 if self <= 1
    self * (self - 1).factorial
  end
end

5.factorial  # => 120
```

Risks:
- Name collisions with existing or future methods.
- Changes are global and affect all code in the same process.
- Debugging is harder because behavior is defined elsewhere.
- Library conflicts when two gems patch the same method differently.

---

**Q8. What are refinements and how do they solve the monkey patching problem?**

Refinements scope open class modifications to specific files or modules, preventing global pollution.

```ruby
module StringExtensions
  refine String do
    def word_count
      split.length
    end
  end
end

# Without using — not available
"hello world".respond_to?(:word_count)  # => false

module MyApp
  using StringExtensions

  def self.run
    "hello world".word_count  # => 2 (only available here)
  end
end

MyApp.run  # => 2
"hello world".word_count  # => NoMethodError (outside using scope)
```

Refinements are lexically scoped — activated only where `using` appears.

---

**Q9. Explain the module hooks: `included`, `extended`, `prepended`, and `inherited`.**

```ruby
module Trackable
  def self.included(base)
    puts "#{self} included into #{base}"
    base.extend(ClassMethods)
  end

  def self.extended(base)
    puts "#{self} extended onto #{base}"
  end

  def self.prepended(base)
    puts "#{self} prepended to #{base}"
  end

  module ClassMethods
    def tracked_by
      "tracked"
    end
  end
end

class User
  include Trackable  # triggers included(User)
end

User.tracked_by  # => "tracked" (ClassMethods added via included hook)
```

`inherited` fires when a class is subclassed:

```ruby
class Plugin
  def self.inherited(subclass)
    @registry ||= []
    @registry << subclass
    puts "Registered plugin: #{subclass}"
  end

  def self.registry
    @registry || []
  end
end

class AudioPlugin < Plugin; end
class VideoPlugin < Plugin; end

Plugin.registry  # => [AudioPlugin, VideoPlugin]
```

---

**Q10. What is `method_added` and what infinite loop trap does it create?**

`method_added` is called whenever a method is defined in a class/module. It receives the method name as a symbol.

```ruby
class Logger
  def self.method_added(method_name)
    # DANGER: defining a method here triggers method_added again → infinite loop
    puts "Method added: #{method_name}"
  end

  def greet; end  # triggers method_added(:greet)
end
```

Safe pattern — use a flag:

```ruby
class InstrumentedClass
  @adding_wrapper = false

  def self.method_added(method_name)
    return if @adding_wrapper
    return if method_name == :initialize

    @adding_wrapper = true
    original = instance_method(method_name)
    define_method(method_name) do |*args, &block|
      puts "Calling #{method_name}"
      original.bind(self).call(*args, &block)
    end
    @adding_wrapper = false
  end

  def hello
    "world"
  end
end

InstrumentedClass.new.hello
# Calling hello
# => "world"
```

---

**Q11. How does `Object#tap` work and when is it useful in metaprogramming contexts?**

`tap` yields the receiver to a block and returns the receiver unchanged. It's useful for debugging chains and for configuring objects inline.

```ruby
result = [1, 2, 3]
  .tap { |a| puts "Before: #{a}" }
  .map { |x| x * 2 }
  .tap { |a| puts "After map: #{a}" }
  .select { |x| x > 2 }

# Before: [1, 2, 3]
# After map: [2, 4, 6]
# => [4, 6]

# DSL-style object building
User.new.tap do |u|
  u.name = "Alice"
  u.email = "alice@example.com"
end
```

---

**Q12. Explain `Object#then` (alias `yield_self`) and how it differs from `tap`.**

`then` yields the receiver to a block and returns the **block's return value** (transformation). `tap` returns the original object.

```ruby
# then — transforms
"hello"
  .then { |s| s.upcase }
  .then { |s| "#{s}!!!" }
# => "HELLO!!!"

# Useful for conditional pipelines
user_id = 42
User.find(user_id)
    .then { |u| u.admin? ? AdminDecorator.new(u) : u }
    .then { |u| render_user(u) }
```

`tap` = inspect/side-effect, keep going. `then` = transform the value.

---

**Q13. How do you dynamically generate attribute accessors using `define_method`?**

```ruby
module Attributable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def typed_attr(name, type)
      var = :"@#{name}"

      define_method(name) do
        instance_variable_get(var)
      end

      define_method(:"#{name}=") do |value|
        raise TypeError, "Expected #{type}, got #{value.class}" unless value.is_a?(type)
        instance_variable_set(var, value)
      end
    end
  end
end

class Product
  include Attributable

  typed_attr :name,  String
  typed_attr :price, Numeric
end

p = Product.new
p.name = "Widget"   # ok
p.price = 9.99      # ok
p.price = "nine"    # => TypeError: Expected Numeric, got String
```

---

**Q14. What is `instance_variable_get`, `instance_variable_set`, and `remove_instance_variable`?**

```ruby
class Config
  def initialize
    @host = "localhost"
    @port = 3000
  end
end

c = Config.new
c.instance_variable_get(:@host)          # => "localhost"
c.instance_variable_set(:@debug, true)   # sets @debug
c.instance_variables                      # => [:@host, :@port, :@debug]
c.remove_instance_variable(:@debug)       # => true, removes it
c.instance_variables                      # => [:@host, :@port]
```

Useful in serialization, testing, and frameworks that need to inspect or manipulate object state.

---

**Q15. Explain `const_get`, `const_set`, and `const_missing`.**

```ruby
# const_get — look up a constant by name (string or symbol)
Kernel.const_get("Array")          # => Array
Kernel.const_get("Math::PI")       # => 3.141592653589793

# With namespace traversal disabled:
Object.const_get("String", false)  # only searches Object, no ancestor chain

# const_set — define a new constant
module Registry
  def self.register(name, klass)
    const_set(name, klass)
  end
end

Registry.register("FooBar", Class.new { def hello = "hi" })
Registry::FooBar.new.hello  # => "hi"

# const_missing — intercept missing constants (like method_missing for constants)
module AutoRequire
  def self.const_missing(name)
    require name.to_s.downcase
    const_get(name)
  rescue LoadError
    super
  end
end
```

---

**Q16. What is `ObjectSpace` and what can you do with it?**

`ObjectSpace` provides access to all live Ruby objects. It is primarily a debugging and framework tool.

```ruby
require 'objspace'

# Count all objects by class
ObjectSpace.count_objects  # => { T_OBJECT: 12, T_STRING: 3456, ... }

# Iterate all instances of a class
ObjectSpace.each_object(String) { |s| puts s if s.frozen? }

# Find all instances of a specific class
instances = ObjectSpace.each_object(MyClass).to_a

# Memory size of an object
ObjectSpace.memsize_of("hello")  # => 40 (bytes, platform-dependent)

# Trace new object allocations (Ruby 2.1+)
ObjectSpace.trace_object_allocations_start
obj = Object.new
puts ObjectSpace.allocation_sourcefile(obj)  # => "example.rb"
puts ObjectSpace.allocation_sourceline(obj)  # => line number
ObjectSpace.trace_object_allocations_stop
```

---

**Q17. What is `Kernel#freeze` and what does it mean for an object to be frozen?**

A frozen object cannot be mutated. Attempting to modify it raises `FrozenError` (formerly `RuntimeError`).

```ruby
str = "hello".freeze
str << " world"  # => FrozenError: can't modify frozen String

# freeze is shallow — nested objects are not frozen
arr = [[1, 2], [3, 4]].freeze
arr << [5]       # => FrozenError
arr[0] << 99     # works! arr[0] is not frozen

# Deep freeze pattern
def deep_freeze(obj)
  case obj
  when Array then obj.each { |e| deep_freeze(e) }.freeze
  when Hash  then obj.each { |k, v| deep_freeze(k); deep_freeze(v) }.freeze
  else obj.freeze
  end
end
```

---

**Q18. What is `BasicObject` and when would you subclass it instead of `Object`?**

`BasicObject` is the minimal root class with almost no methods. `Object` inherits from it and adds `Kernel`.

```ruby
BasicObject.instance_methods.length  # => ~8
Object.instance_methods.length       # => ~60+

# Subclass BasicObject for a clean-slate proxy
class CleanProxy < BasicObject
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    @target.__send__(name, *args, &block)
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end
end
```

Use `BasicObject` when you want to avoid interference from `Object`'s methods (e.g., `class`, `is_a?`, `nil?`) — typical for DSL builders and proxy objects.

---

**Q19. Explain the Ruby method lookup chain and where `method_missing` fits.**

```ruby
# Lookup order for obj.some_method:
# 1. obj's singleton class (eigenclass)
# 2. Prepended modules (in reverse include order)
# 3. obj.class
# 4. Included modules (in reverse include order)
# 5. Superclass (repeat 2-4 up the chain)
# 6. BasicObject
# If not found → method_missing is called following the SAME lookup chain
# If method_missing is not overridden → BasicObject#method_missing raises NoMethodError

module M1; def hello = "M1"; end
module M2; def hello = "M2"; end

class Base
  include M1

  def hello
    "Base"
  end
end

class Child < Base
  prepend M2
end

Child.new.hello  # => "M2" (prepended module wins)
```

---

**Q20. What is `Comparable` and how can `method_missing` power a comparison DSL?**

```ruby
class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  def <=>(other)
    degrees <=> other.degrees
  end

  # method_missing for comparison DSL: hot?, freezing?, boiling?
  def method_missing(name, *args)
    if name.to_s.end_with?('?')
      threshold_name = name.to_s.chomp('?')
      thresholds = { freezing: 0, cold: 10, warm: 20, hot: 30, boiling: 100 }
      if thresholds.key?(threshold_name.to_sym)
        degrees >= thresholds[threshold_name.to_sym]
      else
        super
      end
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    thresholds = %w[freezing cold warm hot boiling]
    (name.to_s.end_with?('?') && thresholds.include?(name.to_s.chomp('?'))) || super
  end
end

t = Temperature.new(35)
t.hot?      # => true
t.boiling?  # => false
t > Temperature.new(20)   # => true (from Comparable)
```

---

**Q21. How does `extend` differ from `include` in the context of metaprogramming?**

```ruby
module Greetable
  def greet
    "Hello, I am #{name}"
  end
end

# include — adds as instance methods
class Person
  include Greetable
  attr_reader :name
  def initialize(n) = @name = n
end
Person.new("Alice").greet  # works on instances

# extend — adds as singleton/class methods (or object-level)
module ClassStats
  def count
    @count ||= 0
  end
  def increment
    @count = count + 1
  end
end

class Order
  extend ClassStats
end
Order.increment
Order.count  # => 1

# extend on a single object
obj = Object.new
obj.extend(Greetable)
# Now only obj has greet; other Object instances don't
```

---

**Q22. What is `prepend` and how does it differ from `include` for method overriding?**

`prepend` inserts the module **before** the class in the method lookup chain, so the module's method runs first and can call `super` to invoke the original.

```ruby
module Timestamped
  def save
    @updated_at = Time.now
    super  # calls the original class#save
  end
end

class Record
  prepend Timestamped

  def save
    puts "Saving record..."
    true
  end
end

r = Record.new
r.save
# Saving record...
# => true, and @updated_at is set

# ancestors show prepended module BEFORE the class
Record.ancestors
# => [Timestamped, Record, Object, Kernel, BasicObject]
```

---

**Q23. How can you use `class_eval` with a string to define methods dynamically?**

```ruby
class ActiveModel
  ATTRIBUTES = [:name, :email, :age]

  ATTRIBUTES.each do |attr|
    class_eval <<~RUBY, __FILE__, __LINE__ + 1
      def #{attr}
        @#{attr}
      end

      def #{attr}=(value)
        @#{attr} = value
      end

      def #{attr}?
        !@#{attr}.nil? && @#{attr} != ""
      end
    RUBY
  end
end

m = ActiveModel.new
m.name = "Alice"
m.name?   # => true
m.age?    # => false
```

Passing `__FILE__` and `__LINE__` preserves accurate backtraces.

---

**Q24. Explain the eigenclass (singleton class) and how to access it.**

Every Ruby object has a hidden singleton class that holds methods defined only for that specific object.

```ruby
obj = Object.new

# Access singleton class
singleton = class << obj
  self
end
# or:
singleton = obj.singleton_class  # Ruby 1.9.2+

# Define singleton method
class << obj
  def special
    "I belong to only this object"
  end
end

obj.special   # => "I belong to only this object"
Object.new.special  # => NoMethodError

# Class methods ARE singleton methods of the class object
class Dog
  class << self
    def species
      "Canis lupus familiaris"
    end
  end
end
Dog.species  # => "Canis lupus familiaris"
```

---

**Q25. What is `Kernel#binding` and how does it relate to metaprogramming with `eval`?**

```ruby
# binding captures the current execution context
def make_binding(x)
  binding
end

b = make_binding(42)
eval("x", b)  # => 42  (x from the binding context)

# ERB uses binding to render templates
require 'erb'

class TemplateRenderer
  def initialize(name, value)
    @name = name
    @value = value
  end

  def render(template)
    ERB.new(template).result(binding)
  end
end

renderer = TemplateRenderer.new("score", 95)
renderer.render("The <%= @name %> is <%= @value %>")
# => "The score is 95"
```

`binding` is how Ruby's template engines, debugging tools, and interactive consoles (IRB, Pry) access local state.

---

**Q26. How does `Struct` use metaprogramming internally, and how can you replicate it?**

```ruby
# Struct.new generates a class dynamically
Point = Struct.new(:x, :y) do
  def distance_to(other)
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end
end

Point.new(0, 0).distance_to(Point.new(3, 4))  # => 5.0

# Replicating Struct with metaprogramming
def make_value_object(*attrs)
  Class.new do
    attrs.each do |attr|
      define_method(attr) { instance_variable_get(:"@#{attr}") }
      define_method(:"#{attr}=") { |v| instance_variable_set(:"@#{attr}", v) }
    end

    define_method(:initialize) do |*values|
      attrs.zip(values).each do |attr, val|
        instance_variable_set(:"@#{attr}", val)
      end
    end

    define_method(:to_h) do
      attrs.each_with_object({}) { |a, h| h[a] = send(a) }
    end
  end
end

Coordinate = make_value_object(:lat, :lon)
c = Coordinate.new(51.5, -0.1)
c.to_h  # => { lat: 51.5, lon: -0.1 }
```

---

**Q27. What are the security risks of using `eval` and `send` with user input?**

```ruby
# DANGEROUS: eval with user input
user_input = "system('rm -rf /')"
eval(user_input)  # arbitrary code execution

# DANGEROUS: send with user input
user_input = "destroy_everything"
object.send(user_input)  # can call private/destructive methods

# SAFER: public_send with whitelisting
ALLOWED_METHODS = %i[name email created_at].freeze

def safe_attribute(record, method_name)
  sym = method_name.to_sym
  raise ArgumentError, "Not allowed" unless ALLOWED_METHODS.include?(sym)
  record.public_send(sym)
end

# SAFEST: explicit mapping
ATTRIBUTE_MAP = {
  "name"  => ->(u) { u.name },
  "email" => ->(u) { u.email }
}.freeze

ATTRIBUTE_MAP.fetch(user_input, -> (_) { raise "Unknown" }).call(user)
```
