# Chapter 06: Metaprogramming — Scenario Questions

## DSL & Framework Design

**S1. You're building a configuration DSL for a web framework. Users should be able to write:**
```ruby
AppConfig.configure do
  database host: "localhost", port: 5432
  cache    backend: :redis, ttl: 300
  feature  :dark_mode, enabled: true
end
```
**How would you implement this using metaprogramming?**

```ruby
class AppConfig
  class ConfigSection
    attr_reader :settings

    def initialize
      @settings = {}
    end

    def method_missing(name, *args)
      @settings[name] = args.first
    end

    def respond_to_missing?(name, include_private = false)
      true
    end
  end

  @sections = {}

  def self.configure(&block)
    instance_eval(&block)
  end

  def self.method_missing(name, *args, &block)
    section = ConfigSection.new
    if block_given?
      section.instance_eval(&block)
    else
      section.settings.merge!(args.first || {})
    end
    @sections[name] = section
  end

  def self.respond_to_missing?(name, include_private = false)
    true
  end

  def self.[](section)
    @sections[section]
  end
end

AppConfig.configure do
  database host: "localhost", port: 5432
end

AppConfig[:database].settings  # => { host: "localhost", port: 5432 }
```

---

**S2. Implement ActiveRecord-style dynamic finders: `find_by_name`, `find_by_email`, `find_by_name_and_email`.**

```ruby
class Model
  def self.columns
    [:id, :name, :email, :age, :created_at]
  end

  def self.where(conditions)
    # Simulated: would query database
    puts "SELECT * WHERE #{conditions.map { |k,v| "#{k}='#{v}'" }.join(' AND ')}"
    []
  end

  def self.method_missing(name, *args)
    method_str = name.to_s

    if method_str.start_with?('find_by_')
      attr_string = method_str.sub('find_by_', '')
      attr_names = attr_string.split('_and_').map(&:to_sym)

      # Validate all attributes exist
      invalid = attr_names - columns
      return super unless invalid.empty?

      # Define the method for future calls (optimization)
      define_singleton_method(name) do |*vals|
        conditions = attr_names.zip(vals).to_h
        where(conditions).first
      end

      conditions = attr_names.zip(args).to_h
      where(conditions).first
    else
      super
    end
  end

  def self.respond_to_missing?(name, include_private = false)
    method_str = name.to_s
    if method_str.start_with?('find_by_')
      attrs = method_str.sub('find_by_', '').split('_and_').map(&:to_sym)
      (attrs - columns).empty?
    else
      super
    end
  end
end

Model.find_by_name("Alice")
# SELECT * WHERE name='Alice'

Model.find_by_name_and_email("Alice", "alice@example.com")
# SELECT * WHERE name='Alice' AND email='alice@example.com'

Model.respond_to?(:find_by_email)  # => true
Model.respond_to?(:find_by_ssn)    # => false (not in columns)
```

---

**S3. Build a simple ORM-like attribute system with dirty tracking (know which attributes changed).**

```ruby
module DirtyTracking
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@tracked_attrs, [])
  end

  module ClassMethods
    def track_attr(*names)
      @tracked_attrs.concat(names)
      names.each do |name|
        ivar     = :"@#{name}"
        original = :"@#{name}_was"

        define_method(name) do
          instance_variable_get(ivar)
        end

        define_method(:"#{name}=") do |value|
          current = instance_variable_get(ivar)
          instance_variable_set(original, current) unless instance_variable_defined?(original)
          instance_variable_set(ivar, value)
        end

        define_method(:"#{name}_changed?") do
          instance_variable_defined?(original) &&
            instance_variable_get(original) != instance_variable_get(ivar)
        end

        define_method(:"#{name}_was") do
          instance_variable_get(original)
        end
      end
    end

    def tracked_attrs
      @tracked_attrs
    end
  end

  def changed_attributes
    self.class.tracked_attrs.select { |attr| send(:"#{attr}_changed?") }
  end

  def changes
    changed_attributes.each_with_object({}) do |attr, h|
      h[attr] = [send(:"#{attr}_was"), send(attr)]
    end
  end

  def changed?
    changed_attributes.any?
  end
end

class User
  include DirtyTracking
  track_attr :name, :email
end

u = User.new
u.name  = "Alice"
u.email = "alice@example.com"
u.name  = "Bob"

u.name_changed?  # => true
u.name_was       # => "Alice"
u.changes        # => { name: ["Alice", "Bob"] }
u.changed?       # => true
```

---

**S4. Create a memoization decorator using `define_method` that can be applied to any method.**

