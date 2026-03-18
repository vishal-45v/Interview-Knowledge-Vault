# Chapter 08: Ruby Performance — Theory Questions

## Benchmarking

**Q1. How do you use the `Benchmark` module to measure Ruby code performance?**

```ruby
require 'benchmark'

# Basic timing
Benchmark.measure do
  1_000_000.times { "hello".upcase }
end
# => #<Benchmark::Tms: real=0.15, utime=0.14, stime=0.00, total=0.14>

# Comparing multiple approaches
Benchmark.bm(20) do |x|  # 20 = label width
  x.report("String#+")   { 100_000.times { s = "hello" + " world" } }
  x.report("String#<<")  { 100_000.times { s = "hello"; s << " world" } }
  x.report("interpolate"){ 100_000.times { s = "hello #{"world"}" } }
end

# Benchmark.bmbm removes the JIT warmup bias by running twice:
Benchmark.bmbm(20) do |x|
  x.report("Array#each")   { 100_000.times { [1,2,3].each { |i| i * 2 } } }
  x.report("Array#map")    { 100_000.times { [1,2,3].map { |i| i * 2 } } }
end

# Output columns:
# user       system    total     real
# 0.140000   0.000000  0.140000  (0.150000)
# user = CPU time in user space
# real = wall clock time (includes I/O waits, sleep)
```

---

**Q2. What is `benchmark-ips` and why is it better than the standard `Benchmark` module for micro-benchmarks?**

`benchmark-ips` (iterations per second) automatically calibrates the sample count, runs a warmup phase, and reports iterations/second instead of time. This eliminates the need to manually choose iteration counts.

```ruby
require 'benchmark/ips'

Benchmark.ips do |x|
  x.config(time: 5, warmup: 2)  # 2s warmup, 5s measurement

  x.report("symbol keys")   { { name: "Alice", age: 30 } }
  x.report("string keys")   { { "name" => "Alice", "age" => 30 } }
  x.report("frozen string") { { "name".freeze => "Alice" } }

  x.compare!  # shows relative performance
end

# Output:
# Calculating -------------------------------------
#      symbol keys     1.234M (± 2.5%) i/s
#      string keys   987.654k (± 3.1%) i/s
#   frozen string    1.198M (± 2.0%) i/s
#
# Comparison:
#      symbol keys: 1234567.0 i/s
#   frozen string:  1198765.0 i/s - 1.03x slower
#      string keys:  987654.0 i/s - 1.25x slower
```

---

**Q3. What is object allocation and why does it cause GC pressure?**

Every Ruby object consumes memory on the heap. The garbage collector must periodically scan all live objects to find and reclaim dead ones. More object allocation means more GC runs, more pauses, and slower overall throughput.

```ruby
require 'objspace'

# Measuring allocations in a block
ObjectSpace.trace_object_allocations_start

def expensive_allocation
  100.times.map { |i| "Item #{i}" }  # 100 string objects + 1 array = 101 allocs
end

before = ObjectSpace.count_objects_size
expensive_allocation
after = ObjectSpace.count_objects_size

# Better: use allocation_count from memory_profiler
require 'memory_profiler'

report = MemoryProfiler.report do
  100.times.map { |i| "Item #{i}" }
end

report.pretty_print
# Total allocated: 400 bytes (101 objects)
# Total retained:  0 bytes (0 objects)

# Low-allocation version:
result = Array.new(100) { |i| "Item #{i}".freeze }
# Still 100 strings but frozen (may be reused) + 1 array
```

---

**Q4. What does `# frozen_string_literal: true` do and what performance gains does it provide?**

When the magic comment `# frozen_string_literal: true` appears at the top of a file, every string literal in that file is automatically frozen and interned — the same string value reuses the same object instead of allocating new ones.

