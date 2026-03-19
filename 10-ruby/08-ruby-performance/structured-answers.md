# Chapter 08: Ruby Performance — Structured Answers

## Answer 1: Complete Performance Investigation Workflow

**Question:** "How do you systematically investigate and fix a slow Ruby application?"

**Answer:**

```ruby
# PHASE 1: Establish a baseline measurement
require 'benchmark'

def measure_baseline
  Benchmark.bm do |x|
    x.report("endpoint") { simulate_request }
  end
end

# PHASE 2: Check the obvious first (N+1, missing indexes)
# For Rails:
# 1. bullet gem detects N+1 in development
# 2. rack-mini-profiler shows per-request SQL breakdown
# 3. EXPLAIN ANALYZE on slow queries

# PHASE 3: CPU profiling with stackprof
require 'stackprof'

StackProf.run(mode: :cpu, out: 'tmp/profile.dump', interval: 1000) do
  100.times { simulate_request }
end

# Analyze results:
# $ stackprof tmp/profile.dump --text --limit 20

# PHASE 4: Memory profiling
require 'memory_profiler'

report = MemoryProfiler.report do
  100.times { simulate_request }
end

report.pretty_print(to_file: 'tmp/memory_report.txt')

# PHASE 5: Identify allocation hotspots from the memory report
# Look for:
# 1. High "Total allocated" vs "Total retained" ratio (normal)
# 2. Unexpected classes allocating many objects (bug/optimization opportunity)
# 3. Retained objects growing each call (memory leak)

# PHASE 6: Benchmark specific optimizations
require 'benchmark/ips'

Benchmark.ips do |x|
  x.config(warmup: 3, time: 5)
  x.report("before") { original_method(data) }
  x.report("after")  { optimized_method(data) }
  x.compare!
end

# PHASE 7: Verify in production-like conditions
# - Test with production-size datasets (not toy data)
# - Include I/O operations that were skipped in unit benchmarks
# - Monitor GC metrics before and after deploy
puts GC.stat.slice(:minor_gc_count, :major_gc_count, :heap_live_slots)
```

---

## Answer 2: Reducing GC Pressure with Object Reuse

**Question:** "Show me concrete techniques to reduce garbage collector pressure."

**Answer:**

```ruby
# TECHNIQUE 1: Avoid allocating in hot paths — freeze/reuse constants
# BAD: allocates a new array each call
def valid_roles
  [:admin, :editor, :viewer]  # new array every call
end

# GOOD: frozen constant
VALID_ROLES = [:admin, :editor, :viewer].freeze

# TECHNIQUE 2: Use frozen_string_literal pragma
# frozen_string_literal: true

# All string literals in this file are now interned (shared)
STATUS_ACTIVE = "active"   # same object every reference
STATUS_DRAFT  = "draft"

# TECHNIQUE 3: Reuse buffer objects in tight loops
class LineProcessor
  def initialize
    @buffer = +""  # single mutable string, reused
  end

  def process_line(input)
    @buffer.clear
    @buffer << "PREFIX: " << input.strip << " :SUFFIX"
    yield @buffer  # caller should not retain the buffer!
  end
end

processor = LineProcessor.new
File.foreach("large_file.txt") do |line|
  processor.process_line(line) { |processed| write_output(processed) }
end
# Only 1 string object for the entire file processing

# TECHNIQUE 4: Preallocate output arrays
def transform_batch(items)
  result = Array.new(items.size)  # preallocate — no resizing
  items.each_with_index do |item, i|
    result[i] = transform(item)
  end
  result
end

# vs: result = [] and result << item  (may trigger reallocation as array grows)

# TECHNIQUE 5: Struct instead of Hash for repeated small objects
# Hash: ~440 bytes per object
# Struct: ~80 bytes per object (less overhead, faster field access)

# BAD (many small hashes):
users = data.map { |d| { id: d[0], name: d[1], email: d[2] } }

# GOOD (Struct):
UserRecord = Struct.new(:id, :name, :email)
users = data.map { |d| UserRecord.new(*d) }

# TECHNIQUE 6: Filter before mapping to reduce work
# map + select (maps everything, then filters)
result = items.map { |i| expensive_transform(i) }.select { |i| i.valid? }

# select + map (filters first, then maps only valid ones)
result = items.select { |i| cheap_filter_check(i) }.map { |i| expensive_transform(i) }

# filter_map (Ruby 2.7+) — both in one pass
result = items.filter_map { |i| expensive_transform(i) if cheap_filter_check(i) }
```