```ruby
module Memoizable
  def memoize(*method_names)
    method_names.each do |method_name|
      original = instance_method(method_name)
      cache_var = :"@_memo_#{method_name}"

      define_method(method_name) do |*args|
        cache = instance_variable_get(cache_var) || instance_variable_set(cache_var, {})
        unless cache.key?(args)
          cache[args] = original.bind(self).call(*args)
        end
        cache[args]
      end
    end
  end
end

class Calculator
  extend Memoizable

  def fibonacci(n)
    return n if n <= 1
    fibonacci(n - 1) + fibonacci(n - 2)
  end

  memoize :fibonacci
end

calc = Calculator.new
calc.fibonacci(40)  # fast after first call
calc.fibonacci(40)  # retrieved from cache
```

---

**S5. Implement a policy/permission system where rules are declared using a DSL.**

```ruby
class Policy
  @rules = {}

  def self.can(action, &condition)
    @rules[action] = condition
  end

  def self.rules
    @rules
  end

  def initialize(user)
    @user = user
  end

  def method_missing(name, *args)
    action = name.to_s.sub(/\?$/, '').to_sym
    rule = self.class.rules[action]
    if rule
      rule.call(@user, *args)
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    action = name.to_s.sub(/\?$/, '').to_sym
    self.class.rules.key?(action) || super
  end
end

class PostPolicy < Policy
  can(:read)   { |user| true }
  can(:write)  { |user| user.role == :editor || user.role == :admin }
  can(:delete) { |user| user.role == :admin }
end

alice = Struct.new(:role).new(:editor)
policy = PostPolicy.new(alice)
policy.read?    # => true
policy.write?   # => true
policy.delete?  # => false
```

---

**S6. Build a state machine DSL using class-level metaprogramming.**

```ruby
module StateMachine
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@states, [])
    base.instance_variable_set(:@transitions, {})
  end

  module ClassMethods
    def state(*names)
      @states.concat(names)
      names.each do |name|
        define_method(:"#{name}?") { @state == name }
      end
    end

    def transition(from:, to:, on:)
      @transitions[on] ||= {}
      @transitions[on][from] = to

      define_method(on) do
        allowed = self.class.instance_variable_get(:@transitions)[on] || {}
        next_state = allowed[@state]
        raise "Cannot #{on} from #{@state}" unless next_state
        @state = next_state
        self
      end
    end

    def initial_state(s)
      define_method(:initialize) { @state = s }
    end

    def states
      @states
    end
  end

  def state
    @state
  end
end

class TrafficLight
  include StateMachine

  initial_state :red

  state :red, :yellow, :green

  transition from: :red,    to: :green,  on: :go
  transition from: :green,  to: :yellow, on: :slow
  transition from: :yellow, to: :red,    on: :stop
end

light = TrafficLight.new
light.state    # => :red
light.go
light.state    # => :green
light.slow
light.state    # => :yellow
light.stop
light.red?     # => true
```

---

**S7. You need to build a logging proxy that intercepts all method calls on any object and logs them with timing.**

```ruby
class LoggingProxy < BasicObject
  def initialize(target, logger = ::Kernel.method(:puts))
    @target = target
    @logger = logger
  end

  def method_missing(name, *args, &block)
    start = ::Time.now
    result = @target.__send__(name, *args, &block)
    elapsed = ((::Time.now - start) * 1000).round(2)
    @logger.call("[LOG] #{@target.class}##{name}(#{args.inspect}) => #{result.inspect} [#{elapsed}ms]")
    result
  rescue => e
    @logger.call("[ERROR] #{@target.class}##{name} raised #{e.class}: #{e.message}")
    raise
  end

  def respond_to_missing?(name, include_private = false)
    @target.respond_to?(name, include_private)
  end
end

proxy = LoggingProxy.new([1, 2, 3])
proxy.length   # [LOG] Array#length([]) => 3 [0.01ms]
proxy.map { |x| x * 2 }  # [LOG] Array#map([]) => [2, 4, 6] [0.02ms]
```

---

**S8. Implement an `attr_accessor` replacement that auto-validates type and range.**

```ruby
module ValidatedAttributes
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def validated_attr(name, type: nil, range: nil, format: nil, nullable: false)
      ivar = :"@#{name}"

      define_method(name) { instance_variable_get(ivar) }

      define_method(:"#{name}=") do |value|
        if value.nil?
          raise ArgumentError, "#{name} cannot be nil" unless nullable
        else
          if type && !value.is_a?(type)
            raise TypeError, "#{name} must be #{type}, got #{value.class}"
          end
          if range && !range.include?(value)
            raise ArgumentError, "#{name} must be in #{range}, got #{value}"
          end
          if format && !format.match?(value.to_s)
            raise ArgumentError, "#{name} does not match format #{format}"
          end
        end
        instance_variable_set(ivar, value)
      end
    end
  end
end

class Employee
  include ValidatedAttributes

  validated_attr :name,   type: String, format: /\A[A-Za-z ]+\z/
  validated_attr :age,    type: Integer, range: 18..65
  validated_attr :salary, type: Numeric, range: 0..Float::INFINITY
  validated_attr :email,  type: String, format: /\A[^@]+@[^@]+\z/, nullable: true
end

e = Employee.new
e.name   = "Alice Johnson"  # ok
e.age    = 30               # ok
e.age    = 200              # ArgumentError: age must be in 18..65
e.name   = "H4ck3r"         # ArgumentError: name does not match format
```

