# Chapter 05 — Concurrency & Threading: Follow-Up Traps

---

## Trap 1 — MRI GVL means threads don't parallelize CPU work

```ruby
# COMMON MISCONCEPTION: "I'll use 4 threads to make my computation 4x faster"

def cpu_work(n)
  n.times { |i| i * i }
end

# Single-threaded
cpu_work(10_000_000)  # baseline

# 4 threads — same speed (or slightly slower due to thread overhead)
4.times.map { Thread.new { cpu_work(10_000_000 / 4) } }.each(&:join)
# NOT 4x faster — GVL allows only one thread to run Ruby code at a time

# The only true parallelism options in MRI Ruby:
# 1. Process.fork (separate OS processes, each with own GVL)
# 2. Ractor (Ruby 3.0+ — each has own GVL-like mechanism)
# 3. C extensions that explicitly release the GVL

# When threads DO help:
# - Waiting for database queries (GVL released)
# - HTTP requests (GVL released)
# - File I/O (GVL released)
# - sleep (GVL released)
```

---

## Trap 2 — Thread#join vs Thread#value: exceptions

```ruby
# join re-raises exceptions; value also re-raises AND returns the result

t = Thread.new { raise "oops" }

# join: waits and re-raises
t.join   # => RuntimeError: oops

# value: waits, returns result if OK, re-raises if exception
t = Thread.new { "hello" }
t.value  # => "hello"

t = Thread.new { raise "oops" }
t.value  # => RuntimeError: oops

# Trap: thread exceptions are SILENTLY ignored if you don't join/value!
t = Thread.new { raise "silent death" }
sleep 0.1
# No error shown! The exception happened and nobody caught it.

# Safe pattern: always join or value your threads
# OR: Thread.abort_on_exception = true  (crash main on thread exception)
Thread.abort_on_exception = true

t = Thread.new { raise "now everyone knows" }
sleep 0.1
# => RuntimeError: now everyone knows (main thread crashes)
```

---

## Trap 3 — Mutex#synchronize is not reentrant

```ruby
mutex = Mutex.new

# Thread that tries to lock itself:
mutex.synchronize do
  puts "outer lock acquired"
  mutex.synchronize do    # DEADLOCK! thread tries to lock what it already holds
    puts "inner lock"
  end
end
# => ThreadError: deadlock; recursive locking

# This happens in real code when:
# 1. Method A locks mutex, calls Method B
# 2. Method B also tries to lock the same mutex
# 3. DEADLOCK

class Database
  def initialize
    @mutex = Mutex.new
  end

  def transaction(&block)
    @mutex.synchronize { block.call }
  end

  def batch_insert(records)
    records.each { |r| insert(r) }  # each insert calls transaction!
  end

  def insert(record)
    transaction { write(record) }   # if batch_insert uses transaction too: DEADLOCK
  end
end

# Fix: use Monitor (reentrant) or restructure to avoid nested locks
require 'monitor'
@mutex = Monitor.new   # drop-in replacement that allows recursive locking
```

---

## Trap 4 — Fiber is cooperative — you must call resume explicitly

```ruby
fiber = Fiber.new do
  puts "started"
  Fiber.yield
  puts "resumed"
end

# The fiber does NOT run until you call resume
puts "before first resume"
fiber.resume              # prints "started", then pauses at Fiber.yield
puts "between resumes"
fiber.resume              # prints "resumed", fiber is now dead
puts "after all resumes"

# Output:
# before first resume
# started
# between resumes
# resumed
# after all resumes

# Trap: calling resume on a dead fiber
fiber.resume  # => FiberError: dead fiber called

# Check if alive:
fiber.alive?  # => false after it returns

# Fibers do NOT run in parallel — they're cooperative in a single thread
# Fiber.yield suspends; resume continues
# This is fundamentally different from threads (preemptive)
```

---

## Trap 5 — Ractor restricts sharing mutable objects (Ractor::IsolationError)

```ruby
# Anything captured from the outer scope by a Ractor must be shareable

# WRONG: capturing mutable variable from outer scope
config = { timeout: 30 }  # mutable hash

r = Ractor.new do
  config[:timeout]  # Ractor::IsolationError!
end

# FIX 1: pass as argument (moves the object)
r = Ractor.new(config) { |c| c[:timeout] }
# WARNING: config is now inaccessible to main ractor (moved)

# FIX 2: freeze (allow sharing)
config.freeze
r = Ractor.new { config[:timeout] }  # frozen is shareable

# FIX 3: use only immutable types
r = Ractor.new { 30 }  # integers are always shareable

# What is shareable?
Ractor.shareable?(42)            # => true (Integer)
Ractor.shareable?(:symbol)       # => true (Symbol)
Ractor.shareable?("frozen".freeze)  # => true (frozen String)
Ractor.shareable?("mutable")     # => false
Ractor.shareable?({}.freeze)     # => false (frozen Hash with mutable content inside)
Ractor.shareable?({}.freeze.tap { |h| Ractor.make_shareable(h) })  # => true
```

---

## Trap 6 — fork + threads is dangerous

