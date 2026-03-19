# Chapter 05 — Concurrency & Threading: Scenario Questions

## Scenario 1 — Will this code always print 100?

```ruby
counter = 0
threads = 100.times.map do
  Thread.new { counter += 1 }
end
threads.each(&:join)
puts counter
```

**Answer:** No. `counter += 1` is not atomic — it's a read-modify-write operation. In MRI Ruby, the GVL might protect against *some* races, but `counter += 1` compiles to multiple bytecode instructions. Two threads can both read `counter` before either writes back. In practice on MRI, this often "works" for small numbers, but it's undefined behavior and will fail under load or on alternative Ruby implementations. The correct fix:

```ruby
mutex = Mutex.new
counter = 0
threads = 100.times.map do
  Thread.new { mutex.synchronize { counter += 1 } }
end
threads.each(&:join)
puts counter  # always 100
```

---

## Scenario 2 — Thread#join vs Thread#value

```ruby
t = Thread.new do
  sleep 0.1
  raise "something went wrong"
end

# What's the difference?
t.join   # ?
t.value  # ?
```

**Answer:** Both `join` and `value` wait for the thread to complete. The difference:
- `join` returns the thread object and re-raises exceptions from the thread
- `value` returns the thread's return value (last expression evaluated) and also re-raises exceptions

```ruby
# If thread raises, both join and value re-raise:
t = Thread.new { raise "error" }
t.join   # => RuntimeError: error  (re-raised in calling thread)

t = Thread.new { raise "error" }
t.value  # => RuntimeError: error  (also re-raised)

# Normal thread:
t = Thread.new { "hello" }
t.join   # => returns the Thread object (not the value)
t.value  # => "hello"  (returns the actual computed value)
```

---

## Scenario 3 — Is this Mutex usage correct?

```ruby
class Cache
  def initialize
    @data  = {}
    @mutex = Mutex.new
  end

  def fetch(key)
    @mutex.synchronize do
      unless @data.key?(key)
        @mutex.synchronize do  # is this OK?
          @data[key] = compute(key)
        end
      end
      @data[key]
    end
  end
end
```

**Answer:** No, this will deadlock. `Mutex` in Ruby is NOT reentrant. The outer `synchronize` holds the lock; the inner `synchronize` tries to acquire the same lock in the same thread — deadlock.

```ruby
# Fix 1: Remove the inner synchronize
def fetch(key)
  @mutex.synchronize do
    @data[key] ||= compute(key)
  end
end

# Fix 2: Use Monitor (reentrant mutex)
require 'monitor'

class Cache
  def initialize
    @data    = {}
    @monitor = Monitor.new
  end

  def fetch(key)
    @monitor.synchronize do
      @monitor.synchronize do   # reentrant — works!
        @data[key] ||= compute(key)
      end
    end
  end
end
```

---

## Scenario 4 — Fiber-based producer/consumer

```ruby
producer = Fiber.new do
  5.times do |i|
    Fiber.yield "item_#{i}"
  end
  :done
end

loop do
  item = producer.resume
  break if item == :done
  puts "Consuming: #{item}"
end
```

**Answer:**
```
Consuming: item_0
Consuming: item_1
Consuming: item_2
Consuming: item_3
Consuming: item_4
```

This is a cooperative concurrency pattern. The fiber produces items one at a time, yielding control back to the main loop each time. No threads, no locks needed — single-threaded but structured as two "coroutines."

---

## Scenario 5 — GVL: CPU vs I/O bound threads

```ruby
# Task A: Make 10 HTTP requests
# Task B: Compute 10 fibonacci numbers

# Which benefits more from threading in MRI Ruby?
require 'net/http'

def compute_fib(n)
  n <= 1 ? n : compute_fib(n-1) + compute_fib(n-2)
end

def fetch_url(url)
  Net::HTTP.get(URI(url))
end
```

**Answer:** HTTP requests (I/O-bound) benefit from threading because the GVL is released during I/O. CPU-bound Fibonacci gets NO benefit — even with multiple threads, only one runs Ruby code at a time.

```ruby
# I/O-bound: threads give ~10x speedup for 10 concurrent requests
urls = 10.times.map { "http://example.com" }
threads = urls.map { |url| Thread.new { fetch_url(url) } }
results = threads.map(&:value)  # ~1s instead of ~10s

# CPU-bound: threads give ~1x speedup (no improvement, maybe slightly worse)
threads = 10.times.map { Thread.new { compute_fib(35) } }
threads.each(&:join)  # same time as sequential
```

