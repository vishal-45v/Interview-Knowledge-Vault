# Chapter 10: Ruby Internals — Follow-Up Traps

## Trap 1: GIL Doesn't Make All Operations Atomic

**The trap:** Developers assume "Ruby has a GIL, so all operations are thread-safe." The GIL prevents TWO threads from executing Ruby bytecode simultaneously, but compound operations (check-then-act) can still have race conditions because the GIL can be released BETWEEN individual bytecode instructions.

```ruby
# WRONG assumption: += is atomic in Ruby
counter = 0
threads = 100.times.map do
  Thread.new { 1000.times { counter += 1 } }
end
threads.each(&:join)
puts counter  # NOT guaranteed to be 100,000!

# Why: counter += 1 compiles to THREE bytecode instructions:
# 1. getlocal counter   (read current value)
# GIL can be released here, thread B reads same value
# 2. putobject 1
# 3. opt_plus           (add 1)
# Thread B also adds 1, both write back: 1 instead of 2 lost!
# 4. setlocal counter   (write back)

# FIX: use Mutex for compound operations
mutex   = Mutex.new
counter = 0
threads = 100.times.map do
  Thread.new { 1000.times { mutex.synchronize { counter += 1 } } }
end
threads.each(&:join)
puts counter  # => 100,000 guaranteed

# Thread-safe atomic operations use Concurrent::AtomicFixnum:
require 'concurrent'
counter = Concurrent::AtomicFixnum.new(0)
100.times.map { Thread.new { 1000.times { counter.increment } } }.each(&:join)
puts counter.value  # => 100,000
```

---

## Trap 2: `GC.compact` Can Break C Extensions That Store Raw Pointers

**The trap:** When `GC.compact` moves objects, all Ruby-level references are updated. But C extensions that cache raw C pointers to Ruby objects (without using `rb_gc_mark` or write barriers) will have dangling pointers after compaction.

```ruby
# This is mostly a concern for C extension authors, but
# affects Ruby developers when using gems that haven't been updated

# Check if a gem is compaction-safe:
# Look for `gc_compact` in the gem's C source
# or check if the gem passes the following test:

def test_compaction_safety
  before = {}
  ObjectSpace.each_object { |o| before[o.object_id] = o.class }

  GC.compact

  after = {}
  ObjectSpace.each_object { |o| after[o.object_id] = o.class }

  # If the application still works after GC.compact, the gem handles it correctly
  puts "Compaction: OK"
rescue => e
  puts "Compaction unsafe: #{e.class}: #{e.message}"
end

# Safe to call GC.compact in Ruby 2.7+ for Ruby-only objects
# Unsafe: old C extensions using raw VALUE* caches

# Verification in Puma:
# Puma 5.5+ supports compaction-safe workers
# config.rb: worker_boot_hook { GC.compact }
```

---

## Trap 3: Pattern Matching `=>` Raises `NoMatchingPatternError` — Not Returns `false`

**The trap:** Using `=>` (the rightward assignment pattern) raises an exception if the pattern doesn't match, unlike `in` inside `case` which can fall through to the next branch.

```ruby
# case/in — falls through if no match
case { name: "Alice", role: :viewer }
in { role: :admin }
  "admin"
in { role: :viewer }
  "viewer"         # ← reaches here
end

# => (rightward assignment) — RAISES if no match
{ name: "Alice", role: :viewer } => { role: :admin }
# => NoMatchingPatternError (not NoMatchingPatternKeyError)

# TRAP: assuming => will just return nil or false
def check_admin(data)
  data => { role: :admin }  # RAISES if role != :admin!
  true
rescue NoMatchingPatternError
  false
end
# This works, but it's using exceptions for flow control (bad)

# BETTER: use case/in for conditional matching
def check_admin(data)
  case data
  in { role: :admin }
    true
  else
    false
  end
end

# => is for one-line GUARANTEED deconstruction (you know the shape matches):
response = { status: 200, body: "ok" }
response => { status:, body: }  # deconstructs into local vars status, body
puts "#{status}: #{body}"       # status = 200, body = "ok"
# Raises if the hash doesn't match the pattern
```

---

## Trap 4: Ractor Isolation — Most Gem Classes Are Not Ractor-Safe

**The trap:** Attempting to pass mutable objects (or objects from classes that use mutable class-level state) to a Ractor raises `Ractor::IsolationError`. Most Ruby gems are NOT Ractor-safe.