```ruby
# Without frozen_string_literal:
str1 = "hello"
str2 = "hello"
str1.equal?(str2)  # => false (two different objects)
str1.object_id == str2.object_id  # => false

# frozen_string_literal: true
# frozen_string_literal: true
str1 = "hello"
str2 = "hello"
str1.equal?(str2)  # => true (same object — interned!)
str1.frozen?       # => true

# Performance: in string-heavy code (parsers, template engines, web apps)
# this reduces allocation by 30-60% for string literals

# Caveat: cannot mutate frozen strings
str = "hello"
str << " world"  # => FrozenError: can't modify frozen String
str += " world"  # fine — creates new string (assignment doesn't mutate)

# Enable for entire project in Ruby 3+:
# Add to .rubocop.yml: Style/FrozenStringLiteralComment: EnforcedStyle: always
```

---

**Q5. What are the memory trade-offs between Symbol and String for hash keys?**

```ruby
# Symbols: interned, single instance per unique name, not GC'd before Ruby 2.2
:user_name.object_id == :user_name.object_id  # => true (always same object)
:user_name.frozen?  # => true (symbols are always frozen)

# Strings: new object per literal (without frozen_string_literal)
"user_name".object_id == "user_name".object_id  # => false (two objects)

# Hash key performance:
hash = {}
1_000_000.times { hash[:name] = 1 }       # fast: same symbol object each time
1_000_000.times { hash["name"] = 1 }      # slower: new "name" string each time (pre-Ruby 3)

# In Ruby 2.2+: dynamic symbols are GC'd
# In Ruby 3.0+: string literals with frozen_string_literal perform similarly to symbols for keys

# Memory impact:
require 'memory_profiler'
MemoryProfiler.report { 10_000.times { { name: "Alice", age: 30 } } }.pretty_print
# vs
MemoryProfiler.report { 10_000.times { { "name" => "Alice", "age" => 30 } } }.pretty_print
# Symbol version allocates fewer objects (20,001 vs 40,001 objects)
```

---

**Q6. Explain Ruby's generational garbage collector. What are minor and major GC runs?**

Ruby (2.1+) uses a generational GC based on the observation that most objects die young (short-lived temporaries created in method calls).

- **Eden heap**: newly allocated objects live here (young generation)
- **Survivor spaces**: objects that survive a minor GC are promoted
- **Old heap**: long-lived objects (models, classes, caches) promoted after multiple survivors

```ruby
# Minor GC: only scans the young generation (fast)
# Major GC: scans everything including old generation (slow)

GC.stat.tap do |s|
  puts "Minor GC count: #{s[:minor_gc_count]}"
  puts "Major GC count: #{s[:major_gc_count]}"
  puts "Heap pages:     #{s[:heap_allocated_pages]}"
  puts "Live objects:   #{s[:heap_live_slots]}"
  puts "Free slots:     #{s[:heap_free_slots]}"
end

# Tune GC via environment variables:
# RUBY_GC_HEAP_INIT_SLOTS=600000     — initial heap size (prevent early GC)
# RUBY_GC_HEAP_GROWTH_FACTOR=1.25    — how fast heap grows (1.25 = 25%)
# RUBY_GC_HEAP_OLDOBJECT_LIMIT_FACTOR=2.0  — when to run major GC
# RUBY_GC_MALLOC_LIMIT=67108864      — malloc limit before GC (64MB)

# Force GC for testing:
GC.start         # request a GC run
GC.compact       # compact heap (Ruby 2.7+)
```

---

**Q7. What is incremental GC and how does it reduce pause times?**

Before incremental GC (pre-Ruby 2.2), Ruby would stop the entire process (stop-the-world) during a GC cycle. For large heaps with millions of objects, this caused pauses of 100ms+.

Incremental GC uses tri-color marking: objects are WHITE (not yet marked), GREY (marked but children not yet visited), or BLACK (fully marked). The GC processes objects incrementally, yielding back to the application between steps.

```ruby
# Demonstrate GC behavior in different modes

# Ruby 2.1: Mark-and-sweep (stop-the-world)
# Ruby 2.2: Generational + Incremental GC
# Ruby 2.7+: Compaction (GC.compact)
# Ruby 3.x: Improved YJIT + GC cooperation

# Check current GC configuration
puts GC::OPTS  # => ["GC_HEAP_INIT_SLOTS", "GC_HEAP_GROWTH_FACTOR", ...]

# Measure GC impact on your code:
GC::Profiler.enable
your_method_here
GC::Profiler.print
GC::Profiler.disable

# Output:
# GC 5 invokes.
# Index  Invoke Time(sec)  Use Size(byte)   Total Size(byte)    GC Time(ms)
#     1              0.01      1234567          2345678            1.23
```

