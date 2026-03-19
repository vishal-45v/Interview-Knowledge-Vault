# Chapter 06: Metaprogramming — Structured Answers

## Answer 1: Implement a Complete ActiveRecord-Like Model with Metaprogramming

**Question:** "Show me how you'd build a lightweight ActiveRecord-like model with dynamic attribute generation, type casting, validation, and dirty tracking."

**Answer:**

```ruby
module Persistence
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@attribute_definitions, {})
  end

  module ClassMethods
    def attribute(name, type: :string, default: nil, validates: nil)
      @attribute_definitions[name] = { type: type, default: default, validates: validates }
      ivar     = :"@#{name}"
      prev_var = :"@#{name}_previous"

      # Reader
      define_method(name) do
        if instance_variable_defined?(ivar)
          instance_variable_get(ivar)
        else
          self.class.attribute_definitions[name][:default]
        end
      end

      # Writer with type casting + dirty tracking
      define_method(:"#{name}=") do |raw_value|
        casted = cast_value(raw_value, type)
        old = instance_variable_get(ivar) if instance_variable_defined?(ivar)
        instance_variable_set(prev_var, old)
        instance_variable_set(ivar, casted)
      end

      # Predicate
      define_method(:"#{name}?") do
        val = send(name)
        !val.nil? && val != "" && val != false
      end

      # Dirty tracking
      define_method(:"#{name}_changed?") do
        instance_variable_defined?(prev_var) &&
          instance_variable_get(prev_var) != send(name)
      end

      define_method(:"#{name}_was") do
        instance_variable_get(prev_var)
      end
    end

    def attribute_definitions
      @attribute_definitions
    end

    def validate(attr, with:)
      @attribute_definitions[attr][:validates] = with
    end
  end

  def cast_value(value, type)
    return nil if value.nil?
    case type
    when :string  then value.to_s
    when :integer then Integer(value)
    when :float   then Float(value)
    when :boolean then [true, "true", 1, "1", "yes"].include?(value)
    when :symbol  then value.to_sym
    else value
    end
  end

  def valid?
    @errors = {}
    self.class.attribute_definitions.each do |attr, defn|
      validator = defn[:validates]
      next unless validator
      value = send(attr)
      result = validator.call(value)
      @errors[attr] = result unless result == true
    end
    @errors.empty?
  end

  def errors
    @errors ||= {}
  end

  def attributes
    self.class.attribute_definitions.keys.each_with_object({}) do |attr, h|
      h[attr] = send(attr)
    end
  end

  def changed_attributes
    self.class.attribute_definitions.keys.select { |a| send(:"#{a}_changed?") }
  end
end

class User
  include Persistence

  attribute :name,  type: :string
  attribute :age,   type: :integer, default: 0
  attribute :admin, type: :boolean, default: false

  validate :name, with: ->(v) { v.to_s.length >= 2 ? true : "too short" }
  validate :age,  with: ->(v) { v.to_i >= 0 ? true : "must be positive" }
end

u = User.new
u.name = "Alice"
u.age  = "30"         # string "30" cast to integer 30
u.admin = true

u.age           # => 30
u.admin?        # => true
u.attributes    # => { name: "Alice", age: 30, admin: true }
u.valid?        # => true

u.name = "B"
u.name_changed? # => true
u.name_was      # => "Alice"
u.valid?        # => false
u.errors        # => { name: "too short" }
```

---

## Answer 2: The Ghost Method Pattern vs. Dynamic Method Definition

**Question:** "What's the difference between using `method_missing` (ghost methods) and defining methods dynamically with `define_method`?"

**Answer:**

Ghost methods (via `method_missing`) are methods that appear to exist but are handled dynamically at runtime. They are never actually defined. Dynamic methods (via `define_method`) are real Ruby methods that exist in the method table.

