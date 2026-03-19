# Chapter 10: Ruby Internals — Scenario Questions

**S1. Use `RubyVM::InstructionSequence` to compare the bytecode of two implementations. What does it tell you?**

```ruby
require 'pp'

# Implementation 1: string interpolation
iseq1 = RubyVM::InstructionSequence.compile(<<~RUBY)
  def greet(name)
    "Hello, #{name}!"
  end
RUBY

# Implementation 2: string concatenation
iseq2 = RubyVM::InstructionSequence.compile(<<~RUBY)
  def greet(name)
    "Hello, " + name + "!"
  end
RUBY

puts "=== Interpolation ==="
puts iseq1.disasm

puts "\n=== Concatenation ==="
puts iseq2.disasm

# Interpolation bytecode (roughly):
# putstring "Hello, "
# getlocal name
# putstring "!"
# concatstrings 3     ← single instruction, optimized by VM
# leave

# Concatenation bytecode:
# putstring "Hello, "
# getlocal name
# opt_plus            ← method call to String#+
# putstring "!"
# opt_plus            ← another method call
# leave

# Conclusion: interpolation compiles to `concatstrings` (VM-level optimized)
# Concatenation calls String#+ twice (two method dispatch cycles)
# → Interpolation is faster AND allocates fewer intermediate strings

# Practical use: verify optimizations
def check_inlining(method_code)
  iseq = RubyVM::InstructionSequence.compile(method_code)
  puts iseq.disasm
  # Look for opt_* instructions (VM-optimized) vs send/invokemethod (method calls)
end
```

---

**S2. Demonstrate the performance difference between Ruby versions using pattern matching as an example.**

```ruby
# Ruby 3.0: pattern matching production-ready
# Before 3.0: only case/when (no deconstruct protocol)

# Feature detection for cross-version code:
def ruby_supports_pattern_matching?
  RUBY_VERSION >= "3.0.0"
end

# Pattern matching with custom deconstruct:
class Response
  attr_reader :status, :body, :headers

  def initialize(status:, body:, headers: {})
    @status  = status
    @body    = body
    @headers = headers
  end

  def deconstruct_keys(keys)
    { status: @status, body: @body, headers: @headers }
  end
end

def handle_response(response)
  case response
  in { status: 200, body: String => body }
    { success: true, data: JSON.parse(body) }
  in { status: 201, body: String => body }
    { success: true, data: JSON.parse(body), created: true }
  in { status: 401 }
    { success: false, error: "Unauthorized" }
  in { status: 404 }
    { success: false, error: "Not found" }
  in { status: (500..) => status }
    { success: false, error: "Server error: #{status}" }
  end
end

# Ruby 3.2 improvement: pattern matching performance
# Benchmark: find pattern (scanning arrays for sub-patterns)
require 'benchmark/ips'

data = [1, 2, "ruby", 4, 5, "rails", 7]

Benchmark.ips do |x|
  x.report("find pattern") do
    case data
    in [*, String => s, *]
      s
    end
  end

  x.report("Array#find") do
    data.find { |x| x.is_a?(String) }
  end
  x.compare!
end
```

---

**S3. Implement a Ruby 3.2 `Data` class and show how it differs from `Struct`.**

```ruby
# Data.define (Ruby 3.2+) — immutable value objects
Point = Data.define(:x, :y)

p = Point.new(x: 1, y: 2)
p.x         # => 1
p.frozen?   # => true (ALWAYS frozen, unlike Struct)

# Non-destructive update (like Elixir/Haskell):
p2 = p.with(x: 5)  # => Point(x: 5, y: 2)
p.equal?(p2)        # => false (new object)
p.x                 # => 1 (original unchanged)

# Value equality:
Point.new(x: 1, y: 2) == Point.new(x: 1, y: 2)  # => true

# Can add custom methods:
ColorPoint = Data.define(:x, :y, :color) do
  def distance_to_origin
    Math.sqrt(x**2 + y**2)
  end

  def to_s
    "(#{x}, #{y}) in #{color}"
  end
end

cp = ColorPoint.new(x: 3, y: 4, color: :red)
cp.distance_to_origin  # => 5.0

# Struct vs Data comparison:
# Struct:
Point_mutable = Struct.new(:x, :y)
ps = Point_mutable.new(1, 2)
ps.x = 5           # works! Struct is mutable
ps.frozen?         # => false (unless .freeze called)

# Data:
pd = Point.new(x: 1, y: 2)
pd.x = 5           # => NoMethodError (no setter defined)
pd.frozen?         # => true

# Pattern matching with Data:
case ColorPoint.new(x: 0, y: 5, color: :blue)
in ColorPoint[x: 0, y:, color:]
  "On y-axis at #{y}, color: #{color}"
end

# Use Data when: value objects, DTOs, records that shouldn't change
# Use Struct when: mutable value objects, quick ad-hoc structures
```