---

**Q8. How do you use the `memory_profiler` gem?**

```ruby
require 'memory_profiler'

report = MemoryProfiler.report do
  # Code to profile
  users = 1000.times.map { |i| User.new(name: "User #{i}", id: i) }
  users.select { |u| u.id.even? }.map(&:name)
end

report.pretty_print

# OUTPUT SECTIONS:
# 1. Total allocated (bytes + objects) — memory created during the block
# 2. Total retained (bytes + objects) — memory still alive after the block (leaks!)
# 3. Allocated memory by gem — which gems allocate most
# 4. Allocated memory by file/line — where in your code allocations happen
# 5. Allocated objects by class — String 5678, Array 234, User 1000, etc.
# 6. Retained objects — what was NOT GC'd

# Key metrics to watch:
# - High allocation + low retention = normal (objects are GC'd)
# - High allocation + high retention = LEAK (objects aren't freed)
# - High allocation from unexpected classes = optimization opportunity

# Reducing allocations:
# BEFORE: 2000 String allocations
users.map { |u| "User: #{u.name}" }

# AFTER: 0 extra String allocations (mutate a single buffer)
buf = +"User: "  # unfrozen string
users.each do |u|
  buf.clear
  buf << "User: " << u.name
  process(buf)
end
```

---

**Q9. What is `ruby-prof` and how does it identify performance bottlenecks?**

`ruby-prof` is a sampling/instrumentation profiler that measures time spent in each method call, showing a call graph of where CPU time is spent.

```ruby
require 'ruby-prof'

RubyProf.start
# ... run your code ...
result = RubyProf.stop

# Print different formats:
printer = RubyProf::FlatPrinter.new(result)
printer.print(STDOUT, min_percent: 1)  # only show methods taking >1%

# Output (flat format):
# %self    total    self    wait   child    calls  name
# 45.23    2.341   1.059   0.000   1.282   10000  String#upcase
# 23.11    1.200   0.541   0.000   0.659   10000  Array#sort

# Call graph (shows caller/callee relationships):
printer = RubyProf::GraphPrinter.new(result)
printer.print(STDOUT)

# For web apps: use rack-mini-profiler for in-browser profiling
# For production sampling: use stackprof (lower overhead)

require 'stackprof'
StackProf.run(mode: :wall, out: 'stackprof.dump') do
  # your code
end
# Analyze: stackprof stackprof.dump --text --limit 20
```

---

**Q10. What is `stackprof` and how does it differ from `ruby-prof`?**

```ruby
require 'stackprof'

# stackprof is a sampling profiler — takes a stack snapshot every N microseconds
# ruby-prof is an instrumentation profiler — instruments every method entry/exit

# stackprof: very low overhead (~1-5%), safe for production sampling
# ruby-prof: higher overhead (~10-30%), not safe for production

# stackprof usage:
StackProf.run(mode: :cpu,         # :cpu, :wall, or :object
              out: '/tmp/profile.dump',
              interval: 1000,     # microseconds between samples
              raw: true) do       # raw data for flamegraph
  100_000.times { process_record }
end

# Analyze results:
# CLI: stackprof /tmp/profile.dump --text
# Flamegraph: stackprof /tmp/profile.dump --flamegraph > /tmp/flame.html

# Modes:
# :cpu    — CPU time only (excludes I/O wait)
# :wall   — wall clock time (includes I/O, sleep)
# :object — object allocations (like tracing allocs)

# In Rails: add to initializer for periodic sampling
require 'stackprof'
StackProf::Middleware.new(app,
  enabled: ENV['PROFILE'],
  mode: :cpu,
  save_every: 5  # save every 5 requests
)
```

---

**Q11. Explain the `||=` memoization pattern and its failure mode with falsy values.**

