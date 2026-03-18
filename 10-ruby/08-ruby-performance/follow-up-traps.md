# Chapter 08: Ruby Performance — Follow-Up Traps

## Trap 1: `||=` Memoization Fails Silently for Falsy Return Values

**The trap:** The `||=` idiom re-evaluates the right-hand side whenever the cached value is `nil` or `false`, defeating the purpose of memoization.

```ruby
class PermissionService
  def admin?(user_id)
    @admin_cache ||= {}
    @admin_cache[user_id] ||= fetch_from_db(user_id)
    # BUG: if fetch_from_db returns false (not admin),
    # ||= will keep re-fetching on every call!
  end

  private

  def fetch_from_db(user_id)
    User.find(user_id).admin?  # Returns false for non-admins
  end
end

service = PermissionService.new
service.admin?(42)  # DB query: returns false
service.admin?(42)  # DB query again! false is falsy, ||= re-evaluates
service.admin?(42)  # DB query again!

# CORRECT: check for key presence, not truthiness
class FixedPermissionService
  def admin?(user_id)
    @admin_cache ||= {}
    unless @admin_cache.key?(user_id)
      @admin_cache[user_id] = fetch_from_db(user_id)
    end
    @admin_cache[user_id]
  end
end

service = FixedPermissionService.new
service.admin?(42)  # DB query: returns false — stored in cache
service.admin?(42)  # cache hit: returns false immediately — no DB query!

# Interview follow-up: "What about nil?"
# Same trap: nil is falsy, so @result ||= method_returning_nil re-evaluates every call
# Fix:
def result
  return @result if defined?(@result)
  @result = nil_returning_method
end
```

---

## Trap 2: `String#+` Creates New Objects; `String#<<` Mutates In Place

**The trap:** In a loop, using `+` for string building creates an exponentially growing number of intermediate objects, all of which pressure the GC.

```ruby
# SLOW: N new string objects created
def build_html_slow(items)
  html = "<ul>"
  items.each do |item|
    html = html + "<li>" + item + "</li>"  # 3 new strings per iteration!
  end
  html + "</ul>"
end

# Fast version:
def build_html_fast(items)
  html = +"<ul>"  # mutable string (+ prefix prevents freeze)
  items.each do |item|
    html << "<li>" << item << "</li>"  # mutates in place
  end
  html << "</ul>"
  html
end

# Or even better for static content:
def build_html_join(items)
  "<ul>#{items.map { |i| "<li>#{i}</li>" }.join}</ul>"
end

# Trap within the trap: frozen_string_literal: true + << raises FrozenError
# frozen_string_literal: true
html = "<ul>"     # frozen by the magic comment
html << "<li>"    # => FrozenError!

# Fix: unfreeze with +
html = +"<ul>"    # or: html = "<ul>".dup
html << "<li>"    # now works

# Memory allocation comparison for 10,000 items:
# String#+  → ~80,000 string objects (3 per iter + intermediate results)
# String#<< → 1 string object (mutated 30,000 times)
```

---

## Trap 3: `map` + `flatten` vs `flat_map` — Two Passes vs One

**The trap:** Using `map { }.flatten` instead of `flat_map` allocates an intermediate nested array and then flattens it, doing twice the work.

```ruby
# TWO PASSES:
users.map { |u| u.post_ids }.flatten
# Step 1: [[1,2], [3,4], [5,6]] — intermediate nested array (allocated)
# Step 2: [1,2,3,4,5,6] — new flat array (allocated)
# Total: 2 intermediate allocations

# ONE PASS:
users.flat_map { |u| u.post_ids }
# Directly produces [1,2,3,4,5,6] — no intermediate nested array

# BUT: they are NOT equivalent for multiple levels of nesting!
nested = [1, [2, [3, 4]], 5]

nested.map { |x| x }.flatten      # => [1, 2, 3, 4, 5]  (flattens ALL levels)
nested.flat_map { |x| [x] }       # => [1, [2, [3, 4]], 5]  (only one level!)

# flat_map is equivalent to map{}.flatten(1) — only one level deep
[[1,2],[3,[4,5]]].flat_map { |a| a }       # => [1, 2, 3, [4, 5]]
[[1,2],[3,[4,5]]].map { |a| a }.flatten    # => [1, 2, 3, 4, 5]

# Use flat_map when:
# - You're mapping then flattening one level (most common case)
# - Performance matters

# Use map + flatten when:
# - You need to flatten multiple levels (flat_map can't do this)
# - flatten(n) with n > 1
```