---

**S4. Write code that demonstrates the difference between threads, fibers, and Ractors.**

```ruby
require 'benchmark'

# THREADS: concurrent but not parallel for CPU (MRI GIL)
def thread_example
  start = Time.now
  threads = 4.times.map do |i|
    Thread.new do
      count = 0
      1_000_000.times { count += 1 }
      count
    end
  end
  threads.map(&:value).sum
  puts "Threads: #{(Time.now - start).round(3)}s"
  # ≈ same as sequential (GIL prevents true parallelism)
end

# FIBERS: cooperative, single-thread, lightweight
def fiber_example
  fibers = 4.times.map do |i|
    Fiber.new do
      count = 0
      1_000_000.times do
        count += 1
        Fiber.yield if count % 10_000 == 0  # cooperatively yield
      end
      count
    end
  end

  # Run fibers interleaved:
  until fibers.all? { |f| f.alive? == false rescue true }
    fibers.each { |f| f.resume rescue nil }
  end
end

# RACTORS: true parallelism (each has its own GVL)
def ractor_example
  start = Time.now
  ractors = 4.times.map do
    Ractor.new do
      count = 0
      1_000_000.times { count += 1 }
      count
    end
  end
  ractors.map(&:take).sum
  puts "Ractors: #{(Time.now - start).round(3)}s"
  # ≈ 4x faster than single-threaded (true parallelism)
end

# I/O-bound comparison (threads WIN here):
require 'net/http'

def io_thread_example
  # Threads release GIL during I/O → effective concurrency
  threads = 10.times.map do
    Thread.new { Net::HTTP.get_response(URI("https://example.com")) }
  end
  threads.map(&:value)
end

# Summary table:
puts <<~TABLE
  Concurrency Model Comparison:
  ┌──────────────┬────────────┬────────────┬──────────────────────────┐
  │ Model        │ CPU Parallel│ I/O Concurrent│ Use Case            │
  ├──────────────┼────────────┼────────────┼──────────────────────────┤
  │ Threads      │ No (GIL)   │ Yes        │ I/O-bound parallelism    │
  │ Fibers       │ No         │ Yes (sched)│ Async I/O, coroutines    │
  │ Ractors      │ Yes        │ Limited    │ CPU-bound parallelism    │
  │ Processes    │ Yes        │ Yes        │ Heavy isolation needed   │
  └──────────────┴────────────┴────────────┴──────────────────────────┘
TABLE
```

---

**S5. Show how Object Shapes in Ruby 3.2+ affect YJIT performance.**

