# Chapter 08: Ruby Performance — Scenario Questions

**S1. Your Rails app is slow. Walk me through a systematic performance investigation process.**

```ruby
# Step 1: Identify WHERE the slowness is (don't optimize blind)
# Tools: New Relic, Skylight, rack-mini-profiler, bullet gem

# rack-mini-profiler shows per-request breakdown in the browser
# Add to Gemfile: gem 'rack-mini-profiler'

# Step 2: Check for N+1 queries first (most common Rails bottleneck)
# Add bullet gem to development:
# gem 'bullet'
# config/environments/development.rb:
config.after_initialize do
  Bullet.enable        = true
  Bullet.alert         = true
  Bullet.rails_logger  = true
  Bullet.add_footer    = true
end

# Step 3: Profile memory allocations
# Add to your controller:
require 'memory_profiler'
report = MemoryProfiler.report { yield }
Rails.logger.info(report.pretty_print_str)

# Step 4: Profile CPU time with stackprof
StackProf.run(mode: :cpu, out: 'tmp/stackprof.dump') do
  10.times { get '/api/v1/products' }
end
# Then: stackprof tmp/stackprof.dump --text --limit 20

# Step 5: Check GC pressure
puts GC.stat.slice(:minor_gc_count, :major_gc_count, :heap_live_slots)

# Step 6: Benchmark specific candidates
require 'benchmark/ips'
Benchmark.ips do |x|
  x.report("current") { current_implementation(data) }
  x.report("optimized") { optimized_implementation(data) }
  x.compare!
end
```

---

**S2. You have a method that processes a 1-million-row CSV file. It's using 2GB of memory. How do you fix it?**

```ruby
# PROBLEM: loading everything into memory
def process_csv(path)
  CSV.read(path).map do |row|  # loads ALL 1M rows into RAM at once
    transform(row)
  end
end

# FIX 1: Streaming with CSV.foreach (lazy row-by-row processing)
def process_csv_streaming(path)
  CSV.foreach(path, headers: true) do |row|  # one row in memory at a time
    process_row(row.to_h)
  end
end

# FIX 2: Lazy enumerator for transformations
def transform_csv(path)
  Enumerator.new do |yielder|
    CSV.foreach(path, headers: true) do |row|
      yielder << transform(row.to_h)
    end
  end.lazy
end

# Usage: only processes what's needed
transform_csv("big.csv").first(100)   # processes 100 rows, stops
transform_csv("big.csv").select { |r| r[:status] == "active" }.first(50)

# FIX 3: Batched processing (good for DB inserts)
def batch_import(path, batch_size: 1000)
  batch = []
  CSV.foreach(path, headers: true) do |row|
    batch << row.to_h
    if batch.size >= batch_size
      User.insert_all(batch)  # one DB call per batch
      batch.clear
    end
  end
  User.insert_all(batch) unless batch.empty?  # final partial batch
end

# FIX 4: Use smarter_csv gem for even better performance
require 'smarter_csv'
SmarterCSV.process("big.csv", chunk_size: 1000) do |chunk|
  process_batch(chunk)  # chunk is array of hashes
end
```

---

**S3. Your application builds a large report by concatenating thousands of strings. It's slow. Optimize it.**

```ruby
# SLOW: + concatenation builds a new object each time
def build_report_slow(items)
  report = ""
  items.each do |item|
    report = report + "- #{item[:name]}: #{item[:value]}\n"  # N allocations
  end
  report
end

# FAST option 1: << mutation (in-place concatenation)
def build_report_mutation(items)
  report = +""  # unfrozen mutable string (+ prefix)
  items.each do |item|
    report << "- " << item[:name] << ": " << item[:value].to_s << "\n"
  end
  report
end

# FAST option 2: join (best for simple cases)
def build_report_join(items)
  lines = items.map { |item| "- #{item[:name]}: #{item[:value]}" }
  lines.join("\n")
end

# FAST option 3: StringIO buffer (best for complex reports)
require 'stringio'
def build_report_stringio(items)
  io = StringIO.new
  io << "Report: #{Time.now}\n"
  io << "=" * 40 << "\n"
  items.each do |item|
    io.printf("%-20s %10.2f\n", item[:name], item[:value])
  end
  io.string
end

# BENCHMARK:
require 'benchmark/ips'
items = Array.new(10_000) { |i| { name: "Item #{i}", value: rand(100.0) } }

Benchmark.ips do |x|
  x.report("+ concat")    { build_report_slow(items) }
  x.report("<< mutate")   { build_report_mutation(items) }
  x.report("join")        { build_report_join(items) }
  x.report("StringIO")    { build_report_stringio(items) }
  x.compare!
end
# join is often fastest; << mutation close behind; + is 5-10x slower
```

