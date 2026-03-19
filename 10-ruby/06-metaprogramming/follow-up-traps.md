# Chapter 06: Metaprogramming — Follow-Up Traps

## Trap 1: `method_missing` Without `respond_to_missing?` Breaks Duck Typing

**The trap:** Implementing `method_missing` but forgetting `respond_to_missing?` means `respond_to?` returns `false` even for methods you handle. This silently breaks any code that checks capability before calling.

```ruby
# BAD — broken duck typing
class BadProxy
  def method_missing(name, *args, &block)
    return "handled" if name == :special_method
    super
  end
  # NO respond_to_missing?
end

proxy = BadProxy.new
proxy.special_method          # => "handled"  (works)
proxy.respond_to?(:special_method)  # => false  (WRONG!)

# This common pattern silently breaks:
def process(duck)
  duck.special_method if duck.respond_to?(:special_method)
end
process(proxy)  # => nil — never called because respond_to? returns false

# GOOD
class GoodProxy
  def method_missing(name, *args, &block)
    return "handled" if name == :special_method
    super
  end

  def respond_to_missing?(name, include_private = false)
    name == :special_method || super
  end
end

good = GoodProxy.new
good.respond_to?(:special_method)  # => true
good.method(:special_method)       # => #<Method: ...>  (also works now)
```

**Interview follow-up:** "What else breaks besides `respond_to?`?"
Answer: `method(:special_method)` raises `NameError`, `respond_to_missing?` being absent also breaks `Marshal.dump`, some reflection libraries, and documentation generators.

---

## Trap 2: `send` Bypasses Private Methods — `public_send` Respects Visibility

**The trap:** Using `send` with user-controlled input can accidentally (or maliciously) call private methods.

```ruby
class UserAccount
  def display_name
    "Alice"
  end

  private

  def reset_password!
    puts "Password reset executed!"
  end

  def delete_account!
    puts "Account deleted!"
  end
end

user = UserAccount.new

# DANGEROUS: user input could be "reset_password!" or "delete_account!"
user_input = "reset_password!"
user.send(user_input.to_sym)  # => "Password reset executed!" — security hole!

# SAFE: public_send raises NoMethodError for private methods
user.public_send(user_input.to_sym)
# => NoMethodError: private method 'reset_password!' called for an instance of UserAccount

# SAFEST: whitelist allowed methods
ALLOWED = %i[display_name].freeze
raise "Forbidden" unless ALLOWED.include?(user_input.to_sym)
user.public_send(user_input.to_sym)
```

**Interview follow-up:** "What about `__send__`?"
Answer: `__send__` behaves like `send` (bypasses private). Its purpose is to be unclobberable — if someone reopens a class and redefines `send`, `__send__` still works. Use it in proxy/delegation code where reliability matters more than visibility enforcement.

---

## Trap 3: Open Class Modifications Are Global and Affect All Instances Everywhere

**The trap:** Monkey patching a class in one file silently changes behavior across your entire application, including third-party gems that depend on the original behavior.

```ruby
# In some_extension.rb loaded early in the boot process
class String
  def blank?
    strip.empty?
  end

  # BUG: accidentally redefining a core method
  def split(delimiter = nil, limit = nil)
    puts "DEBUG: String#split called"  # seemed harmless during dev
    super
  end
end

# Now EVERY string split in your app AND every gem is intercepted
"a,b,c".split(',')  # prints "DEBUG: String#split called" — in production!
CSV.parse(data)      # also prints it, because CSV uses String#split

# Refinements are the safe alternative
module SafeDebugExtensions
  refine String do
    def split(*)
      puts "DEBUG: split called"
      super
    end
  end
end

module MyDebugModule
  using SafeDebugExtensions
  # Only INSIDE this module is String#split intercepted
  "a,b,c".split(',')  # prints debug
end

"a,b,c".split(',')  # normal, no output
```

**Interview follow-up:** "When IS monkey patching acceptable?"
Answer: When adding non-conflicting, well-named utility methods; when you own or control the class; during testing (test helpers); and when refinements would make the API too awkward. Always document it and consider gem scope.

---

## Trap 4: `const_get` With String vs Symbol Behaves Differently for Namespaced Constants

**The trap:** `const_get("Foo::Bar")` traverses the namespace. `const_get(:Foo)` only looks up a simple name. Mixing them up causes surprising `NameError` results.