```ruby
# SHAPE-STABLE class (good for YJIT):
class StableUser
  def initialize(name, email, age)
    @name  = name   # same ivar order every time
    @email = email
    @age   = age
  end

  def greeting
    "Hello, #{@name}!"  # YJIT: knows @name is always at index 0
  end
end

# SHAPE-UNSTABLE class (bad for YJIT):
class UnstableUser
  def set(key, value)
    instance_variable_set(:"@#{key}", value)  # different order = different shape
  end

  def greeting
    "Hello, #{@name}!"  # YJIT: can't predict where @name is
  end
end

# Demonstrating the performance difference:
require 'benchmark/ips'

stable_users = 1000.times.map { |i| StableUser.new("User#{i}", "u#{i}@ex.com", 25) }
unstable_users = 1000.times.map do |i|
  u = UnstableUser.new
  if i.even?
    u.set(:name, "User#{i}")
    u.set(:email, "u#{i}@ex.com")
  else
    u.set(:email, "u#{i}@ex.com")  # different order!
    u.set(:name, "User#{i}")
  end
  u
end

Benchmark.ips do |x|
  x.config(warmup: 3, time: 5)

  x.report("stable shapes") do
    stable_users.each { |u| u.greeting }
  end

  x.report("unstable shapes") do
    unstable_users.each { |u| u.greeting }
  end
  x.compare!
end

# With YJIT: stable shapes can be 2-5x faster than unstable shapes
# Without YJIT: difference is smaller (YARV interpreter is less shape-aware)

# Check YJIT stats to see shape-related optimizations:
if defined?(RubyVM::YJIT) && RubyVM::YJIT.enabled?
  stats = RubyVM::YJIT.runtime_stats
  puts "Compiled code runs: #{(stats[:ratio_in_yjit] * 100).round(1)}%"
end
```

---

**S6. Implement a Fiber-based async coroutine scheduler.**

```ruby
# Minimal Fiber scheduler for educational purposes
class SimpleScheduler
  def initialize
    @fibers = []
    @results = {}
  end

  def spawn(&block)
    fiber = Fiber.new(&block)
    @fibers << fiber
    fiber
  end

  def run
    until @fibers.empty?
      @fibers.reject! do |fiber|
        begin
          fiber.resume
          !fiber.alive?
        rescue FiberError
          true  # remove dead fibers
        end
      end
    end
  end

  def yield_to_scheduler
    Fiber.yield
  end
end

class AsyncSimulator
  def self.simulate_io(label, duration)
    steps = (duration * 10).to_i
    steps.times do
      sleep(0.001)   # tiny sleep simulates I/O
      Fiber.yield    # yield to scheduler
    end
    "#{label} done"
  end
end

scheduler = SimpleScheduler.new

f1 = scheduler.spawn { AsyncSimulator.simulate_io("fetch_users", 0.05) }
f2 = scheduler.spawn { AsyncSimulator.simulate_io("fetch_orders", 0.03) }
f3 = scheduler.spawn { AsyncSimulator.simulate_io("fetch_products", 0.07) }

start = Time.now
scheduler.run
elapsed = Time.now - start

puts "Elapsed: #{(elapsed * 1000).round(1)}ms"
# Sequential would take: 50 + 30 + 70 = 150ms
# Cooperative: ~70ms (longest task duration, all run concurrently)
```

---

**S7. Inspect the Ruby GC at runtime to understand heap growth.**

```ruby
# Monitoring GC in a long-running process
class GCMonitor
  def initialize(interval: 100)
    @interval    = interval
    @call_count  = 0
    @snapshots   = []
  end

  def call(block)
    @call_count += 1

    result = block.call

    if @call_count % @interval == 0
      take_snapshot
    end

    result
  end

  def take_snapshot
    stats = GC.stat
    @snapshots << {
      time:              Time.now,
      heap_live_slots:   stats[:heap_live_slots],
      heap_free_slots:   stats[:heap_free_slots],
      heap_pages:        stats[:heap_allocated_pages],
      minor_gc_count:    stats[:minor_gc_count],
      major_gc_count:    stats[:major_gc_count],
      total_allocated:   stats[:total_allocated_objects],
    }
  end

  def report
    return if @snapshots.size < 2

    first = @snapshots.first
    last  = @snapshots.last

    puts "GC Report (#{@snapshots.size} snapshots):"
    puts "  Live slots growth:    #{last[:heap_live_slots] - first[:heap_live_slots]}"
    puts "  Heap pages growth:    #{last[:heap_pages] - first[:heap_pages]}"
    puts "  Minor GCs:            #{last[:minor_gc_count] - first[:minor_gc_count]}"
    puts "  Major GCs:            #{last[:major_gc_count] - first[:major_gc_count]}"
    puts "  Objects allocated:    #{last[:total_allocated] - first[:total_allocated]}"

    # Growth per minor GC (objects surviving per GC cycle):
    minor_gcs = last[:minor_gc_count] - first[:minor_gc_count]
    if minor_gcs > 0
      surviving_per_gc = (last[:heap_live_slots] - first[:heap_live_slots]).to_f / minor_gcs
      puts "  Surviving objs/GC:    #{surviving_per_gc.round(1)}"
      puts "  → #{surviving_per_gc > 100 ? 'Potential memory leak!' : 'Normal'}"
    end
  end
end

monitor = GCMonitor.new(interval: 50)
1000.times do |i|
  monitor.call(-> {
    # Your code here — could be a request processing simulation
    Array.new(100) { "item #{i}" }  # creates 100 strings + 1 array
  })
end
monitor.report
```