---

## Answer 3: Benchmarking Correctly — Warmup, Variance, and Significance

**Question:** "How do you write reliable benchmarks? What are the common mistakes?"

**Answer:**

```ruby
require 'benchmark/ips'
require 'benchmark'

# MISTAKE 1: Not warming up (JIT, caches not initialized)
# BAD:
time = Benchmark.measure { my_method }
# First call may include: require overhead, method cache misses, YJIT compilation

# GOOD: warm up first
10.times { my_method }  # warmup
Benchmark.ips do |x|
  x.config(warmup: 3, time: 5)  # explicit warmup + measurement periods
  x.report("method") { my_method }
end

# MISTAKE 2: Measuring too few iterations (results dominated by noise)
# BAD:
Benchmark.measure { 10.times { operation } }  # too few — noise matters

# GOOD: benchmark-ips automatically determines meaningful sample count
Benchmark.ips { |x| x.report("op") { operation } }

# MISTAKE 3: Garbage in = garbage out (use identical data per run)
# BAD: shuffled data changes cache behavior between runs
Benchmark.ips do |x|
  x.report("sort") { [3,1,4,1,5,9].shuffle.sort }  # shuffle changes every iter!
end

# GOOD: prepare data outside the benchmark
data = [3,1,4,1,5,9]
Benchmark.ips do |x|
  x.report("sort") { data.dup.sort }  # dup so we measure sort, not setup
end

# MISTAKE 4: GC runs sporadically affecting timing
# SOLUTION: Use bmbm (rehearsal run to force GC before measurement)
Benchmark.bmbm do |x|
  x.report("option1") { option1 }
  x.report("option2") { option2 }
end

# MISTAKE 5: Micro-benchmark doesn't reflect real usage
# A micro-benchmark showing 2x speedup for string comparison
# might represent 0.001% of your app's total time
# Always profile the full request first, then benchmark candidates

# CORRECT WORKFLOW:
# 1. Profile with stackprof to find the real bottleneck
# 2. Benchmark ONLY the bottleneck code
# 3. Verify the fix with a realistic workload test
# 4. Deploy and monitor production metrics

# Interpreting benchmark-ips output:
# "1.23M (± 2.5%) i/s" — 1.23 million iterations/second, ±2.5% variation
# Low variation (< 5%) = reliable measurement
# High variation (> 15%) = noisy — investigate why (GC, I/O, etc.)
```

---

## Answer 4: Memory Profiling a Real Application

**Question:** "Walk me through profiling and fixing a memory leak in a production Ruby service."

**Answer:**

```ruby
# STEP 1: Baseline measurement
initial_rss = `ps -o rss= -p #{Process.pid}`.to_i  # resident set size in KB
puts "Initial RSS: #{initial_rss}KB"

# STEP 2: Run the operation that might be leaking
1000.times { process_request }

after_rss = `ps -o rss= -p #{Process.pid}`.to_i
puts "After 1000 requests RSS: #{after_rss}KB"
puts "Growth: #{after_rss - initial_rss}KB"

# If memory grew significantly, dig deeper:

# STEP 3: Count retained objects
require 'objspace'

GC.start
before_count = ObjectSpace.count_objects_size
1000.times { process_request }
GC.start  # force GC to collect dead objects
after_count  = ObjectSpace.count_objects_size

growth = after_count.merge(before_count) { |k, after, before| after - before }
         .select { |_, v| v > 0 }
         .sort_by { |_, v| -v }