```ruby
module Outer
  module Inner
    class Thing; end
  end
end

# String with :: — traverses namespace correctly
Object.const_get("Outer::Inner::Thing")  # => Outer::Inner::Thing

# Symbol — does NOT traverse ::
Object.const_get(:"Outer::Inner::Thing")  # => NameError! (on most Ruby versions)

# Safe pattern for user-provided class names
def safe_constantize(name)
  name.to_s.split("::").reduce(Object) do |mod, part|
    mod.const_get(part, false)  # false = don't search ancestors
  end
rescue NameError
  nil
end

safe_constantize("Outer::Inner::Thing")  # => Outer::Inner::Thing
safe_constantize("Nonexistent::Class")   # => nil

# The second argument to const_get matters:
module A
  CONST = "A"
end
module B
  include A
  # const_get with inherit=true (default) searches ancestors
  const_get(:CONST)         # => "A"  (found via A)
  const_get(:CONST, false)  # => NameError (not directly in B)
end
```

---

## Trap 5: `method_added` Creates an Infinite Loop When You Define Methods Inside It

**The trap:** `method_added` is called every time a method is defined, including methods you define inside `method_added` itself.

```ruby
# INFINITE LOOP — don't do this
class Broken
  def self.method_added(method_name)
    define_method(:"#{method_name}_logged") do  # triggers method_added again!
      puts "called #{method_name}_logged"       # → triggers for "hello_logged"
    end                                          # → defines "hello_logged_logged"
  end                                            # → infinite recursion → stack overflow

  def hello; end
end

# SAFE — guard with a flag
class Instrumented
  @instrumenting = false

  def self.method_added(method_name)
    return if @instrumenting
    return if method_name.to_s.end_with?('_original')

    @instrumenting = true
    original_method = instance_method(method_name)
    define_method(method_name) do |*args, &block|
      puts "Calling: #{method_name}"
      original_method.bind(self).call(*args, &block)
    end
    @instrumenting = false
  end

  def greet(name)
    "Hello, #{name}"
  end

  def farewell(name)
    "Goodbye, #{name}"
  end
end

Instrumented.new.greet("Alice")
# Calling: greet
# => "Hello, Alice"
```

---

## Trap 6: `define_method` Closure Captures Variables by Reference, Not by Value

**The trap:** In a loop, all methods defined with `define_method` may capture the **same** variable reference, resulting in all methods using the final loop value.

```ruby
# TRAP: all methods print "c"
class Broken
  letters = ['a', 'b', 'c']
  letters.each do |letter|
    define_method("print_#{letter}") do
      puts letter  # closes over the variable `letter`, not its current value
    end
  end
end

b = Broken.new
b.print_a  # => "c"  (!!!)
b.print_b  # => "c"  (!!!)
b.print_c  # => "c"

# Ruby's each block creates a NEW binding per iteration, so this actually works
# correctly in modern Ruby. The trap is real in older code using for loops:

class AlsoBroken
  for letter in ['a', 'b', 'c']  # for loop does NOT create new scope!
    define_method("print_#{letter}") { puts letter }
  end
end

AlsoBroken.new.print_a  # => "c" (captures final value)

# SAFE: always use each/map (block-based iteration) with define_method
class Safe
  ['a', 'b', 'c'].each do |letter|
    define_method("print_#{letter}") { puts letter }
  end
end

Safe.new.print_a  # => "a"
Safe.new.print_b  # => "b"
```

---

## Trap 7: `instance_eval` vs `class_eval` — Wrong Context Defines Wrong Kind of Method

**The trap:** Using `instance_eval` on a class when you mean `class_eval` causes methods to be defined as class-level singleton methods, not instance methods.

```ruby
class MyClass; end

# instance_eval on a CLASS — methods become singleton (class-level) methods
MyClass.instance_eval do
  def class_method
    "I am a class method"
  end
end

MyClass.class_method      # => "I am a class method"
MyClass.new.class_method  # => NoMethodError

# class_eval on a CLASS — methods become instance methods
MyClass.class_eval do
  def instance_method_here
    "I am an instance method"
  end
end

MyClass.new.instance_method_here   # => "I am an instance method"
MyClass.instance_method_here       # => NoMethodError

# Mnemonic:
# instance_eval  → in the context of the object's singleton class
#                  (for a Class object, that means class-level methods)
# class_eval     → in the context of the class itself
#                  (defines methods available to instances)
```

---

## Trap 8: `method_missing` Is Slow — Define the Method on First Call

**The trap:** Every `method_missing` invocation is slower than a regular method call because Ruby walks the entire ancestor chain before giving up. In hot loops, this causes measurable degradation.