```ruby
# After fork, only the forking thread survives in the child process
# ALL OTHER THREADS ARE KILLED in the child

mutex = Mutex.new

# Thread holding a mutex
bg_thread = Thread.new do
  mutex.lock
  sleep 10   # holding the lock for a while
end

sleep 0.1   # let bg_thread get the lock

pid = Process.fork do
  # In the child: bg_thread is GONE, but mutex is still locked!
  mutex.lock   # DEADLOCK — mutex is locked by a thread that no longer exists
  puts "child: doing work"
end

Process.wait(pid)
# child process deadlocks forever

# Real-world impact: Unicorn/Puma web servers fork after initializing
# Any background threads or locked mutexes cause issues in child processes

# Best practice: don't start background threads before forking
# Use after_fork hooks to reinitialize shared resources
```

---

## Trap 7 — Thread local variables leak between requests

```ruby
# Thread-local variables persist for the LIFE of the thread
# In a thread pool, threads are reused across requests

# DANGEROUS: setting thread-local without cleanup
class RequestMiddleware
  def call(env)
    Thread.current[:request_id] = env["HTTP_X_REQUEST_ID"]
    @app.call(env)
    # no cleanup!
  end
end

# Thread is returned to the pool with :request_id still set
# Next request using this thread sees the previous request's :request_id!

# SAFE: always clean up thread locals
class RequestMiddleware
  def call(env)
    Thread.current[:request_id] = env["HTTP_X_REQUEST_ID"]
    @app.call(env)
  ensure
    Thread.current[:request_id] = nil  # clean up in ensure
  end
end
```

---

## Trap 8 — Queue#pop blocks forever if producers finish without signaling

```ruby
queue = Queue.new

producer = Thread.new do
  5.times { |i| queue << i }
  # Forgot to send termination signal!
end

consumer = Thread.new do
  loop do
    item = queue.pop   # blocks after 5 items — forever!
    puts item
  end
end

producer.join
consumer.join  # main thread hangs here forever

# Fix: use a sentinel/poison pill
producer = Thread.new do
  5.times { |i| queue << i }
  queue << nil   # sentinel value
end

consumer = Thread.new do
  loop do
    item = queue.pop
    break if item.nil?   # stop on sentinel
    puts item
  end
end

producer.join
consumer.join  # exits cleanly

# Or use Queue#close (Ruby 2.3+):
producer = Thread.new do
  5.times { |i| queue << i }
  queue.close   # signals "no more items"
end

consumer = Thread.new do
  while item = queue.pop
    puts item
  end
  # queue.pop returns nil when queue is closed and empty
end
```

---

## Trap 9 — Ractor.new passes by move, not copy

```ruby
data = [1, 2, 3, 4, 5]

r = Ractor.new(data) { |arr| arr.sum }
r.take   # => 15

# TRAP: data is now inaccessible to the main Ractor!
data.inspect  # => Ractor::MovedError: moved object cannot be used

# Objects passed to Ractor.new (as arguments) are MOVED, not copied
# The main ractor loses ownership

# Fix 1: pass a copy
r = Ractor.new(data.dup) { |arr| arr.sum }
r.take     # => 15
data.first # => 1  (data still accessible)

# Fix 2: freeze before passing (frozen objects are shared, not moved)
data.freeze
r = Ractor.new(data) { |arr| arr.sum }
r.take     # => 15
data.first # => 1  (still accessible because it was shared, not moved)
```

---

## Trap 10 — Thread safety in class-level state

```ruby
# Class-level instance variables are NOT thread-safe by default

class RequestCounter
  @count = 0   # class instance variable

  def self.increment
    @count += 1   # READ then WRITE — race condition!
  end

  def self.count
    @count
  end
end

# 1000 concurrent requests:
threads = 1000.times.map { Thread.new { RequestCounter.increment } }
threads.each(&:join)
RequestCounter.count  # might be 987, 993, etc. — not 1000

# Fix: use Mutex
class RequestCounter
  @count = 0
  @mutex = Mutex.new

  def self.increment
    @mutex.synchronize { @count += 1 }
  end

  def self.count
    @mutex.synchronize { @count }
  end
end

# Or use concurrent-ruby:
require 'concurrent'
class RequestCounter
  @count = Concurrent::AtomicFixnum.new(0)

  def self.increment
    @count.increment
  end

  def self.count
    @count.value
  end
end
```

---

## Trap 11 — SizedQueue vs Queue for backpressure

```ruby
# Queue grows unboundedly — producer can overwhelm consumer
queue = Queue.new
producer = Thread.new { 1_000_000.times { |i| queue << i } }  # no brakes!

# SizedQueue blocks producer when full — provides backpressure
queue = SizedQueue.new(100)  # at most 100 items
producer = Thread.new { 1_000_000.times { |i| queue << i } }
# producer automatically pauses when queue is full

# This prevents OOM errors when producer is faster than consumer
```

---

## Trap 12 — respond_to? is not always thread-safe

```ruby
# Checking respond_to? then calling is a TOCTOU race condition
# (Time of Check, Time of Use)

def process(obj)
  # UNSAFE: another thread could modify obj between check and use
  if obj.respond_to?(:process)
    obj.process   # obj might have changed!
  end
end

# Safe: duck-type inside synchronized block, or rescue
def safe_process(obj)
  obj.process
rescue NoMethodError
  # handle missing method
end

# Or with synchronization if shared mutable object:
@mutex.synchronize do
  obj.process if obj.respond_to?(:process)
end
```