---

**S8. Demonstrate how YARV optimizes simple operations with `opt_*` instructions.**

```ruby
# YARV has specialized instructions for common operations
# These skip the full method dispatch cycle

# View YARV bytecode:
def show_bytecode(code)
  iseq = RubyVM::InstructionSequence.compile(code,
    nil, nil, 0,
    {specialized_instruction: true}
  )
  iseq.disasm
end

# Integer addition:
puts show_bytecode("a + b")
# opt_plus  ← specialized: checks if both are Integers, does C-level addition
# No method dispatch if types are known!

# Array indexing:
puts show_bytecode("arr[0]")
# opt_aref  ← specialized: checks if Array, does direct C-level index

# Comparison:
puts show_bytecode("x < y")
# opt_lt    ← specialized comparison

# Without specialization (complex dispatch):
puts show_bytecode("obj.send(:+, other)")
# opt_send_without_block  ← full method lookup

# Practical implication:
require 'benchmark/ips'

arr = [1, 2, 3, 4, 5]
Benchmark.ips do |x|
  x.report("arr[0]")              { arr[0] }          # opt_aref
  x.report("arr.send(:[], 0)")    { arr.send(:[], 0) } # full dispatch
  x.report("arr.public_send")     { arr.public_send(:[], 0) }
  x.compare!
end
# arr[0] is significantly faster (specialized instruction)
# send variants require full method lookup
```

---

**S9. Show the difference in memory layout between Struct and Data and regular classes.**

```ruby
require 'objspace'

# Regular class (RObject with ivar table):
class RegularPoint
  def initialize(x, y)
    @x = x
    @y = y
  end
end

# Struct (optimized ivar storage in C struct):
StructPoint = Struct.new(:x, :y)

# Data (Ruby 3.2+, immutable, similar to Struct):
DataPoint = Data.define(:x, :y)

# Memory comparison:
regular = RegularPoint.new(1, 2)
struct  = StructPoint.new(1, 2)
data    = DataPoint.new(x: 1, y: 2)

puts "Regular class: #{ObjectSpace.memsize_of(regular)} bytes"
puts "Struct:        #{ObjectSpace.memsize_of(struct)} bytes"
puts "Data:          #{ObjectSpace.memsize_of(data)} bytes"
# All similar: ~40-80 bytes depending on Ruby version

# The real difference is in access patterns:
require 'benchmark/ips'

Benchmark.ips do |x|
  x.report("regular @x") { regular.instance_variable_get(:@x) }
  x.report("struct .x")  { struct.x }
  x.report("data .x")    { data.x }
  x.compare!
end
# Struct and Data access is slightly faster (specialized accessor)

# Object identity and equality:
a = RegularPoint.new(1, 2)
b = RegularPoint.new(1, 2)
a == b   # => false (default Object#== checks identity)

s1 = StructPoint.new(1, 2)
s2 = StructPoint.new(1, 2)
s1 == s2  # => true (Struct#== compares values)

d1 = DataPoint.new(x: 1, y: 2)
d2 = DataPoint.new(x: 1, y: 2)
d1 == d2  # => true (Data#== compares values, like Struct)
```

---

**S10. Implement a simple profiler using `TracePoint`.**