```ruby
# SLOW — method_missing called every time
class SlowDynamic
  def method_missing(name, *args)
    if name.to_s.start_with?('find_')
      attr = name.to_s.sub('find_', '')
      "Looking for #{attr} with #{args.first}"
    else
      super
    end
  end
end

# FAST — define the real method on first miss (ghostbusting pattern)
class FastDynamic
  def method_missing(name, *args)
    if name.to_s.start_with?('find_')
      attr = name.to_s.sub('find_', '')
      # Define the method so next call goes directly to it
      self.class.define_method(name) do |*a|
        "Looking for #{attr} with #{a.first}"
      end
      send(name, *args)  # call the newly defined method
    else
      super
    end
  end

  def respond_to_missing?(name, include_private = false)
    name.to_s.start_with?('find_') || super
  end
end

fd = FastDynamic.new
fd.find_name("Alice")   # method_missing fires, defines method, calls it
fd.find_name("Bob")     # direct method call now — fast!
```

---

## Trap 9: `eval` With String Loses Debugging Context and Is a Security Risk

**The trap:** String eval loses the file/line context for backtraces and opens a code injection vulnerability.

```ruby
# BAD: no file/line context, security risk
class_eval("def #{method_name}; end")
# Backtraces show "(eval):1" — useless for debugging

# BETTER: pass __FILE__ and __LINE__ for useful backtraces
class_eval(<<~RUBY, __FILE__, __LINE__ + 1)
  def #{method_name}
    "dynamic method"
  end
RUBY
# Backtrace now shows actual file and line number

# BEST: use define_method to avoid string eval entirely
define_method(method_name) { "dynamic method" }
# Full Ruby stack trace, closures work, no security risk

# String eval is only necessary when the generated code is complex
# and can't be expressed as a block (rare in practice)
```

---

## Trap 10: Frozen Objects Cannot Be Extended at Runtime

**The trap:** Calling `extend` on a frozen object raises a `FrozenError`. Frozen objects are common with interned symbols and frozen string literals.

```ruby
module Greeter
  def greet
    "hello"
  end
end

str = "hello".freeze
str.extend(Greeter)
# => FrozenError: can't modify frozen String: "hello"

# Also fails for frozen symbols
:foo.freeze  # symbols are often already frozen
:foo.extend(Greeter)  # => FrozenError

# Practical implication: you can't extend frozen object with modules after creation
# Solution: extend before freezing, or use a wrapper
class GreetableString
  include Greeter
  def initialize(str) = @str = str
  def to_s = @str
end
```

---

## Trap 11: `prepend` Changes `ancestors` — Affects `super` Calls Unexpectedly

**The trap:** When you prepend a module, `super` in the original method calls the prepended module's version if the module overrides it — not the parent class.

```ruby
module Logging
  def process
    puts "Before process"
    result = super  # calls Process#process (the prepended-into class)
    puts "After process"
    result
  end
end

class Processor
  prepend Logging

  def process
    puts "Processing..."
    42
  end
end

Processor.ancestors
# => [Logging, Processor, Object, Kernel, BasicObject]

Processor.new.process
# Before process
# Processing...
# After process
# => 42

# TRAP: if a subclass also includes Logging, super chains differently
class SpecialProcessor < Processor
  prepend Logging  # Logging is now prepended AGAIN

  def process
    puts "Special processing"
    super  # calls Logging#process again → infinite loop!
  end
end
```

---

## Trap 12: `ObjectSpace` Is Disabled or Restricted in Some Ruby Implementations

**The trap:** Code that depends on `ObjectSpace.each_object` works in MRI (CRuby) but may be unavailable, incomplete, or behave differently in JRuby, TruffleRuby, or Rubinius.

```ruby
# Works reliably in MRI:
ObjectSpace.each_object(MyClass).to_a  # returns all live instances

# JRuby: ObjectSpace is disabled by default for performance
# You must start JRuby with: jruby --enable-objectspace
# Or: require 'java'; com.jruby.JRuby.each_object...

# Cross-runtime safe alternative: maintain your own registry
module Trackable
  def self.included(base)
    base.instance_variable_set(:@instances, [])
    base.extend(ClassMethods)
  end

  module ClassMethods
    def track(instance)
      # Use WeakRef to avoid preventing GC
      require 'weakref'
      @instances.reject!(&:weakref_alive?.method(:!)) rescue nil
      @instances << WeakRef.new(instance)
    end

    def live_instances
      @instances.select(&:weakref_alive?).map { |ref| ref.__getobj__ rescue nil }.compact
    end
  end

  def initialize(*)
    super
    self.class.track(self)
  end
end
```