---

## Trap 4: YJIT Only Helps CPU-Bound Code — I/O-Bound Code Sees No Benefit

**The trap:** Enabling YJIT and expecting all your benchmarks to improve. Web apps that spend most time waiting for the database see minimal speedup.

```ruby
# YJIT helps (CPU-bound):
def cpu_intensive
  result = 0
  1_000_000.times { |i| result += i * i * Math.sin(i) }
  result
end
# YJIT: ~2-3x faster (compiles the hot loop to native code)

# YJIT doesn't help (I/O-bound):
def io_bound
  10.times { User.where(id: 1..100).to_a }  # blocked on DB
end
# YJIT: 0% improvement (blocked on network/disk, not CPU)

# Measuring YJIT impact correctly:
puts "YJIT enabled: #{RubyVM::YJIT.enabled?}"

# Check compilation coverage:
stats = RubyVM::YJIT.runtime_stats
puts "YJIT calls:    #{stats[:yjit_insns_count]}"
puts "Ratio:         #{(stats[:ratio_in_yjit] * 100).round(1)}%"
# If ratio is < 10%, YJIT has little opportunity to help

# Realistic YJIT gains in production:
# Rails app (I/O heavy): 3-10% throughput improvement
# Math-heavy service:    30-100% improvement
# Pure computation:      up to 3x improvement
# String parsing:        10-25% improvement

# Trap: benchmarking with a cold JIT
# Always warm up YJIT before measuring:
require 'benchmark/ips'
Benchmark.ips do |x|
  x.config(warmup: 5)  # 5 seconds for YJIT to warm up
  x.report("cpu_task") { cpu_intensive }
end
```

---

## Trap 5: Symbol GC — Symbols Before Ruby 2.2 Were Never Garbage Collected

**The trap:** Using `String#to_sym` or `"string".to_sym` to convert user-provided strings to symbols in pre-2.2 Ruby causes unbounded symbol table growth and eventual memory exhaustion.

```ruby
# Pre-Ruby 2.2 — permanent memory leak:
params.each_key do |key|
  process(key.to_sym)  # each unique key creates a permanent symbol!
end
# After 1M requests with 10 unique param names: fine
# After 1M requests with 1M unique param names: 1M symbols, all permanent

# Ruby 2.2+ — dynamic symbols can be GC'd:
:foo.frozen?  # => true (static/interned symbols are permanent)
"foo".to_sym.frozen?  # => true, but CAN be GC'd if no references

# Safe patterns for any Ruby version:
# 1. Don't convert user input to symbols (keep as strings)
params.each_key do |key|
  process(key)  # key is already a string in most frameworks
end

# 2. Validate against known symbols
ALLOWED_KEYS = %i[name email age].freeze
def safe_process(key)
  sym = ALLOWED_KEYS.find { |k| k.to_s == key }
  raise ArgumentError, "Unknown key: #{key}" unless sym
  process(sym)
end

# 3. Benchmarking the impact:
require 'memory_profiler'
MemoryProfiler.report do
  1000.times { |i| "key_#{i}".to_sym }  # creates 1000 dynamic symbols
end.pretty_print
```

---

## Trap 6: Premature Optimization — Benchmarking Before Profiling

**The trap:** Optimizing random code that isn't the bottleneck. The classic trap is spending hours optimizing a method that takes 5ms when the real bottleneck is a 500ms database query.

```ruby
# WRONG APPROACH: guess and optimize
def process_users(users)
  # Someone spent 2 hours making this 10% faster
  users.flat_map { |u| u.permissions.map(&:name) }  # now 9ms instead of 10ms
end

# The actual bottleneck:
def load_users_with_permissions
  # This runs an N+1 query: 1 query for users + 1 per user for permissions
  User.all.map do |u|
    { user: u, permissions: u.permissions }  # 1 + N queries!
  end
end
# Fixing the N+1 saves 490ms, not 1ms

# CORRECT APPROACH:
# 1. Measure (benchmark or profiler)
StackProf.run(mode: :wall, out: 'tmp/stackprof.dump') do
  load_and_process_users
end

# 2. Find the actual bottleneck from the profiler output
# stackprof tmp/stackprof.dump --text
# Most time: 87% in ActiveRecord::Relation#load_target (DB queries)

# 3. THEN optimize the bottleneck
users = User.includes(:permissions).all  # eager load → 2 queries, done
```

---

## Trap 7: GC.start Is a Hint, Not a Command