---

**S4. You have a hot loop that calls a polymorphic method. After profiling, it's the bottleneck. What do you try?**

```ruby
# HOT LOOP (1M iterations calling method on mixed types):
def process_all(items)
  items.map { |item| item.process }  # polymorphic dispatch
end

# Optimization 1: Avoid method dispatch with duck-typed lambda
PROCESSORS = {
  "Integer" => ->(x) { x * 2 },
  "String"  => ->(x) { x.upcase },
  "Float"   => ->(x) { x.round(2) }
}.freeze

def process_fast(items)
  items.map { |item| PROCESSORS[item.class.name]&.call(item) || item.process }
end

# Optimization 2: Separate by type before loop (type-stable dispatch)
def process_typed(items)
  grouped = items.group_by(&:class)
  result  = []

  grouped[Integer]&.each { |n| result << n * 2 }
  grouped[String]&.each  { |s| result << s.upcase }
  grouped[Float]&.each   { |f| result << f.round(2) }

  result
end

# Optimization 3: Use Enumerable#each_with_object to avoid intermediate arrays
def process_no_intermediate(items)
  items.each_with_object([]) do |item, acc|
    acc << item.process
  end
end

# Optimization 4: Consider C extension for the truly hot inner loop
# Write a simple C extension that processes Integer arrays with no overhead
```

---

**S5. Your memoization is broken and calling expensive operations repeatedly. Debug and fix it.**

```ruby
class PriceCalculator
  def unit_price(product_id)
    @prices ||= {}
    @prices[product_id] ||= fetch_price(product_id)  # BUG: fails for price = 0 or false
  end

  def discounted?(product_id)
    @discount ||= check_discount(product_id)  # BUG: fails if no discount (false/nil)
  end

  private

  def fetch_price(product_id)
    # Returns 0 for free products!
    DB.query("SELECT price FROM products WHERE id = ?", product_id).first
  end

  def check_discount(product_id)
    # Returns false when no discount
    DiscountService.has_discount?(product_id)
  end
end

# BUG DEMONSTRATION:
calc = PriceCalculator.new
calc.unit_price(1)   # fetches: 0 (free product)
calc.unit_price(1)   # ||= re-evaluates because 0 is falsy → fetches again!

calc.discounted?(1)  # check returns false
calc.discounted?(1)  # ||= re-evaluates → fetches again!

# FIX: proper caching with explicit key? check
class FixedPriceCalculator
  def unit_price(product_id)
    @prices ||= {}
    unless @prices.key?(product_id)
      @prices[product_id] = fetch_price(product_id)
    end
    @prices[product_id]
  end

  def discounted?(product_id)
    unless defined?(@discounted)
      @discounted = check_discount(product_id)
    end
    @discounted
  end
end

# FIX 2: Using Hash#fetch with a default sentinel
SENTINEL = Object.new  # unique object that can't be a real value

def unit_price(product_id)
  @prices ||= {}
  cached = @prices.fetch(product_id, SENTINEL)
  if cached.equal?(SENTINEL)
    @prices[product_id] = fetch_price(product_id)
  else
    cached
  end
end
```

---

**S6. Profile and optimize a Ruby data pipeline that transforms 100k records.**