```ruby
# TRAP: passing a regular (mutable) Hash to a Ractor
config = { database_url: "postgres://..." }
r = Ractor.new(config) { |c| c[:database_url] }
# => Ractor::IsolationError: can not share mutable object: {:database_url=>"postgres://..."}

# FIX: freeze the object first
config = { database_url: "postgres://..." }.freeze
r = Ractor.new(config) { |c| c[:database_url] }
# => Works!

# TRAP: using ActiveRecord or other stateful gems inside a Ractor
r = Ractor.new { User.find(1) }
# Ractor::IsolationError — ActiveRecord uses global connection pools,
# callbacks, class-level state → all mutable and non-Ractor-safe

# SAFE: pure computation Ractors with frozen/primitive data
numbers = (1..1_000_000).to_a.freeze
r = Ractor.new(numbers) { |nums| nums.sum { |n| n * n } }
r.take  # => works

# Ractor.make_shareable: deep-freezes an object for sharing
data = Ractor.make_shareable({ key: "value", nums: [1, 2, 3] })
# Now it's shareable, but also deeply frozen — can't mutate after

# Practical limitation (2026): Ractors are only practical for:
# - Pure numeric computation
# - String processing with frozen strings
# - Code with NO external dependencies on non-Ractor-safe classes
```

---

## Trap 5: `Data.define` Instances Are Always Frozen — No After-Initialization Changes

**The trap:** Ruby 3.2's `Data` class creates immutable objects. Unlike Struct, you cannot modify fields after creation, and after-create hooks that try to set attributes will fail.

```ruby
# TRAP: trying to modify a Data instance
Point = Data.define(:x, :y)
p = Point.new(x: 1, y: 2)
p.instance_variable_set(:@x, 5)
# => FrozenError: can't modify frozen Point: #<data Point x=1, y=2>

# TRAP: forgetting .with() for "updates"
p2 = p
p2.x = 5  # => NoMethodError: undefined method 'x='

# CORRECT: use .with() for non-destructive "updates"
p3 = p.with(x: 5)  # => Point(x: 5, y: 2)
p.x                # => 1 (original unchanged)

# TRAP: subclassing and adding mutable state
class ColorPoint < Point  # => TypeError: cannot subclass Data class

# Data classes cannot be subclassed
# Instead, use Data.define again with more fields:
ColorPoint = Data.define(:x, :y, :color)

# TRAP: trying to use Data like a Struct for accumulator patterns
# WRONG:
Point = Data.define(:x, :y)
total = Point.new(x: 0, y: 0)
points.each { |p| total = total.with(x: total.x + p.x) }  # creates N objects!
# Each .with() creates a new object — memory allocation in a loop

# BETTER for accumulation: use Struct, or plain Ruby variables:
total_x = 0
total_y = 0
points.each { |p| total_x += p.x; total_y += p.y }
```

---

## Trap 6: `require` Caches Results — Calling It Twice Doesn't Reload

**The trap:** `require` registers the loaded file in `$LOADED_FEATURES` and returns `false` on subsequent calls for the same file, without reloading it. Code that changes file content and re-requires it doesn't pick up the changes.

```ruby
require 'json'  # => true (loaded)
require 'json'  # => false (already in $LOADED_FEATURES, NOT reloaded)

# TRAP in development: modifying a file and expecting require to reload it
File.write('/tmp/test_module.rb', 'module TestMod; VALUE = 1; end')
require '/tmp/test_module.rb'  # => true, VALUE = 1

File.write('/tmp/test_module.rb', 'module TestMod; VALUE = 2; end')
require '/tmp/test_module.rb'  # => false! VALUE is STILL 1

# FIX: use load (always reloads, no caching):
load '/tmp/test_module.rb'  # reloads every time
TestMod::VALUE  # => 2

# Rails uses load (via Zeitwerk) in development for this reason
# In production: require (load once, fast)

# Checking what's loaded:
puts $LOADED_FEATURES.grep(/json/)
# => ["/usr/lib/ruby/3.2.0/json.rb"]

# Force reload by removing from $LOADED_FEATURES:
$LOADED_FEATURES.delete_if { |f| f.include?('my_module') }
require 'my_module'  # now reloads
```

---

## Trap 7: `Kernel#pp` vs `puts` vs `p` — Different Encoding of Output

**The trap:** Using `puts` for inspecting complex objects gives misleading output; `p` is better for debugging; `pp` is best for deep structures.

```ruby
obj = { name: "Alice", roles: [:admin, :editor], meta: { score: 9.5, active: true } }

puts obj
# {} — useless! puts calls to_s, which for Hash returns {}

p obj
# {:name=>"Alice", :roles=>[:admin, :editor], :meta=>{:score=>9.5, :active=>true}}
# Better! p calls inspect

require 'pp'
pp obj
# {:name=>"Alice",
#  :roles=>[:admin, :editor],
#  :meta=>{:score=>9.5, :active=>true}}
# Best for complex nested objects — pretty-printed with proper indentation

# For strings:
str = "Hello\nWorld\t!"
puts str   # Hello        (newline is interpreted, tab shown as tab)
           # World	!
p str      # "Hello\nWorld\t!"  (shows escape sequences, shows it's a String)

# For nil:
puts nil   # (empty line — invisible!)
p nil      # nil  (explicitly shows the value)

# Rule: use p during debugging, puts for user output, pp for complex objects
# Never use puts to inspect objects in debugging — you'll see misleading output
```