puts "Growing types: #{growth.first(5)}"

# STEP 4: Find the leak with memory_profiler
require 'memory_profiler'

report = MemoryProfiler.report(allow_files: 'app/') do
  100.times { process_request }
end

puts "Retained objects:"
report.retained_objects_by_class.first(10).each do |stat|
  puts "  #{stat.klass}: #{stat.count} objects, #{stat.memsize} bytes"
end

# STEP 5: Common leak patterns and fixes

# LEAK 1: Growing class-level array
class EventStore
  @events = []  # grows forever!
  def self.record(event) = @events << event
end
# Fix: use bounded collection
class EventStore
  MAX_EVENTS = 1000
  @events = []
  def self.record(event)
    @events << event
    @events.shift if @events.size > MAX_EVENTS  # evict oldest
  end
end

# LEAK 2: Callback holding references
class Publisher
  @subscribers = []
  def self.subscribe(obj) = @subscribers << obj
  # obj never removed → subscriber lives as long as Publisher class
end
# Fix: use WeakRef
require 'weakref'
class SafePublisher
  @subscribers = []
  def self.subscribe(obj)
    @subscribers << WeakRef.new(obj)
    @subscribers.reject! { |s| !s.weakref_alive? rescue true }  # prune dead refs
  end
end

# LEAK 3: Thread-local data not cleared
def process_in_thread
  Thread.current[:request_context] = build_context  # never cleared!
  do_work
  # FIX:
ensure
  Thread.current[:request_context] = nil  # always clear in ensure
end
```

---

## Answer 5: Using YJIT Effectively

**Question:** "How do you enable and validate YJIT performance improvements in your application?"

**Answer:**

```ruby
# ENABLING YJIT:
# Option 1: Command line
# ruby --yjit app.rb

# Option 2: Gemfile
# gem 'rails', '~> 7.0'
# In config/environments/production.rb:
# RubyVM::YJIT.enable  # Ruby 3.3+

# Option 3: Environment variable
# RUBY_YJIT_ENABLE=1 ruby app.rb

# VALIDATING YJIT IS ACTIVE AND COMPILING:
if defined?(RubyVM::YJIT) && RubyVM::YJIT.enabled?
  stats = RubyVM::YJIT.runtime_stats
  ratio = (stats[:ratio_in_yjit] * 100).round(1)
  puts "YJIT compilation ratio: #{ratio}%"
  puts "Total YJIT calls: #{stats[:yjit_insns_count]}"
  # > 90% ratio means most of your hot code is YJIT-compiled — good!
  # < 10% ratio means YJIT has little to work with
end

# IDENTIFYING YJIT-FRIENDLY CODE:
# Ruby code that YJIT optimizes well:
def yjit_friendly
  # 1. Type-stable variables (same type on every call)
  total = 0
  [1, 2, 3].each { |n| total += n }  # Integer arithmetic — YJIT inlines this

  # 2. Predictable method calls (same receiver type)
  "hello".length  # String#length always called on String — YJIT specializes

  # 3. Short, frequently-called methods
  def add(a, b) = a + b  # YJIT inlines the body at call sites
end

# Code YJIT can't fully optimize:
def yjit_unfriendly
  # 1. method_missing heavy code
  # 2. send with dynamic method names
  obj.send(method_name_variable)  # can't specialize on unknown method

  # 3. Highly polymorphic call sites
  items.each { |i| i.process }   # if items have 20 different types, harder to specialize

  # 4. C extension calls (YJIT can't look inside)
  JSON.parse(large_json)  # C code, no YJIT benefit inside
end

# MEASURING YJIT IMPACT IN PRODUCTION:
# A/B test: deploy half your servers with YJIT, half without
# Compare: requests/second, p99 latency, CPU utilization