---

**S9. Implement a simple version of `Struct` from scratch using metaprogramming.**

```ruby
def MyStruct(*attrs)
  Class.new do
    attrs.each do |attr|
      attr_accessor attr
    end

    define_method(:initialize) do |*values|
      if values.length > attrs.length
        raise ArgumentError, "Too many arguments (#{values.length} for #{attrs.length})"
      end
      attrs.each_with_index do |attr, i|
        instance_variable_set(:"@#{attr}", values[i])
      end
    end

    define_method(:to_h) do
      attrs.each_with_object({}) { |a, h| h[a] = send(a) }
    end

    define_method(:==) do |other|
      other.class == self.class && to_h == other.to_h
    end

    define_method(:to_s) do
      "#<struct #{attrs.map { |a| "#{a}=#{send(a).inspect}" }.join(', ')}>"
    end

    define_method(:members) { attrs }

    define_method(:[]) do |key|
      send(key.is_a?(Integer) ? attrs[key] : key)
    end
  end
end

Point = MyStruct(:x, :y)
p = Point.new(1, 2)
p.x      # => 1
p.to_h   # => { x: 1, y: 2 }
p == Point.new(1, 2)  # => true
p[0]     # => 1
p[:y]    # => 2
```

---

**S10. Build a module that adds automatic `to_json` and `from_json` class methods using field declarations.**

```ruby
require 'json'

module Serializable
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@serializable_fields, [])
  end

  module ClassMethods
    def field(name, type: String)
      @serializable_fields << { name: name, type: type }
      attr_accessor name
    end

    def serializable_fields
      @serializable_fields
    end

    def from_json(json_string)
      data = JSON.parse(json_string)
      obj = new
      @serializable_fields.each do |f|
        value = data[f[:name].to_s]
        obj.send(:"#{f[:name]}=", cast(value, f[:type]))
      end
      obj
    end

    private

    def cast(value, type)
      return value if value.nil?
      case type
      when :integer then value.to_i
      when :float   then value.to_f
      when :boolean then value == true || value == "true"
      else value.to_s
      end
    end
  end

  def to_json(*_args)
    self.class.serializable_fields.each_with_object({}) do |f, h|
      h[f[:name].to_s] = send(f[:name])
    end.to_json
  end
end

class Person
  include Serializable
  field :name
  field :age,    type: :integer
  field :active, type: :boolean
end

p = Person.from_json('{"name":"Alice","age":"30","active":true}')
p.name    # => "Alice"
p.age     # => 30
p.to_json # => '{"name":"Alice","age":30,"active":true}'
```

---

**S11. You are asked to implement `method_missing`-based dynamic scopes like `by_status_active`, `by_role_admin`.**

```ruby
class QueryBuilder
  def initialize(model_class)
    @model_class = model_class
    @conditions = {}
  end

  def method_missing(name, *args)
    pattern = /\Aby_([a-z_]+?)_([a-z_]+)\z/
    match = name.to_s.match(pattern)
    if match
      column = match[1].to_sym
      value  = match[2]
      # Convert value type intelligently
      value = case value
              when /\A\d+\z/ then value.to_i
              when 'true' then true
              when 'false' then false
              else value
              end
      @conditions[column] = value
      self  # chainable
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    name.to_s.match?(/\Aby_[a-z_]+_[a-z_]+\z/) || super
  end

  def to_sql
    return "SELECT * FROM #{table_name}" if @conditions.empty?
    clauses = @conditions.map { |k, v| "#{k} = #{v.inspect}" }.join(' AND ')
    "SELECT * FROM #{table_name} WHERE #{clauses}"
  end

  private

  def table_name
    @model_class.name.downcase + 's'
  end
end

class User; end

query = QueryBuilder.new(User)
  .by_status_active
  .by_role_admin

query.to_sql
# => "SELECT * FROM users WHERE status = \"active\" AND role = \"admin\""
```

---

**S12. Create a class that uses `ObjectSpace` to enforce a singleton-per-key pattern.**