**The trap:** Calling `GC.start` at specific points thinking it will immediately free memory. It requests a GC run but Ruby may defer or decline it.

```ruby
# Code that assumes GC.start frees memory:
big_data = Array.new(1_000_000) { "x" * 100 }
big_data = nil   # dereference
GC.start         # REQUEST GC (may or may not run immediately)

# ObjectSpace still shows the objects:
puts ObjectSpace.count_objects[:T_STRING]  # might still show them!

# Ruby's GC runs opportunistically:
# - When heap is full
# - When allocation threshold is exceeded
# - When GC.start is called (but it can be deferred)

# For testing memory cleanup:
big_data = nil
3.times { GC.start }  # multiple calls increases likelihood
GC.compact rescue nil  # Ruby 2.7+ heap compaction

# For Puma workers or long-running processes, a better approach is
# to use fork + exit for guaranteed memory release:
pid = fork do
  process_large_dataset(data)
  exit 0
end
Process.waitpid(pid)
# ALL memory from the child process is released on exit — guaranteed
```

---

## Trap 8: `ObjectSpace.each_object` Prevents GC During Iteration

**The trap:** Iterating `ObjectSpace.each_object` holds a reference to every live object, preventing GC from collecting anything during that iteration. On large heaps, this can temporarily double memory usage.

```ruby
# TRAP: calling this on a production app with millions of objects
count = 0
ObjectSpace.each_object(String) { |s| count += 1 }
# During this loop: GC is inhibited, memory can spike

# Safer approach: count by class quickly with count_objects
ObjectSpace.count_objects.select { |k, v| k.to_s.start_with?('T_') }

# Or use memsize_of_all for memory stats without iteration:
require 'objspace'
ObjectSpace.memsize_of_all(String)  # total bytes of all live strings

# If you MUST iterate, minimize what you do inside the block:
suspect_allocations = []
ObjectSpace.each_object(SuspiciousClass) do |obj|
  # Just collect object_ids, don't hold direct references
  suspect_allocations << obj.object_id
end
# Look up by object_id later if needed (though object may be GC'd by then)
```

---

## Trap 9: `lazy` Enumerator Has Overhead — Not Always Faster

**The trap:** Using `lazy` on small collections or simple transformations actually makes them slower due to Enumerator overhead.

```ruby
require 'benchmark/ips'

small = (1..100)
large = (1..1_000_000)

Benchmark.ips do |x|
  # Small collection: lazy is SLOWER
  x.report("small eager") { small.select { |n| n.odd? }.first(10) }
  x.report("small lazy")  { small.lazy.select { |n| n.odd? }.first(10) }

  # Large collection (only need first 10): lazy is FASTER
  x.report("large eager") { large.select { |n| n.odd? }.first(10) }
  x.report("large lazy")  { large.lazy.select { |n| n.odd? }.first(10) }
  x.compare!
end

# Results:
# small eager: 3.2M i/s
# small lazy:  1.1M i/s  ← slower! Enumerator overhead > savings
# large eager: 52K i/s   ← processes ALL 1M elements
# large lazy:  4.9M i/s  ← stops after finding 10 matches

# Rule: Use lazy when:
# 1. Collection is large (> ~1000 elements)
# 2. You don't need all results (using first/take/each with break)
# 3. Generating from an infinite source
# Don't use lazy for small collections or when you need all results
```

---

## Trap 10: `benchmark-ips` Results Vary Under GC Pressure

**The trap:** If the code under benchmark triggers GC during measurement, results are noisy and misleading. A benchmark that allocates less may show higher variance rather than genuine speed.

```ruby
require 'benchmark/ips'

# Noisy benchmark: allocates heavily inside measured code
Benchmark.ips do |x|
  x.report("allocating") do
    Array.new(10_000) { rand }  # triggers GC during measurement!
  end

  x.report("minimal alloc") do
    sum = 0
    10_000.times { sum += rand }  # no GC pressure
    sum
  end
  x.compare!
end
# Results may show "allocating" as seemingly competitive
# because GC is not evenly distributed across samples

# Better benchmark with GC disabled (for comparison only, not production):
Benchmark.ips do |x|
  x.config(time: 5, warmup: 2)

  x.report("with alloc") do
    GC.disable
    result = Array.new(1000) { rand }
    GC.enable
    result
  end
end

# Even better: use memory_profiler separately to understand allocation,
# and benchmark the CPU-only path with minimal allocations
# The cleanest benchmarks measure ONLY what you intend to measure
```