```ruby
class Calculator
  def result
    @result ||= expensive_computation
  end

  private

  def expensive_computation
    sleep(0.1)  # simulated expensive work
    42
  end
end

# Works correctly for truthy values:
calc = Calculator.new
calc.result  # computes (100ms)
calc.result  # cached: 42 (instant)

# FAILURE: falsy values defeat ||=
class FlagService
  def enabled?(feature)
    @cache ||= {}
    @cache[feature] ||= fetch_flag(feature)  # BUG: if flag is false, re-fetches every time!
  end

  def fetch_flag(feature)
    # simulate DB/API lookup
    false  # or nil
  end
end

service = FlagService.new
service.enabled?(:beta)  # fetches, returns false
service.enabled?(:beta)  # fetches AGAIN because false is falsy → re-evaluates ||=

# FIX: use key? check or explicit nil check
class CorrectFlagService
  def enabled?(feature)
    @cache ||= {}
    unless @cache.key?(feature)
      @cache[feature] = fetch_flag(feature)
    end
    @cache[feature]
  end
end

# FIX 2: defined? for instance variable memoization
def result
  return @result if defined?(@result)
  @result = expensive_computation  # safe for nil/false return values
end
```

---

**Q12. What is lazy evaluation and when does it improve performance for large collections?**

```ruby
# Eager evaluation — processes all elements even if only a few are needed
numbers = (1..Float::INFINITY)

# INFINITE LOOP — tries to process all numbers
numbers.select { |n| n.odd? }.first(10)

# Lazy evaluation — processes only what's needed
numbers.lazy.select { |n| n.odd? }.first(10)
# => [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]

# Performance comparison for large datasets:
require 'benchmark/ips'

data = (1..1_000_000)

Benchmark.ips do |x|
  x.report("eager") do
    data.select { |n| n % 3 == 0 }.map { |n| n * 2 }.first(10)
    # Generates 333,333 numbers, maps 333,333, takes first 10
  end

  x.report("lazy") do
    data.lazy.select { |n| n % 3 == 0 }.map { |n| n * 2 }.first(10)
    # Processes only 30 numbers total (finds 10 matches and stops)
  end
  x.compare!
end
# lazy is 50-100x faster for this case

# When lazy evaluation helps:
# - Large datasets where you only need the first N results
# - Chained transformations that can short-circuit
# - Potentially infinite sequences

# When lazy doesn't help:
# - You need ALL results (lazy overhead negates benefit)
# - Small collections (overhead > benefit)
# - Operations that can't short-circuit
```

---

**Q13. What is YJIT (Ruby 3.1+) and what kind of code does it accelerate?**

YJIT (Yet Another Just-In-Time Compiler) is a method-based JIT compiler built into MRI Ruby starting in version 3.1. It compiles Ruby bytecode to native machine code at runtime for frequently called methods.

```ruby
# Enable YJIT (Ruby 3.1+):
# At startup: ruby --yjit script.rb
# In code:    RubyVM::YJIT.enable  (Ruby 3.3+)

# Check if YJIT is enabled:
puts RubyVM::YJIT.enabled?  # => true/false

# YJIT statistics:
puts RubyVM::YJIT.runtime_stats.inspect

# YJIT excels at:
# - CPU-bound numeric computation
# - Hot loops with predictable types
# - Method calls with stable receiver types

# Example where YJIT helps (3x faster):
def sum_squares(n)
  total = 0
  n.times { |i| total += i * i }
  total
end

# YJIT has MINIMAL impact on:
# - I/O-bound code (waiting for network/disk)
# - Method-missing heavy metaprogramming
# - C-extension heavy code (FFI calls)
# - Cold code (called only once)

# MJIT (Ruby 2.6-3.2): original JIT, now deprecated in favor of YJIT
# YJIT vs MJIT: YJIT is lazy (compiles hot paths), MJIT was eager
```

---

**Q14. What are native C extensions and how do they affect Ruby performance?**