```ruby
# Ghost method approach — method_missing
class GhostFinder
  def method_missing(name, *args)
    if name.to_s =~ /\Afind_by_(\w+)\z/
      attr = $1
      puts "Ghost: finding where #{attr} = #{args.first}"
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    name.to_s.start_with?('find_by_') || super
  end
end

# Dynamic method approach — define_method at class load time
class RealFinder
  COLUMNS = [:name, :email, :age].freeze

  COLUMNS.each do |col|
    define_method(:"find_by_#{col}") do |value|
      puts "Real: finding where #{col} = #{value}"
    end
  end
end

gf = GhostFinder.new
rf = RealFinder.new

# Behavior difference:
gf.find_by_anything("foo")   # works — any attribute
rf.find_by_name("Alice")     # works — only defined columns

# Performance: define_method wins after first call
# Introspection: define_method wins
rf.respond_to?(:find_by_name)  # => true
rf.method(:find_by_name)       # => #<Method: ...>

# method_missing is better when:
# - You don't know all possible method names at class load time
# - The set of methods is effectively infinite
# - You're building a proxy/decorator that wraps an unknown interface

# define_method is better when:
# - The set of methods is known and finite
# - You need performance (no method_missing overhead after first call)
# - You need proper introspection (method, respond_to?, documentation)
```

**Key trade-offs:**

| Aspect | `method_missing` | `define_method` |
|--------|-----------------|-----------------|
| Speed | Slower (full lookup per call) | Normal method speed |
| Introspection | Requires `respond_to_missing?` hack | Full native support |
| Stack traces | Shows `method_missing` in trace | Shows actual method name |
| Use case | Unknown/infinite method space | Known, finite method set |

---

## Answer 3: How `include`, `extend`, and `prepend` Affect the Method Lookup Chain

**Question:** "Explain exactly where in the ancestor chain modules appear when you include, extend, or prepend them."

**Answer:**

```ruby
module M1
  def describe = "M1"
end

module M2
  def describe = "M2"
end

# 1. include — module inserted AFTER the class in ancestors
class WithInclude
  include M1
  include M2  # last included appears first (just after class)

  def describe = "WithInclude"
end

WithInclude.ancestors
# => [WithInclude, M2, M1, Object, Kernel, BasicObject]
WithInclude.new.describe  # => "WithInclude" (class wins)

# 2. prepend — module inserted BEFORE the class in ancestors
class WithPrepend
  prepend M1
  prepend M2  # last prepended appears first (before class)

  def describe = "WithPrepend"
end

WithPrepend.ancestors
# => [M2, M1, WithPrepend, Object, Kernel, BasicObject]
WithPrepend.new.describe  # => "M2" (first prepended wins)

# 3. extend — adds to the SINGLETON CLASS's include chain
class WithExtend
  extend M1
end

# Adds to singleton class ancestors:
WithExtend.singleton_class.ancestors
# => [#<Class:WithExtend>, M1, #<Class:Object>, ...]
WithExtend.describe  # => "M1" (class method now)

# super chaining with prepend:
module Wrapper
  def describe
    "Wrapper(#{super})"  # super calls the prepended-into class
  end
end

class Base
  prepend Wrapper
  def describe = "Base"
end

Base.new.describe  # => "Wrapper(Base)"
```

---

## Answer 4: Building a DSL with `instance_eval` and `class_eval`

**Question:** "How would you implement a test framework's DSL (like RSpec's `describe`/`it`) using Ruby metaprogramming?"

**Answer:**

```ruby
class SpecRunner
  @suites = []

  def self.describe(description, &block)
    suite = Suite.new(description)
    suite.instance_eval(&block)
    @suites << suite
    suite
  end

  def self.run_all
    @suites.each(&:run)
  end

  class Suite
    def initialize(description)
      @description = description
      @examples    = []
      @befores     = []
      @afters      = []
      @lets        = {}
    end

    def it(description, &block)
      @examples << { description: description, block: block }
    end

    def before(&block)
      @befores << block
    end

    def after(&block)
      @afters << block
    end

    def let(name, &block)
      @lets[name] = block
    end

    def run
      puts "\n#{@description}"
      @examples.each do |example|
        context = ExampleContext.new(@lets)
        @befores.each { |b| context.instance_eval(&b) }
        begin
          context.instance_eval(&example[:block])
          puts "  ✓ #{example[:description]}"
        rescue => e
          puts "  ✗ #{example[:description]}: #{e.message}"
        ensure
          @afters.each { |a| context.instance_eval(&a) }
        end
      end
    end
  end

  class ExampleContext
    def initialize(lets)
      @let_cache = {}
      lets.each do |name, block|
        define_singleton_method(name) do
          @let_cache[name] ||= instance_eval(&block)
        end
      end
    end

    def expect(actual)
      Expectation.new(actual)
    end
  end

  class Expectation
    def initialize(actual)
      @actual = actual
    end

    def to(matcher)
      result, message = matcher.call(@actual)
      raise "Expected #{@actual.inspect} #{message}" unless result
    end
  end
end

def eq(expected)
  ->(actual) { [actual == expected, "to eq #{expected.inspect}, got #{actual.inspect}"] }
end

# Usage:
SpecRunner.describe "Array" do
  let(:arr) { [1, 2, 3] }

  before { @count = 0 }

  it "has correct length" do
    expect(arr.length).to eq(3)
  end

  it "supports map" do
    expect(arr.map { |x| x * 2 }).to eq([2, 4, 6])
  end
end

SpecRunner.run_all
# Array
#   ✓ has correct length
#   ✓ supports map
```

