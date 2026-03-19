# Chapter 05 — Concurrency & Threading: Structured Answers

---

## Answer 1 — "Explain the GVL and when threads are useful in Ruby"

The GVL (Global VM Lock) is a mutex inside MRI Ruby's VM that allows only one thread to execute Ruby bytecode at a time. It exists to protect MRI's internal data structures from corruption in multi-threaded environments.

**What the GVL means in practice:**

```ruby
# CPU-bound: no benefit from multiple threads
# Example: parallel matrix multiplication
def matrix_multiply(a, b)
  # pure Ruby computation
  a.size.times.map { |i|
    a[i].size.times.map { |j|
      a[i].zip(b.transpose[j]).sum { |x, y| x * y }
    }
  }
end

# Even with 8 threads, only 1 runs at a time for CPU work
# → no speedup over a single thread
```

**When threads DO provide speedup:**

```ruby
# I/O-bound: GVL IS released during blocking operations
require 'net/http'

# Sequential: 5 seconds (1s per request)
5.times { Net::HTTP.get(URI("http://httpbin.org/delay/1")) }

# Threaded: ~1 second (all requests run concurrently)
threads = 5.times.map {
  Thread.new { Net::HTTP.get(URI("http://httpbin.org/delay/1")) }
}
threads.each(&:join)

# GVL is released for:
# - Network I/O (HTTP, TCP, database)
# - File I/O
# - sleep / wait
# - C extensions that call rb_thread_blocking_region
```

**The full concurrency comparison:**

| Approach | CPU parallelism | I/O concurrency | Memory | Complexity |
|----------|----------------|-----------------|--------|------------|
| Threads | No (GVL) | Yes | Shared | Medium |
| Ractor (3.0+) | Yes | Yes | Isolated | High |
| Fork | Yes | Yes | Copied | Medium |
| Fiber | No | Yes (with scheduler) | Shared | Low |

---

## Answer 2 — "How do you make a class thread-safe in Ruby?"

```ruby
# Thread-safety means: correct behavior regardless of thread scheduling

# Level 1: Instance variables (per-object state) — usually thread-safe
#           because each object is typically owned by one thread

# Level 2: Class-level state — NEEDS synchronization
class ConnectionPool
  @pool  = []
  @mutex = Mutex.new

  class << self
    def checkout
      @mutex.synchronize do
        @pool.pop || create_new_connection
      end
    end

    def checkin(conn)
      @mutex.synchronize do
        @pool << conn
      end
    end

    private

    def create_new_connection
      Database.connect
    end
  end
end

# Level 3: Shared mutable state — ALWAYS needs synchronization
class Counter
  def initialize
    @value = 0
    @mutex = Mutex.new
  end

  def increment
    @mutex.synchronize { @value += 1 }
  end

  def value
    @mutex.synchronize { @value }
  end
end

# Level 4: Using concurrent-ruby for complex needs
require 'concurrent'

class Cache
  def initialize
    @store = Concurrent::Hash.new
  end

  def fetch(key, &block)
    @store.compute_if_absent(key, &block)
  end

  def delete(key)
    @store.delete(key)
  end
end
```

---

## Answer 3 — "Explain Fibers and when you'd use them instead of Threads"

```ruby
# Fiber: cooperative coroutine — manually scheduled, single-threaded
# Thread: preemptive concurrency — OS schedules, shared memory

# USE FIBERS WHEN:
# 1. You need lazy/incremental computation
# 2. You want coroutine-style control flow
# 3. You're implementing generators or state machines
# 4. You want async I/O without threads (Ruby 3.1+ Fiber Scheduler)

# Example 1: Lazy data source
class DatabaseCursor
  def initialize(query)
    @query = query
    @fiber = Fiber.new { fetch_pages }
  end

  def next_record
    @fiber.resume
  end

  private

  def fetch_pages
    page = 1
    loop do
      records = DB.query(@query, page: page, per_page: 100)
      break if records.empty?
      records.each { |r| Fiber.yield(r) }
      page += 1
    end
    nil  # signals exhaustion
  end
end

cursor = DatabaseCursor.new("SELECT * FROM users")
while (record = cursor.next_record)
  process(record)
end

# Example 2: State machine with Fiber
traffic_light = Fiber.new do
  loop do
    Fiber.yield :green
    Fiber.yield :yellow
    Fiber.yield :red
  end
end

10.times { puts traffic_light.resume }
# green, yellow, red, green, yellow, red, ...
```

---

## Answer 4 — "When would you use Ractors and what are the constraints?"