```ruby
# C extensions are shared libraries (.so on Linux, .bundle on macOS)
# that implement Ruby methods in C. They bypass Ruby's interpreter overhead.

# Common C extension gems:
# nokogiri    — XML/HTML parsing (libxml2)
# oj          — JSON parsing (10-20x faster than json gem)
# msgpack     — binary serialization
# bcrypt-ruby — password hashing
# pg          — PostgreSQL client

# Using FFI (Foreign Function Interface) without C extension:
require 'ffi'

module FastHash
  extend FFI::Library
  ffi_lib 'c'
  attach_function :strlen, [:string], :size_t
end

FastHash.strlen("hello")  # calls C's strlen directly

# When to consider C extensions:
# 1. Pure Ruby algorithm too slow even after optimization
# 2. Wrapping existing C/C++ libraries
# 3. Tight numeric loops (use fiddle, ffi, or write extension in C)

# Alternative: use existing fast gems
require 'oj'
Oj.dump({ key: "value" })   # C-level JSON — much faster than JSON.generate

# Measure C extension benefit:
require 'benchmark/ips'
Benchmark.ips do |x|
  x.report("stdlib json") { JSON.generate({ name: "Alice", data: [1,2,3] }) }
  x.report("oj json")     { Oj.dump({ name: "Alice", data: [1,2,3] }) }
  x.compare!
end
```

---

**Q15. How do N+1 queries manifest in plain Ruby (without Rails)?**

```ruby
# N+1 in plain Ruby — repeated individual lookups in a loop
class UserRepository
  def find(id)
    DB.query("SELECT * FROM users WHERE id = ?", id).first
  end

  def find_many(ids)
    DB.query("SELECT * FROM users WHERE id IN (?)", ids)
  end
end

class PostRepository
  def find_by_user(user_id)
    DB.query("SELECT * FROM posts WHERE user_id = ?", user_id)
  end

  def find_by_users(user_ids)
    DB.query("SELECT * FROM posts WHERE user_id IN (?)", user_ids)
      .group_by { |p| p["user_id"] }
  end
end

posts = PostRepository.new.all  # 1 query for N posts

# N+1 pattern:
posts.each do |post|
  user = UserRepository.new.find(post["user_id"])  # N individual queries!
  puts "#{user["name"]}: #{post["title"]}"
end

# Fix: batch loading
user_ids = posts.map { |p| p["user_id"] }.uniq
users_by_id = UserRepository.new.find_many(user_ids)
                 .each_with_object({}) { |u, h| h[u["id"]] = u }

posts.each do |post|
  user = users_by_id[post["user_id"]]  # hash lookup — no query
  puts "#{user["name"]}: #{post["title"]}"
end
# Total: 2 queries instead of N+1
```

---

**Q16. What is `String#<<` vs `String#+` and why does it matter for performance?**

```ruby
# String#+ creates a NEW string object each time
result = ""
1000.times do |i|
  result = result + "item #{i}, "  # Creates 1000 intermediate string objects!
end

# String#<< mutates in place — no new object created
result = ""
1000.times do |i|
  result << "item #{i}, "  # Only 1 string object, mutated 1000 times
end

# Performance difference:
require 'benchmark/ips'
Benchmark.ips do |x|
  x.report("String#+") do
    s = ""
    100.times { |i| s = s + "x" }
  end

  x.report("String#<<") do
    s = ""
    100.times { |i| s << "x" }
  end
  x.compare!
end
# String#<< is typically 2-5x faster for repeated concatenation

# String building patterns:
# SLOW: + in a loop
parts = ["hello", " ", "world", "!"]
result = parts.reduce("") { |acc, s| acc + s }  # 4 intermediate strings

# FAST: join
result = parts.join  # one pass, one output string

# FAST: << in a buffer
buf = +""  # unary + creates unfrozen string
parts.each { |s| buf << s }
```

---

**Q17. What is `map` + `flatten` vs `flat_map` performance difference?**

```ruby
data = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

# map + flatten: two passes, two intermediate arrays
result = data.map { |arr| arr.map { |x| x * 2 } }.flatten
# Pass 1: [[2,4,6], [8,10,12], [14,16,18]]  ← intermediate array
# Pass 2: flatten creates another new array

# flat_map: single pass, one output array
result = data.flat_map { |arr| arr.map { |x| x * 2 } }
# Direct output without intermediate nesting

require 'benchmark/ips'
large_data = Array.new(10_000) { Array.new(100) { rand(100) } }

Benchmark.ips do |x|
  x.report("map+flatten") do
    large_data.map { |arr| arr.map { |x| x * 2 } }.flatten
  end

  x.report("flat_map") do
    large_data.flat_map { |arr| arr.map { |x| x * 2 } }
  end
  x.compare!
end
# flat_map is ~2x faster and allocates less memory

# Important: flat_map only flattens ONE level deep
[[1, [2]], [3, [4]]].flat_map { |a| a }   # => [1, [2], 3, [4]]  (not fully flat)
[[1, [2]], [3, [4]]].map { |a| a }.flatten # => [1, 2, 3, 4]      (fully flat)
```