---

## Answer 5: `Refinements` — Scope-Safe Monkey Patching

**Question:** "What problem do refinements solve and what are their limitations?"

**Answer:**

```ruby
# The problem: monkey patching is global and permanent
class Integer
  def hours
    self * 3600  # affects ALL integers in ALL code
  end
end

# Refinements: scoped, reversible augmentation
module TimeHelpers
  refine Integer do
    def hours
      self * 3600
    end

    def minutes
      self * 60
    end
  end

  refine Symbol do
    def from_now
      Time.now + send(self)  # only works within using scope
    end
  end
end

# Must explicitly activate with `using`
module Scheduling
  using TimeHelpers

  def self.schedule_in(amount, unit)
    Time.now + amount.send(unit)
  end

  2.hours    # => 7200 — works here
  30.minutes # => 1800 — works here
end

2.hours  # => NoMethodError — not available outside

# Limitations of refinements:
# 1. Cannot be used in class/method bodies (only at file/module level)
# 2. Dynamic dispatch doesn't use refinements:
module Demonstrating
  using TimeHelpers

  def self.call_dynamically
    method_name = :hours
    2.send(method_name)  # => NoMethodError! send bypasses refinements
    2.hours              # => 7200  (direct call works)
  end
end

# 3. Refinements don't apply inside eval strings
# 4. Cannot refine BasicObject
# 5. Once a refinement is active in a scope, you can't deactivate it

# Good use cases:
# - Adding methods to built-in types for a specific domain
# - Test helpers that extend objects only in test files
# - Library-internal extensions that shouldn't leak to users
```

---

## Answer 6: `ObjectSpace` for Memory Debugging and Instance Tracking

**Question:** "How would you use `ObjectSpace` to debug a memory leak where objects aren't being garbage collected?"

**Answer:**

```ruby
require 'objspace'

# Step 1: Identify which class is accumulating instances
def count_instances_by_class(top_n = 10)
  counts = Hash.new(0)
  ObjectSpace.each_object { |o| counts[o.class] += 1 }
  counts.sort_by { |_, v| -v }.first(top_n).to_h
end

before = count_instances_by_class
# ... run code that might leak ...
after = count_instances_by_class

growth = after.merge(before) { |_k, a, b| a - b }
         .select { |_, v| v > 0 }
         .sort_by { |_, v| -v }

puts "Growing classes: #{growth}"

# Step 2: Find WHERE leaking objects are allocated
ObjectSpace.trace_object_allocations_start

# ... run the leaking code ...

ObjectSpace.each_object(LeakingClass) do |obj|
  file = ObjectSpace.allocation_sourcefile(obj)
  line = ObjectSpace.allocation_sourceline(obj)
  puts "Allocated at #{file}:#{line}"
end

ObjectSpace.trace_object_allocations_stop

# Step 3: Check object retention with GC::Profiler
GC::Profiler.enable
10.times { GC.start }
puts GC::Profiler.report
GC::Profiler.disable

# Step 4: Find objects retained after GC
class LeakDetector
  def self.find_uncollected_instances(klass)
    3.times { GC.compact rescue GC.start }
    ObjectSpace.each_object(klass).to_a
  end
end

# Common leak patterns:
# - Class-level arrays/hashes that grow unbounded
# - Callbacks holding references to instances
# - Thread-local variables not cleared
# - Singleton registries that never evict

class CacheThatLeaks
  @cache = {}  # grows forever — use a size-bounded cache instead

  def self.store(key, value)
    @cache[key] = value
  end
end

# Better: use a bounded cache
require 'lru_redux'  # or implement manually
class BoundedCache
  MAX_SIZE = 1000

  def initialize
    @data = {}
    @keys = []
  end

  def store(key, value)
    evict_oldest if @keys.size >= MAX_SIZE
    @data[key] = value
    @keys << key
  end

  private

  def evict_oldest
    old_key = @keys.shift
    @data.delete(old_key)
  end
end
```