```ruby
# ORIGINAL: functional, clean, but slow
def transform_pipeline(records)
  records
    .map    { |r| parse_dates(r) }         # allocates 100k objects
    .select { |r| r[:status] == "active" } # allocates up to 100k objects
    .map    { |r| enrich_with_geo(r) }     # allocates N objects (after filter)
    .map    { |r| format_output(r) }       # allocates N objects
end

# OPTIMIZED 1: lazy chain (only processes records that make it through)
def transform_lazy(records)
  records
    .lazy
    .map    { |r| parse_dates(r) }
    .select { |r| r[:status] == "active" }
    .map    { |r| enrich_with_geo(r) }
    .map    { |r| format_output(r) }
    .force  # materialize only at the end
end

# OPTIMIZED 2: fused loop (single pass, less allocation)
def transform_fused(records)
  result = []
  records.each do |r|
    parsed = parse_dates(r)
    next unless parsed[:status] == "active"

    enriched = enrich_with_geo(parsed)
    result << format_output(enriched)
  end
  result
end

# OPTIMIZED 3: parallel processing with Ractors (Ruby 3.0+)
def transform_parallel(records)
  workers = 4
  chunks  = records.each_slice(records.size / workers + 1).to_a
  ractors = chunks.map do |chunk|
    Ractor.new(chunk) do |c|
      c.map { |r| transform_single(r) }
    end
  end
  ractors.flat_map(&:take)
end

# Benchmark the approaches:
require 'benchmark'
records = generate_test_data(100_000)
Benchmark.bm(20) do |x|
  x.report("original")   { transform_pipeline(records) }
  x.report("lazy")       { transform_lazy(records) }
  x.report("fused")      { transform_fused(records) }
end
```

---

**S7. You're asked to reduce memory usage of a long-running background worker that processes jobs from a queue.**

```ruby
# PROBLEM: Long-running worker accumulates memory over time
class BackgroundWorker
  def run
    loop do
      job = queue.pop
      process(job)
      # Memory leak: objects referenced in closures, cached results
    end
  end

  def process(job)
    # Creates many objects per job; none are explicitly freed
    data = fetch_data(job[:id])            # large hash
    result = expensive_transform(data)      # another large object
    store_result(job[:id], result)
    # data and result are dereferenced, but GC might not run immediately
  end
end

# FIX 1: Explicit GC hints after batches
class OptimizedWorker
  GC_EVERY = 100  # run GC hint every 100 jobs

  def run
    processed = 0
    loop do
      job = queue.pop
      process(job)
      processed += 1

      if processed % GC_EVERY == 0
        GC.start          # hint to GC to run
        GC.compact        # compact heap (Ruby 2.7+)
      end
    end
  end
end

# FIX 2: Fork a child process per batch (memory is released on exit)
class ForkingWorker
  BATCH_SIZE = 1000

  def run
    loop do
      batch = queue.pop_batch(BATCH_SIZE)

      pid = fork do
        process_batch(batch)
        exit 0
      end
      Process.waitpid(pid)
      # Child process's memory is fully released after exit
    end
  end
end

# FIX 3: Use WeakRef for caches that can be GC'd
require 'weakref'

class WorkerWithSoftCache
  def initialize
    @cache = {}
  end

  def get_config(key)
    cached = @cache[key]
    begin
      return cached.__getobj__ if cached&.weakref_alive?
    rescue WeakRef::RefError
      # Object was GC'd
    end

    value = load_config(key)
    @cache[key] = WeakRef.new(value)
    value
  end
end
```

---

**S8. Write a benchmark comparing different ways to find unique elements in a large array.**