```ruby
# Use Ractors when:
# - You have CPU-bound work that can be parallelized
# - Each unit of work is truly independent (no shared state)
# - You're on Ruby 3.0+ in production

# Example: parallel document processing
documents = load_documents(1000)   # 1000 text documents

# Without Ractors: single-threaded, ~60s
documents.map { |doc| analyze(doc) }

# With Ractors: true parallelism on multi-core
BATCH_SIZE = 100
results = documents.each_slice(BATCH_SIZE).flat_map do |batch|
  ractors = batch.map { |doc|
    Ractor.new(doc.freeze) { |d| analyze(d) }
  }
  ractors.map(&:take)
end

# Constraints you must know:
# 1. No shared mutable state — pass copies or frozen objects
# 2. No globals without special handling
# 3. Most existing gems are NOT Ractor-safe (yet)
# 4. Ractor#send moves objects (main ractor loses them)
# 5. Error handling is more complex
# 6. Still experimental/evolving in Ruby 3.x

# Message passing between Ractors:
receiver = Ractor.new do
  result = 0
  while (n = Ractor.receive) != :done
    result += n
  end
  result
end

(1..100).each { |n| receiver.send(n) }
receiver.send(:done)
puts receiver.take   # => 5050
```

---

## Answer 5 — "Explain producer-consumer pattern in Ruby with Threads"

```ruby
# Classic producer-consumer with Queue (thread-safe)
require 'thread'

class Pipeline
  def initialize(num_workers: 4)
    @queue      = SizedQueue.new(50)    # bounded: max 50 items at once
    @results    = Queue.new
    @num_workers = num_workers
  end

  def process(items, &work)
    start_workers(work)
    feed_items(items)
    collect_results(items.size)
  end

  private

  def start_workers(work)
    @workers = @num_workers.times.map do
      Thread.new do
        loop do
          item = @queue.pop
          break if item == :done
          begin
            result = work.call(item)
            @results << { item: item, result: result, error: nil }
          rescue => e
            @results << { item: item, result: nil, error: e.message }
          end
        end
      end
    end
  end

  def feed_items(items)
    items.each { |item| @queue << item }
    @num_workers.times { @queue << :done }
    @workers.each(&:join)
  end

  def collect_results(count)
    count.times.map { @results.pop }
  end
end

# Usage:
pipeline = Pipeline.new(num_workers: 8)
urls = 1000.times.map { |i| "http://api.example.com/item/#{i}" }

results = pipeline.process(urls) do |url|
  Net::HTTP.get(URI(url))
end

successful = results.select { |r| r[:error].nil? }
puts "Processed: #{successful.size}/#{results.size}"
```

---

## Answer 6 — "How does the Fiber Scheduler enable async I/O?"

```ruby
# Ruby 3.1+ introduced a Fiber Scheduler interface
# Third-party schedulers (like `async` gem) implement this

# Traditional threads for concurrent I/O:
# - Each thread is an OS thread (expensive: ~8MB stack each)
# - 1000 concurrent requests = 1000 OS threads = ~8GB RAM

# Fiber scheduler: thousands of concurrent I/O operations on ONE thread
# - Each "connection" is a Fiber (~few KB)
# - Scheduler switches between fibers when they block on I/O
# - OS level: non-blocking I/O (select/epoll/kqueue)

require 'async'
require 'async/http/internet'

Async do |task|
  internet = Async::HTTP::Internet.new

  # 1000 concurrent HTTP requests — single thread, ~1000 fibers
  responses = 1000.times.map do |i|
    task.async do
      internet.get("http://api.example.com/item/#{i}")
    end
  end

  results = responses.map { |task| task.wait.read }
ensure
  internet&.close
end

# How it works:
# 1. task.async creates a Fiber
# 2. Fiber runs until it hits I/O (HTTP request)
# 3. Scheduler suspends the fiber (Fiber.yield internally)
# 4. Scheduler switches to another fiber
# 5. When I/O completes (epoll event), scheduler resumes the waiting fiber
# 6. No OS threads needed for waiting — just OS file descriptors
```

---

## Answer 7 — "How do you implement a thread-safe connection pool?"

```ruby
class ConnectionPool
  def initialize(size:, &block)
    @size       = size
    @create     = block
    @mutex      = Mutex.new
    @available  = ConditionVariable.new
    @connections = []
    @checked_out = 0

    # Pre-create all connections
    size.times { @connections << @create.call }
  end

  def with_connection
    conn = checkout
    yield conn
  ensure
    checkin(conn) if conn
  end

  def checkout
    @mutex.synchronize do
      @available.wait(@mutex) while @connections.empty?
      @connections.pop.tap { @checked_out += 1 }
    end
  end

  def checkin(conn)
    @mutex.synchronize do
      @connections.push(conn)
      @checked_out -= 1
      @available.signal  # wake up a waiting thread
    end
  end

  def stats
    @mutex.synchronize do
      { total: @size, available: @connections.size, in_use: @checked_out }
    end
  end
end

# Usage:
pool = ConnectionPool.new(size: 5) { Database.connect }

threads = 20.times.map do
  Thread.new do
    pool.with_connection do |conn|
      conn.query("SELECT 1")
    end
  end
end

threads.each(&:join)
# 20 concurrent operations share 5 connections safely
```