---

## Answer 7: Implementing a Complete Proxy with `BasicObject`

**Question:** "Why use `BasicObject` for proxies and how do you implement one correctly?"

**Answer:**

```ruby
# Problem with Object-based proxy: Object has ~60 methods that "pass through"
# accidentally to the proxy object instead of being delegated
class ObjectProxy
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    @target.send(name, *args, &block)
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end
end

arr = ObjectProxy.new([1, 2, 3])
arr.class    # => ObjectProxy (wrong! should delegate to Array and return Array)
arr.is_a?(Array)  # => false (wrong!)
arr.nil?     # => false (correct, but by accident — Object#nil? ran on proxy)

# Solution: BasicObject proxy
class CleanProxy < BasicObject
  def initialize(target)
    @target = target
  end

  # Must use __send__ because send doesn't exist on BasicObject
  def method_missing(name, *args, &block)
    @target.__send__(name, *args, &block)
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end

  # BasicObject doesn't have respond_to? itself, but Ruby calls respond_to_missing?
  # We must manually define respond_to? since BasicObject doesn't have it
  def respond_to?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end

  # class is important for logging/debugging
  def class
    @target.class
  end

  def inspect
    @target.inspect
  end

  def is_a?(klass)
    @target.is_a?(klass)
  end
end

arr = CleanProxy.new([1, 2, 3])
arr.class         # => Array (correctly delegated)
arr.is_a?(Array)  # => true
arr.length        # => 3
arr.map { |x| x * 2 }  # => [2, 4, 6]

# Practical use: transparent caching proxy
class CachingProxy < BasicObject
  def initialize(target, cache = {})
    @target = target
    @cache  = cache
  end

  def method_missing(name, *args, &block)
    cache_key = [name, args]
    unless @cache.key?(cache_key)
      @cache[cache_key] = @target.__send__(name, *args, &block)
    end
    @cache[cache_key]
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end
end
```

---

## Answer 8: The `included` Hook Pattern for Extending Class AND Instance Behavior

**Question:** "Show me the canonical pattern for a module that adds both class methods and instance methods when included."

**Answer:**

```ruby
module Validatable
  # Called when module is included in a class
  def self.included(base)
    base.extend(ClassMethods)          # add class-level methods
    base.include(InstanceMethods)      # add instance-level methods (optional explicit)
    base.instance_variable_set(:@validations, [])

    # Hook into the class to run validations on initialize
    base.class_eval do
      alias_method :__original_initialize, :initialize rescue nil

      def initialize(attrs = {})
        attrs.each { |k, v| send(:"#{k}=", v) if respond_to?(:"#{k}=") }
        __run_validations!
      end
    end
  end

  module ClassMethods
    def validates(attr, **options)
      @validations ||= []
      @validations << { attr: attr, options: options }
    end

    def validations
      @validations || []
    end
  end

  module InstanceMethods
    def valid?
      __run_validations!
      @errors.empty?
    end

    def errors
      @errors ||= {}
    end

    private

    def __run_validations!
      @errors = {}
      self.class.validations.each do |v|
        attr  = v[:attr]
        value = send(attr) rescue nil
        opts  = v[:options]

        if opts[:presence] && (value.nil? || value.to_s.strip.empty?)
          @errors[attr] = "can't be blank"
        end

        if opts[:length] && value.respond_to?(:length)
          range = opts[:length]
          @errors[attr] = "is too short" if range.is_a?(Range) && value.length < range.min
          @errors[attr] = "is too long"  if range.is_a?(Range) && value.length > range.max
        end

        if opts[:format] && !opts[:format].match?(value.to_s)
          @errors[attr] = "is invalid"
        end
      end
    end
  end
end

class Contact
  include Validatable

  attr_accessor :name, :email, :phone

  validates :name,  presence: true, length: 2..50
  validates :email, presence: true, format: /\A[^@]+@[^@]+\.[^@]+\z/
  validates :phone, format: /\A\+?\d{7,15}\z/
end

c = Contact.new(name: "Alice", email: "alice@example.com", phone: "+15551234567")
c.valid?   # => true
c.errors   # => {}

c2 = Contact.new(name: "A", email: "not-an-email", phone: "abc")
c2.valid?  # => false
c2.errors  # => { name: "is too short", email: "is invalid", phone: "is invalid" }
```