# For a CPU-bound service:
# Expect: 15-50% throughput increase
# For I/O-heavy Rails app:
# Expect: 3-10% throughput increase (most time spent waiting for DB)
```

---

## Answer 6: Optimizing Database-Intensive Ruby Code (Without Rails)

**Question:** "How do you optimize Ruby code that makes many database queries?"

**Answer:**

```ruby
# Setup: plain Ruby with Sequel ORM (demonstrates concepts without Rails)
require 'sequel'
DB = Sequel.connect(ENV['DATABASE_URL'])

# PROBLEM 1: N+1 — load associated records in a loop
# SLOW: N+1 queries
def user_post_counts_slow(user_ids)
  user_ids.map do |user_id|
    count = DB[:posts].where(user_id: user_id).count  # 1 query per user!
    { user_id: user_id, post_count: count }
  end
end

# FAST: single query with GROUP BY
def user_post_counts_fast(user_ids)
  counts = DB[:posts]
    .where(user_id: user_ids)
    .group_and_count(:user_id)
    .each_with_object({}) { |row, h| h[row[:user_id]] = row[:count] }

  user_ids.map { |id| { user_id: id, post_count: counts.fetch(id, 0) } }
end

# PROBLEM 2: Bulk operations
# SLOW: one INSERT per record
def save_users_slow(users)
  users.each { |u| DB[:users].insert(u) }  # N queries
end

# FAST: batch insert
def save_users_fast(users)
  users.each_slice(1000) do |batch|
    DB[:users].multi_insert(batch)  # one INSERT per batch
  end
end

# PROBLEM 3: Loading more columns than needed
# SLOW: loads all columns (may include large blobs, unused data)
def user_names_slow
  DB[:users].all.map { |u| u[:name] }
end

# FAST: select only needed columns
def user_names_fast
  DB[:users].select(:id, :name).map { |u| u[:name] }
end

# PROBLEM 4: Counting without loading
# SLOW: loads all records to count
def active_user_count_slow
  DB[:users].where(active: true).all.count  # loads all active users into memory!
end

# FAST: COUNT at DB level
def active_user_count_fast
  DB[:users].where(active: true).count  # SELECT COUNT(*) — one integer returned
end

# PROBLEM 5: Pagination
# SLOW: loads all records, slices in Ruby
def paginate_slow(page, per_page)
  DB[:users].all.drop((page - 1) * per_page).first(per_page)
end

# FAST: LIMIT/OFFSET at DB level
def paginate_fast(page, per_page)
  DB[:users].limit(per_page).offset((page - 1) * per_page).all
end

# Even better: keyset pagination (no OFFSET performance penalty)
def paginate_keyset(last_id, per_page)
  DB[:users].where(Sequel.lit("id > ?", last_id)).limit(per_page).all
end
```

---

## Answer 7: Profiling and Optimizing a Computation-Heavy Algorithm

**Question:** "You have a graph search algorithm running in 10 seconds. Walk me through optimizing it."

**Answer:**

```ruby
# Original: naive breadth-first search with poor data structures
def find_shortest_path_v1(graph, start, target)
  visited = []  # Array: O(n) membership check
  queue   = [[start]]

  until queue.empty?
    path = queue.shift  # Array#shift: O(n) — moves all elements!
    node = path.last
    return path if node == target
    next if visited.include?(node)  # O(n) membership check

    visited << node
    graph[node].each do |neighbor|
      queue << (path + [neighbor])  # path + creates new array each time!
    end
  end

  nil
end

# Profile it:
require 'stackprof'
StackProf.run(mode: :cpu, out: 'graph.dump') do
  10.times { find_shortest_path_v1(large_graph, :start, :end) }
end
# Top hot spots: Array#shift, Array#include?, Array#+

# Optimized v2: Fix data structures
def find_shortest_path_v2(graph, start, target)
  visited = Set.new          # Set: O(1) membership check
  queue   = [[start]]        # still Array#shift — will fix

  until queue.empty?
    path = queue.shift       # still O(n) — fix this next
    node = path.last
    return path if node == target
    next if visited.include?(node)  # now O(1) with Set

    visited.add(node)
    graph[node].each do |neighbor|
      queue << path + [neighbor]  # still allocates new array
    end
  end
  nil