```ruby
# TracePoint provides hooks into Ruby's execution events
# Useful for profiling, coverage, tracing, debugging

class MethodProfiler
  def initialize
    @times   = Hash.new { |h, k| h[k] = { count: 0, total_time: 0.0 } }
    @starts  = {}
    @trace   = nil
  end

  def enable
    @trace = TracePoint.new(:call, :return) do |tp|
      key = "#{tp.defined_class}##{tp.method_id}"

      case tp.event
      when :call
        @starts[key] = Process.clock_gettime(Process::CLOCK_MONOTONIC)
      when :return
        start = @starts.delete(key)
        if start
          elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
          @times[key][:count]      += 1
          @times[key][:total_time] += elapsed
        end
      end
    end
    @trace.enable
    self
  end

  def disable
    @trace&.disable
    self
  end

  def report(top_n: 20)
    puts "\n=== Method Profile Report ==="
    @times
      .sort_by { |_, v| -v[:total_time] }
      .first(top_n)
      .each do |method, stats|
        avg = (stats[:total_time] / stats[:count] * 1000).round(3)
        total = (stats[:total_time] * 1000).round(3)
        puts "  #{method}: #{stats[:count]} calls, #{total}ms total, #{avg}ms avg"
      end
  end
end

profiler = MethodProfiler.new.enable

# Code to profile:
require 'json'
100.times { JSON.parse('{"key": "value", "num": 42}') }
100.times { [1, 2, 3].map { |x| x * 2 }.select { |x| x > 2 } }

profiler.disable
profiler.report

# TracePoint can also trace:
# :line       — each line of code executed
# :class      — class/module defined
# :end        — end of class/module
# :raise      — exception raised
# :b_call     — block call
# :b_return   — block return
# :thread_begin, :thread_end — thread events
# :script_compiled — when code is compiled (Ruby 3.2+)
```

---

**S11. Use `ObjectSpace.allocation_sourcefile` to find memory allocation hotspots.**

```ruby
require 'objspace'

ObjectSpace.trace_object_allocations_start

# Code to analyze:
def build_user_list(count)
  count.times.map do |i|
    { id: i, name: "User #{i}", email: "user#{i}@example.com" }
  end
end

users = build_user_list(1000)
puts "Created #{users.size} users"

# Analyze where objects are allocated:
allocation_map = Hash.new(0)

ObjectSpace.each_object do |obj|
  file = ObjectSpace.allocation_sourcefile(obj)
  line = ObjectSpace.allocation_sourceline(obj)
  next unless file && file.include?(__FILE__)  # only from this file

  key = "#{file}:#{line}"
  allocation_map[key] += 1
end

ObjectSpace.trace_object_allocations_stop

puts "\nTop allocation sites:"
allocation_map
  .sort_by { |_, count| -count }
  .first(10)
  .each { |loc, count| puts "  #{loc}: #{count} objects" }

# Expected output shows the map/times/Hash.new lines
# as the hotspots for allocations
```

---

**S12. Show how to use `RubyVM::AbstractSyntaxTree` to inspect Ruby code.**

```ruby
# Available in Ruby 2.6+
ast = RubyVM::AbstractSyntaxTree.parse(<<~RUBY)
  def factorial(n)
    return 1 if n <= 1
    n * factorial(n - 1)
  end
RUBY

puts ast.inspect
# #<RubyVM::AbstractSyntaxTree::Node:SCOPE ...>

# Traverse the AST:
def find_method_calls(node, methods = [])
  return methods unless node.is_a?(RubyVM::AbstractSyntaxTree::Node)

  if node.type == :CALL || node.type == :VCALL || node.type == :FCALL
    methods << node.children[1]  # method name
  end

  node.children.each { |child| find_method_calls(child, methods) }
  methods
end

ast = RubyVM::AbstractSyntaxTree.parse(<<~RUBY)
  def process(data)
    data.upcase.strip.split(',').map { |s| s.to_i }
  end
RUBY

puts find_method_calls(ast).compact.uniq
# => [:upcase, :strip, :split, :map, :to_i]

# Parse from a file:
ast = RubyVM::AbstractSyntaxTree.parse_file("app/models/user.rb")

# Get source location:
puts ast.first_lineno   # first line of the code
puts ast.last_lineno    # last line of the code

# Practical uses:
# - Static analysis tools
# - Code generation
# - Finding deprecated API usage
# - Custom linting rules
```