---

## Answer 9: Using `const_missing` for Auto-Loading Constants

**Question:** "How does Rails autoloading work conceptually? Implement a simplified version using `const_missing`."

**Answer:**

```ruby
module AutoLoader
  LOAD_PATHS = [
    File.join(__dir__, 'models'),
    File.join(__dir__, 'services'),
    File.join(__dir__, 'lib'),
  ].freeze

  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def const_missing(name)
      file_name = name.to_s
                      .gsub(/([A-Z]+)([A-Z][a-z])/, '\1_\2')
                      .gsub(/([a-z\d])([A-Z])/, '\1_\2')
                      .downcase

      AutoLoader::LOAD_PATHS.each do |path|
        full_path = File.join(path, "#{file_name}.rb")
        if File.exist?(full_path)
          require full_path
          return const_get(name) if const_defined?(name)
        end
      end

      super  # raises NameError if not found
    end
  end
end

# Simulated files that would be loaded:
# models/user.rb → defines class User
# services/payment_processor.rb → defines class PaymentProcessor

# Usage (in a module that includes AutoLoader):
module App
  include AutoLoader

  def self.run
    user = User.new          # triggers const_missing(:User)
    processor = PaymentProcessor.new  # triggers const_missing(:PaymentProcessor)
  end
end

# Zeitwerk (modern Rails autoloader) uses a more sophisticated version:
# - It maps file paths to constant names upfront (eager loading index)
# - Uses Module#autoload (built-in Ruby mechanism) instead of const_missing
# - Supports reloading (for development mode)

# Module#autoload — Ruby's built-in lazy constant loading:
autoload :MyClass, 'path/to/my_class'
# MyClass is not loaded until it's first referenced
MyClass.new  # triggers require 'path/to/my_class' at this point
```

---

## Answer 10: Wrapping Methods with `prepend` for Transparent Instrumentation

**Question:** "How would you instrument all methods in a class to record call counts and timing without modifying the original class?"

**Answer:**

```ruby
module Instrumentation
  def self.instrument(klass)
    instrument_module = Module.new do
      klass.instance_methods(false).each do |method_name|
        define_method(method_name) do |*args, &block|
          tracker = Instrumentation::Tracker.instance
          tracker.record_start(klass, method_name)
          start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)

          begin
            result = super(*args, &block)
            tracker.record_success(klass, method_name, Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time)
            result
          rescue => e
            tracker.record_error(klass, method_name, e)
            raise
          end
        end
      end
    end

    klass.prepend(instrument_module)
  end

  class Tracker
    require 'singleton'
    include Singleton

    def initialize
      @stats = Hash.new { |h, k| h[k] = { calls: 0, errors: 0, total_time: 0.0 } }
    end

    def record_start(klass, method)
      @stats["#{klass}##{method}"][:calls] += 1
    end

    def record_success(klass, method, elapsed)
      @stats["#{klass}##{method}"][:total_time] += elapsed
    end

    def record_error(klass, method, error)
      @stats["#{klass}##{method}"][:errors] += 1
    end

    def report
      @stats.sort_by { |_, v| -v[:total_time] }.each do |key, stats|
        avg = stats[:calls] > 0 ? (stats[:total_time] / stats[:calls] * 1000).round(2) : 0
        puts "#{key}: #{stats[:calls]} calls, #{avg}ms avg, #{stats[:errors]} errors"
      end
    end
  end
end

class DataProcessor
  def process(data)
    data.map { |x| x * 2 }
  end

  def validate(data)
    data.all? { |x| x.is_a?(Numeric) }
  end
end

Instrumentation.instrument(DataProcessor)

processor = DataProcessor.new
processor.process([1, 2, 3])
processor.validate([1, "a", 3])
processor.process([4, 5])

Instrumentation::Tracker.instance.report
# DataProcessor#process: 2 calls, 0.01ms avg, 0 errors
# DataProcessor#validate: 1 calls, 0.02ms avg, 0 errors
```