```ruby
require 'benchmark/ips'
require 'set'

data = Array.new(100_000) { rand(10_000) }  # lots of duplicates

Benchmark.ips do |x|
  x.config(time: 5, warmup: 2)

  x.report("Array#uniq") do
    data.uniq
  end

  x.report("Set + to_a") do
    Set.new(data).to_a
  end

  x.report("sort + uniq") do
    data.sort.uniq
  end

  x.report("each_with_object hash") do
    data.each_with_object({}) { |v, h| h[v] = true }.keys
  end

  x.report("tally.keys") do
    data.tally.keys
  end

  x.compare!
end

# Memory comparison using memory_profiler:
require 'memory_profiler'

[:uniq, :set, :hash_dedup].each do |approach|
  report = MemoryProfiler.report do
    case approach
    when :uniq   then data.uniq
    when :set    then Set.new(data).to_a
    when :hash_dedup then data.each_with_object({}) { |v, h| h[v] = nil }.keys
    end
  end
  puts "#{approach}: #{report.total_allocated} bytes allocated"
end

# Typical results:
# Array#uniq:           fastest for simple values (C implementation)
# Set:                  good when you need Set semantics (member? is O(1))
# sort + uniq:          slowest (O(n log n))
# tally.keys:           Ruby 2.7+, returns with counts if needed
```

---

**S9. Your symbol table is growing unexpectedly. Identify the cause and fix it.**

```ruby
# PROBLEM: Dynamic symbol creation from user input
# In Ruby < 2.2, symbols were NEVER garbage collected
# Creating symbols from user input = potential memory bomb

# BAD: Dynamic symbol creation
def process_param(key)
  key.to_sym  # if key comes from user input, unlimited symbols!
end

# Demonstration of the symbol table growth:
before = Symbol.all_symbols.size
1000.times { |i| :"dynamic_#{i}".to_s.to_sym }  # creates 1000 permanent symbols
after = Symbol.all_symbols.size
puts "Symbols created: #{after - before}"

# In Ruby < 2.2: these 1000 symbols live forever (memory leak)
# In Ruby 2.2+: dynamically created symbols ARE garbage collectible
# BUT: interned symbols (symbol literals, :foo) are still permanent

# SAFE alternatives:
def process_param_safe(key)
  # Option 1: keep as string
  key.to_s

  # Option 2: use frozen string
  key.to_s.freeze

  # Option 3: validate against known symbols
  ALLOWED = %i[name email age role].freeze
  sym = key.to_sym
  raise ArgumentError unless ALLOWED.include?(sym)
  sym
end

# Check for symbol table size in long-running process:
GC.stat[:total_allocated_objects]  # not specific to symbols
Symbol.all_symbols.count           # total number of live symbols
# In a healthy app this should be roughly stable after boot

# Visualizing symbol growth:
baseline = Symbol.all_symbols.count
# ... run some code ...
new_syms = Symbol.all_symbols.count - baseline
puts "New symbols created: #{new_syms}"
```

---

**S10. Optimize a recursive Fibonacci implementation for production use.**

```ruby
# Naive recursive: O(2^n) time, each call re-computes
def fib_naive(n)
  return n if n <= 1
  fib_naive(n - 1) + fib_naive(n - 2)
end

fib_naive(40)  # takes ~1.5 seconds, 2^40 calls

# Memoization: O(n) time, O(n) space
def fib_memo(n, cache = {})
  return cache[n] if cache.key?(n)
  return n if n <= 1
  cache[n] = fib_memo(n - 1, cache) + fib_memo(n - 2, cache)
end

fib_memo(40)   # instant
fib_memo(1000) # still fast, but stack depth issue

# Iterative: O(n) time, O(1) space, no stack overflow
def fib_iter(n)
  return n if n <= 1
  a, b = 0, 1
  (n - 1).times { a, b = b, a + b }
  b
end

fib_iter(1_000_000)  # fast and no stack issues

# Constant time using matrix exponentiation (O(log n)):
def fib_matrix(n)
  def matrix_mult(a, b)
    [[a[0][0]*b[0][0] + a[0][1]*b[1][0],  a[0][0]*b[0][1] + a[0][1]*b[1][1]],
     [a[1][0]*b[0][0] + a[1][1]*b[1][0],  a[1][0]*b[0][1] + a[1][1]*b[1][1]]]
  end

  def matrix_pow(m, n)
    return [[1,0],[0,1]] if n == 0
    return m if n == 1
    half = matrix_pow(m, n / 2)
    result = matrix_mult(half, half)
    n.odd? ? matrix_mult(result, m) : result
  end

  return n if n <= 1
  matrix_pow([[1,1],[1,0]], n)[0][1]
end

# Benchmark:
require 'benchmark/ips'
Benchmark.ips do |x|
  x.report("naive")  { fib_naive(30) }
  x.report("memo")   { fib_memo(30) }
  x.report("iter")   { fib_iter(30) }
  x.report("matrix") { fib_matrix(30) }
  x.compare!
end
```