---

**Q18. How does memoization interact with thread safety?**

```ruby
# Single-threaded memoization (common pattern):
class Config
  def settings
    @settings ||= load_from_file
  end
end

# Thread safety problem in multi-threaded environments:
class UnsafeCache
  def fetch(key)
    @cache ||= {}               # race condition: two threads can both reach here
    @cache[key] ||= compute(key)  # two threads can both compute and overwrite
  end
end

# Thread-safe memoization with Mutex:
class SafeCache
  def initialize
    @mutex = Mutex.new
    @cache = {}
  end

  def fetch(key)
    # Double-checked locking pattern
    return @cache[key] if @cache.key?(key)

    @mutex.synchronize do
      return @cache[key] if @cache.key?(key)  # recheck inside lock
      @cache[key] = compute(key)
    end
  end

  private

  def compute(key)
    # expensive computation
  end
end

# Ruby's Concurrent::Map from concurrent-ruby gem (lock-free):
require 'concurrent'

class ConcurrentCache
  def initialize
    @cache = Concurrent::Map.new
  end

  def fetch(key)
    @cache.compute_if_absent(key) { compute(key) }
  end
end
```

---

**Q19. What are the key performance improvements in Ruby 3.0 vs 2.7?**

```ruby
# Ruby 3.0 target: "3x faster than Ruby 2.0"

# 1. Improved Fiber scheduler for async I/O
# 2. Ractors: parallel execution (experimental)
# 3. Type checking with RBS + Steep/Sorbet
# 4. Keyword argument changes (separation from positional)

# Ruby 3.1 additions:
# - YJIT (production-ready)
# - error_highlight gem (points to exact token in error)
# - Refinements in module#include context

# Ruby 3.2 additions:
# - Object shapes (massive performance improvement for ivar access)
# - Data class (immutable Struct alternative)
# - YJIT improvements (50-60% speedup over 3.1 YJIT)
# - Production-ready Ractors

# Ruby 3.3 additions:
# - YJIT: 15-25% faster than 3.2 YJIT
# - Prism parser (faster, better error messages)
# - RubyVM::YJIT.enable at runtime

# Benchmarking version differences:
# Use ruby-benchmark-suite or optcarrot to compare versions
# Typical real-world Rails app speedup: Ruby 2.7 → 3.2: ~20-30%
```

---

**Q20. How do you profile memory usage in a Rails application in production?**

```ruby
# 1. Derailed Benchmarks gem (production-like profiling)
# bundle exec derailed exec perf:mem
# Measures memory used when booting the app and per request

# 2. memory_profiler gem + Rack middleware
require 'memory_profiler'

class MemoryProfileMiddleware
  def initialize(app)
    @app = app
  end

  def call(env)
    return @app.call(env) unless env['HTTP_X_PROFILE_MEMORY']

    report = MemoryProfiler.report { @app.call(env) }
    [200, {}, [report.pretty_print_str]]
  end
end

# 3. Scout APM / DataDog APM for production monitoring
# Track: heap_live_slots, heap_free_slots, GC count

# 4. Custom metrics with StatsD
module GCMetrics
  def self.report
    stats = GC.stat
    StatsD.gauge('ruby.gc.heap_live_slots',  stats[:heap_live_slots])
    StatsD.gauge('ruby.gc.heap_free_slots',  stats[:heap_free_slots])
    StatsD.gauge('ruby.gc.minor_gc_count',   stats[:minor_gc_count])
    StatsD.gauge('ruby.gc.major_gc_count',   stats[:major_gc_count])
    StatsD.gauge('ruby.gc.total_allocated',  stats[:total_allocated_objects])
  end
end

# Call GCMetrics.report after each request
```