end

# Optimized v3: Use proper queue + avoid path array copying
require 'set'

def find_shortest_path_v3(graph, start, target)
  visited  = Set.new
  # Use a simple deque (Array as queue is O(n) for shift — use index trick):
  queue    = [[start]]
  qi       = 0  # queue index instead of shifting

  while qi < queue.size
    path = queue[qi]
    qi += 1
    node = path.last

    return path if node == target
    next if visited.include?(node)

    visited.add(node)
    graph[node]&.each do |neighbor|
      unless visited.include?(neighbor)
        queue << (path + [neighbor])  # still copying — acceptable tradeoff
      end
    end
  end

  nil
end

# Benchmark:
require 'benchmark/ips'
graph = generate_test_graph(1000, 0.05)  # 1000 nodes, 5% density

Benchmark.ips do |x|
  x.config(warmup: 2, time: 5)
  x.report("v1: Array visited") { find_shortest_path_v1(graph, :n0, :n999) }
  x.report("v2: Set visited")   { find_shortest_path_v2(graph, :n0, :n999) }
  x.report("v3: index queue")   { find_shortest_path_v3(graph, :n0, :n999) }
  x.compare!
end
# v1: ~500 i/s
# v2: ~2000 i/s (4x speedup from Set)
# v3: ~3500 i/s (7x speedup overall)
```

---

## Answer 8: Lazy Evaluation and Infinite Sequences

**Question:** "Explain lazy evaluation in Ruby with practical examples, including generating infinite sequences."

**Answer:**

```ruby
# FINITE LAZY CHAIN: saves memory and computation
class DataPipeline
  def initialize(source_path)
    @source_path = source_path
  end

  def process
    # None of these run until .force or iteration
    File.foreach(@source_path)
      .lazy
      .map    { |line| JSON.parse(line.chomp) }
      .select { |record| record["active"] == true }
      .map    { |record| normalize(record) }
      .first(100)  # stops after collecting 100 valid records
    # May scan only a small fraction of the file
  end
end

# INFINITE SEQUENCES with Enumerator
class FibonacciSequence
  include Enumerable

  def each
    a, b = 0, 1
    loop do
      yield a
      a, b = b, a + b
    end
  end
end

fibs = FibonacciSequence.new

# Works because of lazy evaluation:
fibs.lazy.select { |n| n.even? }.first(10)
# => [0, 2, 8, 34, 144, 610, 2584, 10946, 46368, 196418]

# Find first Fibonacci > 1 million
fibs.lazy.find { |n| n > 1_000_000 }
# => 1346269

# All Fibonacci numbers between 100 and 1000:
fibs.lazy.select { |n| n >= 100 }.take_while { |n| n <= 1000 }.to_a
# => [144, 233, 377, 610, 987]

# CUSTOM LAZY ENUMERATOR for database streaming:
class DatabaseCursor
  def initialize(query, batch_size: 1000)
    @query      = query
    @batch_size = batch_size
  end

  def each
    return to_enum(:each) unless block_given?

    offset = 0
    loop do
      batch = execute_query(offset: offset, limit: @batch_size)
      break if batch.empty?

      batch.each { |record| yield record }
      break if batch.size < @batch_size  # last batch
      offset += @batch_size
    end
  end

  # Make it lazy-capable
  def lazy = each.lazy
end

cursor = DatabaseCursor.new("SELECT * FROM events ORDER BY id")
# Process only events matching a condition, stop after 50:
cursor.lazy
  .select { |e| e[:event_type] == "purchase" }
  .first(50)
# Fetches batches from DB lazily until 50 purchases found
```

---

## Answer 9: String Performance Deep Dive

**Question:** "When does Ruby intern strings? How do frozen strings affect performance?"

**Answer:**

```ruby
# STRING INTERNING RULES IN RUBY:

# 1. Symbol literals are always interned (same object)
:hello.object_id == :hello.object_id  # => true always