---

## Trap 8: `respond_to?` Returns Wrong Results for BasicObject Subclasses

**The trap:** `BasicObject` doesn't have `respond_to?` — it's defined in `Kernel` which is mixed into `Object`. Calling `respond_to?` on a `BasicObject` instance raises `NoMethodError`.

```ruby
class MyProxy < BasicObject
  def double(x)
    x * 2
  end
end

proxy = MyProxy.new
proxy.double(5)       # => 10

proxy.respond_to?(:double)
# => NoMethodError: undefined method 'respond_to?' for #<MyProxy>
# Because BasicObject doesn't include Kernel!

# FIX: define respond_to? manually
class MyProxy < BasicObject
  def double(x)
    x * 2
  end

  def respond_to?(method_name, include_private = false)
    [:double].include?(method_name) || super
  rescue ::NoMethodError
    # super eventually calls BasicObject#respond_to? which doesn't exist
    false
  end
end

# OR: delegate to the real respond_to?
class FullProxy < BasicObject
  def initialize(target)
    @target = target
  end

  def respond_to?(method_name, include_private = false)
    @target.respond_to?(method_name, include_private)
  end

  def method_missing(name, *args, &block)
    @target.__send__(name, *args, &block)
  end
end

proxy = FullProxy.new([1, 2, 3])
proxy.respond_to?(:length)  # => true (delegates to Array's respond_to?)
```

---

## Trap 9: `Encoding::UTF_8` vs `"UTF-8"` String — Both Work But Have Different Performance

**The trap:** Passing encoding names as strings instead of `Encoding` constants requires a lookup every time. It's a minor performance trap but worth knowing for hot paths.

```ruby
# String name lookup (slower — requires lookup in encoding table):
"hello".encode("UTF-8")
File.read("data.txt", encoding: "UTF-8")

# Constant reference (faster — direct Encoding object):
"hello".encode(Encoding::UTF_8)
File.read("data.txt", encoding: Encoding::UTF_8)

# Checking encoding:
puts Encoding::UTF_8.class     # => Encoding
puts Encoding::UTF_8.name      # => "UTF-8"
puts Encoding::UTF_8.to_s      # => "UTF-8"
puts Encoding::UTF_8.inspect   # => #<Encoding:UTF-8>

# Common encoding constants:
Encoding::UTF_8         # most common
Encoding::ASCII         # 7-bit ASCII
Encoding::ASCII_8BIT    # binary data (alias: Encoding::BINARY)
Encoding::ISO_8859_1    # Latin-1
Encoding::Windows_1252  # Windows Western European
Encoding::UTF_16        # UTF-16 (BOM required)
Encoding::UTF_16BE      # UTF-16 big-endian
Encoding::UTF_16LE      # UTF-16 little-endian

# Trap: binary strings and encoding:
binary_data = "\xFF\xFE\x00\x48"  # raw bytes
binary_data.encoding  # => ASCII-8BIT (Ruby's "binary" encoding)
# Trying to convert directly to UTF-8 may raise Encoding::UndefinedConversionError
binary_data.encode("UTF-8")  # => Encoding::UndefinedConversionError

# Safe conversion:
binary_data.encode("UTF-8", invalid: :replace, undef: :replace, replace: "?")
```

---

## Trap 10: `TracePoint` Has Significant Overhead — Never Leave It Enabled in Production

**The trap:** `TracePoint` hooks into every method call, line execution, or event in Ruby's execution. Leaving it enabled in production causes severe slowdowns.

```ruby
# DANGER: enabling TracePoint and forgetting to disable it
tp = TracePoint.new(:call) { |t| log_call(t) }
tp.enable

# ... some code ...
# FORGOT to call tp.disable!

# Performance impact:
require 'benchmark'

tp = TracePoint.new(:call) { }  # empty callback

Benchmark.measure do
  1_000_000.times { "hello".upcase }
end
# Without TracePoint: ~0.08s

tp.enable
Benchmark.measure do
  1_000_000.times { "hello".upcase }
end
# With TracePoint: ~0.8s (10x slower!)
tp.disable

# SAFE pattern: always use ensure or block form
TracePoint.new(:call) { |t| puts t.method_id }.enable do
  # TracePoint is ONLY active inside this block
  run_code_to_trace
end
# Automatically disabled after the block

# Or explicit enable/disable with ensure:
tp = TracePoint.new(:call) { }
begin
  tp.enable
  run_code_to_trace
ensure
  tp.disable  # ALWAYS disable, even on exception
end

# For production monitoring: use StackProf instead (sampling, ~1% overhead)
# TracePoint is only for development/debugging
```