```ruby
class Registry
  @instances = {}

  def self.for(key)
    @instances[key] ||= new(key)
  end

  def self.instance_count
    @instances.size
  end

  def self.all_instances
    # Use ObjectSpace to verify our tracking matches reality
    live = ObjectSpace.each_object(self).to_a
    { tracked: @instances.values, live_count: live.count }
  end

  def initialize(key)
    @key = key
  end

  attr_reader :key
end

r1 = Registry.for(:database)
r2 = Registry.for(:database)
r3 = Registry.for(:cache)

r1.equal?(r2)           # => true (same object)
r1.equal?(r3)           # => false
Registry.instance_count # => 2
```

---

**S13. Implement a simple event system (pub/sub) where subscribers are registered using DSL-style class declarations.**

```ruby
module EventEmitter
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@listeners, Hash.new { |h, k| h[k] = [] })
  end

  module ClassMethods
    def on(event, method_name = nil, &block)
      handler = method_name ? instance_method(method_name) : block
      @listeners[event] << handler
    end

    def listeners
      @listeners
    end
  end

  def emit(event, *args)
    self.class.listeners[event].each do |handler|
      if handler.is_a?(UnboundMethod)
        handler.bind(self).call(*args)
      else
        instance_exec(*args, &handler)
      end
    end
  end
end

class OrderProcessor
  include EventEmitter

  on :created do |order|
    puts "Sending confirmation email for order #{order[:id]}"
  end

  on :created do |order|
    puts "Updating inventory for order #{order[:id]}"
  end

  on :cancelled do |order|
    puts "Processing refund for order #{order[:id]}"
  end

  def process(order)
    # ... processing logic
    emit(:created, order)
  end
end

processor = OrderProcessor.new
processor.process({ id: 123, amount: 99.99 })
# Sending confirmation email for order 123
# Updating inventory for order 123
```

---

**S14. Explain how you'd use refinements to add JSON serialization to third-party classes without monkey patching.**

```ruby
module JsonSerializable
  refine Time do
    def to_json_value
      iso8601
    end
  end

  refine Symbol do
    def to_json_value
      to_s
    end
  end

  refine NilClass do
    def to_json_value
      "null"
    end
  end

  refine Integer do
    def to_json_value
      to_s
    end
  end

  refine String do
    def to_json_value
      "\"#{self}\""
    end
  end

  refine Array do
    def to_json_value
      "[#{map(&:to_json_value).join(',')}]"
    end
  end

  refine Hash do
    def to_json_value
      pairs = map { |k, v| "\"#{k}\":#{v.to_json_value}" }
      "{#{pairs.join(',')}}"
    end
  end
end

module MySerializer
  using JsonSerializable

  def self.serialize(data)
    data.to_json_value
  end
end

MySerializer.serialize({ name: "Alice", created: Time.now, tags: [:admin, :user] })
# => '{"name":"Alice","created":"2026-03-17T...","tags":["admin","user"]}'
```

---

**S15. Build a `before_action`/`after_action` callback system similar to Rails controllers using metaprogramming.**

```ruby
module Callbacks
  def self.included(base)
    base.extend(ClassMethods)
    base.instance_variable_set(:@before_actions, Hash.new { |h, k| h[k] = [] })
    base.instance_variable_set(:@after_actions, Hash.new { |h, k| h[k] = [] })
  end

  module ClassMethods
    def before_action(callback, only: nil, except: nil)
      targets = resolve_targets(only, except)
      targets.each { |m| @before_actions[m] << callback }
    end

    def after_action(callback, only: nil, except: nil)
      targets = resolve_targets(only, except)
      targets.each { |m| @after_actions[m] << callback }
    end

    def method_added(method_name)
      return if @wrapping  # prevent infinite loop
      return unless @before_actions.key?(method_name) || @after_actions.key?(method_name)

      @wrapping = true
      wrap_with_callbacks(method_name)
      @wrapping = false
    end

    private

    def wrap_with_callbacks(method_name)
      original = instance_method(method_name)
      befores  = @before_actions[method_name].dup
      afters   = @after_actions[method_name].dup

      define_method(method_name) do |*args, &block|
        befores.each { |cb| send(cb) }
        result = original.bind(self).call(*args, &block)
        afters.each  { |cb| send(cb) }
        result
      end
    end

    def resolve_targets(only, except)
      return Array(only) if only
      return (instance_methods(false) - Array(except)) if except
      instance_methods(false)
    end
  end
end

class ApplicationController
  include Callbacks

  before_action :authenticate, only: [:index, :show]
  after_action  :log_request,  only: [:index]

  def authenticate
    puts "Authenticating..."
  end

  def log_request
    puts "Logging request..."
  end

  def index
    puts "index action"
  end

  def show
    puts "show action"
  end
end

ctrl = ApplicationController.new
ctrl.index
# Authenticating...
# index action
# Logging request...
ctrl.show
# Authenticating...
# show action
```