---

## Scenario 6 — Thread-safe singleton

```ruby
class Config
  @@instance = nil

  def self.instance
    @@instance ||= new
  end

  private_class_method :new
end
```

**Is this thread-safe?**

**Answer:** No. `@@instance ||= new` is a race condition. Two threads could both evaluate `@@instance` as `nil` simultaneously and both call `new`.

```ruby
# Fix: double-checked locking pattern
class Config
  @@instance = nil
  @@mutex    = Mutex.new

  def self.instance
    return @@instance if @@instance  # fast path (no lock)
    @@mutex.synchronize do
      @@instance ||= new             # slow path (with lock)
    end
  end

  private_class_method :new
end
```

---

## Scenario 7 — Ractor isolation error

```ruby
data = [1, 2, 3, 4, 5]

r = Ractor.new(data) do |arr|
  arr.sum
end

puts r.take
```

**Answer:** This actually works in Ruby 3.0+ because passing an object to Ractor.new MOVES it (the main Ractor loses access). Let's check a true isolation error:

```ruby
data = [1, 2, 3, 4, 5]

r = Ractor.new do
  data.sum   # data NOT passed as argument — captured from outer scope
end

r.take  # => Ractor::IsolationError: can not access non-shareable objects in closure
```

The fix:
```ruby
data = [1, 2, 3, 4, 5]

# Pass as argument (moves the array):
r = Ractor.new(data) { |arr| arr.sum }
r.take  # => 15

# Or freeze to allow sharing:
data = [1, 2, 3, 4, 5].freeze
r = Ractor.new { data.sum }  # frozen data can be read from any Ractor
r.take  # => 15
```

---

## Scenario 8 — Thread pool with Queue

```ruby
# Build a simple thread pool that processes a work queue

class ThreadPool
  def initialize(size)
    @queue   = Queue.new
    @workers = size.times.map do
      Thread.new do
        loop do
          job = @queue.pop
          break if job == :shutdown
          job.call
        end
      end
    end
  end

  def submit(&job)
    @queue << job
  end

  def shutdown
    @workers.size.times { @queue << :shutdown }
    @workers.each(&:join)
  end
end

pool = ThreadPool.new(4)
results = []
mutex = Mutex.new

10.times do |i|
  pool.submit do
    result = i * i
    mutex.synchronize { results << result }
  end
end

pool.shutdown
puts results.sort.inspect
```

**Answer:** `[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]`. The pool processes jobs from the queue using 4 threads. Results are collected into a shared array protected by a mutex.

---

## Scenario 9 — Fiber scheduler for async I/O

```ruby
# Ruby 3.1+ Fiber Scheduler enables non-blocking I/O without threads

require 'async'

Async do
  responses = 5.times.map do |i|
    Async do
      # simulated async HTTP call
      Async::HTTP::Internet.get("http://httpbin.org/delay/1")
    end
  end
  responses.map(&:wait)
end
```

**Answer:** This pattern (using the `async` gem) enables concurrent I/O on a single thread using the Fiber scheduler. Unlike threads (which use the GVL), fibers cooperatively yield during I/O. All 5 requests run concurrently, completing in ~1s instead of ~5s, using zero threads.

---

## Scenario 10 — fork + reconnect database

```ruby
# Problem: database connections are NOT safe to share across fork

class DatabaseWorker
  def initialize
    @db = Database.connect  # establishes connection
  end

  def process_in_fork
    # What must happen here?
    pid = Process.fork do
      # @db is inherited from parent — DANGEROUS!
      # The child shares the socket/file descriptor
      @db.execute("SELECT 1")  # might work, might corrupt
    end
    Process.wait(pid)
  end
end
```

**Answer:** After `fork`, the child process must NOT use the parent's database connection. Both parent and child would share the same OS socket, causing corruption.

```ruby
def process_in_fork
  pid = Process.fork do
    # Disconnect from parent's connection
    @db.disconnect

    # Establish a FRESH connection in the child
    @db = Database.connect

    @db.execute("SELECT 1")

    @db.disconnect
    exit
  end
  Process.wait(pid)
end
```

This is why gems like Unicorn (Rails web server) call `after_fork` hooks to reconnect database pools after forking worker processes.

---

## Scenario 11 — Deadlock detection