---

**S11. You're asked to optimize a hot path that allocates many temporary arrays. Show allocation reduction techniques.**

```ruby
require 'memory_profiler'

# ORIGINAL: many intermediate arrays
def process_data(records)
  records
    .map    { |r| [r[:id], r[:name].upcase] }   # intermediate array
    .select { |id, name| name.start_with?("A") } # another intermediate
    .map    { |id, name| { id: id, display: name } }  # final array
end

report = MemoryProfiler.report { process_data(Array.new(10_000) { |i| { id: i, name: "Name #{i}" } }) }
puts "Original: #{report.total_allocated} bytes"

# OPTIMIZED 1: Single-pass with each_with_object
def process_data_v2(records)
  records.each_with_object([]) do |r, acc|
    uname = r[:name].upcase
    acc << { id: r[:id], display: uname } if uname.start_with?("A")
  end
end

# OPTIMIZED 2: filter_map (Ruby 2.7+) - select + map in one pass
def process_data_v3(records)
  records.filter_map do |r|
    uname = r[:name].upcase
    { id: r[:id], display: uname } if uname.start_with?("A")
  end
end

# OPTIMIZED 3: Pre-allocate result array
def process_data_v4(records)
  result = Array.new  # reserve capacity if you know approximate size
  result = []
  records.each do |r|
    uname = r[:name].upcase
    result << { id: r[:id], display: uname } if uname.start_with?("A")
  end
  result
end

report = MemoryProfiler.report { process_data_v3(Array.new(10_000) { |i| { id: i, name: "Name #{i}" } }) }
puts "filter_map: #{report.total_allocated} bytes"
# filter_map version: ~40% fewer allocations
```

---

**S12. How would you diagnose and fix a Ruby application that is CPU-bound vs I/O-bound?**

```ruby
# Diagnosing CPU-bound vs I/O-bound:
require 'benchmark'

result = Benchmark.measure { your_operation }
puts "User CPU: #{result.utime}"   # time in user-space Ruby code
puts "System:   #{result.stime}"   # time in kernel (I/O syscalls)
puts "Real:     #{result.real}"    # wall clock time

# CPU-bound: result.utime ≈ result.real (CPU is the bottleneck)
# I/O-bound: result.real >> result.utime (waiting for external resources)

# CPU-BOUND OPTIMIZATIONS:
# 1. Enable YJIT: ruby --yjit app.rb
# 2. Use Ractors for parallel execution (Ruby 3+)
# 3. Offload to C extensions
# 4. Algorithmic improvements (O(n²) → O(n log n))

# I/O-BOUND OPTIMIZATIONS:
# 1. Async I/O with Fiber scheduler
require 'async'
Async do
  internet = Async::HTTP::Internet.new
  responses = urls.map do |url|
    Async { internet.get(url) }
  end.map(&:wait)
end

# 2. Thread pool for parallel I/O
require 'concurrent-ruby'
pool = Concurrent::FixedThreadPool.new(10)
futures = urls.map do |url|
  Concurrent::Future.execute(executor: pool) { fetch(url) }
end
results = futures.map(&:value)

# 3. Connection pooling
require 'connection_pool'
DB_POOL = ConnectionPool.new(size: 10, timeout: 5) { PG.connect(DB_URL) }

DB_POOL.with { |conn| conn.exec(sql) }  # borrows from pool, returns when done
```