# 2. String literals: interned only with frozen_string_literal: true
# Without pragma:
"hello".object_id == "hello".object_id  # => false (new object each time)

# frozen_string_literal: true
"hello".object_id == "hello".object_id  # => true (interned, same object)

# 3. String#-@ (freeze) interns the string:
str1 = -"hello"  # String#-@ freezes and interns
str2 = -"hello"
str1.equal?(str2)  # => true (same interned object)

# 4. String.new("hello").freeze does NOT always intern
s1 = "hello".freeze
s2 = "hello".freeze
s1.equal?(s2)  # => depends on Ruby version (may or may not be interned)

# PERFORMANCE IMPLICATIONS:
require 'benchmark/ips'

# Frozen string as hash key (avoids dup on access):
frozen_key = "user_id".freeze

Benchmark.ips do |x|
  x.report("frozen key")  { { frozen_key => 42 } }
  x.report("literal key") { { "user_id" => 42 } }  # new string each time
  x.compare!
end

# String comparison:
Benchmark.ips do |x|
  str = "hello world"
  x.report("== compare")    { str == "hello world" }
  x.report("frozen ==")     { str == "hello world".freeze }
  x.report("equal? (obj)")  { str.equal?("hello world") }  # always false
  x.compare!
end

# In practice:
# - Use frozen string literals pragma in all files
# - Use symbols for hash keys in your own code
# - Use frozen strings for constant values used as hash keys
CACHE_KEY_PREFIX = "user:profile:".freeze  # interned, reused

def cache_key(user_id)
  CACHE_KEY_PREFIX + user_id.to_s  # + creates new string, but prefix reused
  # vs:
  "user:profile:#{user_id}"  # interpolation creates one string (faster for simple cases)
end
```

---

## Answer 10: Ractor-Based Parallelism for CPU-Intensive Work

**Question:** "How do Ractors help with parallel execution and what are their limitations?"

**Answer:**

```ruby
# Ractors provide true parallelism by running on separate OS threads
# with NO shared mutable state (unlike threads with GIL in MRI)

# Simple Ractor example:
result_ractor = Ractor.new(1..1_000_000) do |range|
  range.sum { |n| n * n }  # runs in parallel with main thread
end

# Main thread can do other work while Ractor runs:
other_work = do_something_else

result = result_ractor.take  # blocks until Ractor completes
puts result

# Parallel map with Ractors:
def parallel_map(items, workers: 4)
  items.each_slice(items.size.fdiv(workers).ceil.to_i)
       .map.with_index do |chunk, i|
         Ractor.new(chunk) do |c|
           c.map { |item| transform(item) }
         end
       end
       .flat_map(&:take)
end

# LIMITATIONS:
# 1. Only shareable objects can be passed to/from Ractors:
#    - Frozen objects (fine)
#    - Immutable primitives: Integer, Float, Symbol (fine)
#    - Objects explicitly made shareable: Ractor.make_shareable(obj)
#    - NOT: regular mutable objects (raises Ractor::IsolationError)

# Example of what WON'T work:
mutable = { key: "value" }  # mutable hash
Ractor.new(mutable) { |m| m[:key] }
# => Ractor::IsolationError: can not share mutable object

# What WORKS:
frozen = { key: "value" }.freeze
Ractor.new(frozen) { |f| f[:key] }  # => "value" ✓

# 2. Most gems are NOT Ractor-safe (they use shared mutable state)
# ActiveRecord, for example, cannot be used inside Ractors

# PRACTICAL USE CASES FOR RACTORS (as of Ruby 3.3):
# - CPU-bound data transformation (no I/O)
# - Mathematical computation (numerical methods, statistics)
# - Parallel text/binary parsing with frozen input data
# - Worker pools that process independent data chunks

# For I/O-bound parallel work, use async/threads instead:
require 'async'
Async do
  tasks = urls.map { |url| Async { HTTP.get(url) } }
  results = tasks.map(&:wait)
end
```