```ruby
mutex1 = Mutex.new
mutex2 = Mutex.new

t1 = Thread.new do
  mutex1.synchronize do
    puts "T1 has mutex1, waiting for mutex2..."
    sleep 0.1  # give T2 time to grab mutex2
    mutex2.synchronize { puts "T1 has both!" }
  end
end

t2 = Thread.new do
  mutex2.synchronize do
    puts "T2 has mutex2, waiting for mutex1..."
    sleep 0.1  # give T1 time to grab mutex1
    mutex1.synchronize { puts "T2 has both!" }
  end
end

t1.join
t2.join
```

**Answer:** This code deadlocks. Neither T1 nor T2 can proceed:
- T1 holds mutex1, waits for mutex2
- T2 holds mutex2, waits for mutex1

Ruby will eventually raise `fatal: deadlock detected` (after the main thread detects all threads are blocked). The fix: always acquire locks in the same order.

```ruby
# Fix: both threads acquire in the same order (mutex1 then mutex2)
t1 = Thread.new do
  mutex1.synchronize { mutex2.synchronize { puts "T1 done" } }
end

t2 = Thread.new do
  mutex1.synchronize { mutex2.synchronize { puts "T2 done" } }
end
```

---

## Scenario 12 — Thread.current for request context

```ruby
class RequestContext
  def self.set_user(user)
    Thread.current[:current_user] = user
  end

  def self.current_user
    Thread.current[:current_user]
  end

  def self.clear
    Thread.current[:current_user] = nil
  end
end

# In a web framework (e.g., middleware):
class AuthMiddleware
  def call(env)
    user = authenticate(env)
    RequestContext.set_user(user)
    @app.call(env)
  ensure
    RequestContext.clear  # always clean up!
  end
end

# In a service anywhere in the request:
class AuditService
  def log(action)
    user = RequestContext.current_user
    puts "User #{user&.id} performed: #{action}"
  end
end
```

**Answer:** This works correctly because each HTTP request is handled by one thread (in threaded servers like Puma), and `Thread.current` provides isolated per-thread storage. The `ensure` block is critical — without it, the user from request N would leak into request N+1 if the thread is reused.

---

## Scenario 13 — Ractor parallel computation

```ruby
# Use Ractors to compute squares in parallel (Ruby 3.0+)

inputs = (1..8).to_a

ractors = inputs.map do |n|
  Ractor.new(n) { |x| x * x }
end

results = ractors.map(&:take)
puts results.inspect
```

**Answer:** `[1, 4, 9, 16, 25, 36, 49, 64]`. This is true CPU parallelism — all Ractors run simultaneously on multiple cores. Unlike threads (limited by GVL), Ractors achieve actual speedup for CPU-bound work on multi-core machines.

---

## Scenario 14 — Fiber.yield vs Thread communication

```ruby
# Consumer-producer with Fiber (single-threaded)
def each_page
  Fiber.new do
    page = 1
    loop do
      records = database_query(page: page, per_page: 100)
      break if records.empty?
      records.each { |r| Fiber.yield r }
      page += 1
    end
  end
end

fiber = each_page
loop do
  record = fiber.resume
  break if record.nil?
  process(record)
end
```

**Answer:** This is an implementation of external iteration using Fibers. The fiber fetches database pages lazily, yielding one record at a time. The consumer processes records without knowing about pagination. This is how Enumerator is implemented internally in Ruby.

---

## Scenario 15 — Choosing between Thread, Fiber, and Ractor

You're building a web scraper that needs to fetch 1000 URLs and is then CPU-bound processing the HTML. What concurrency model would you choose?

**Answer:**

```ruby
# Phase 1: I/O-bound (fetching URLs) — use Threads
require 'thread'
require 'net/http'

url_queue    = Queue.new
result_queue = Queue.new

1000.times { |i| url_queue << "http://example.com/page/#{i}" }

# 20 threads for I/O-bound fetching (GVL released during I/O)
fetchers = 20.times.map do
  Thread.new do
    loop do
      url = url_queue.pop(true) rescue nil
      break unless url
      result_queue << Net::HTTP.get(URI(url))
    end
  end
end

fetchers.each(&:join)

# Phase 2: CPU-bound (parsing HTML) — use Ractors or fork
htmls = result_queue.size.times.map { result_queue.pop }

# Ruby 3.0+: Ractors for CPU-bound work
ractors = htmls.map do |html|
  Ractor.new(html) { |h| parse_html(h) }
end

results = ractors.map(&:take)
```

The key insight: I/O-bound work → Threads (GVL released for I/O, less overhead than processes). CPU-bound work → Ractors or Processes (break free of GVL).
