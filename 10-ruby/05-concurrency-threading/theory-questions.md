# Chapter 05 — Concurrency & Threading: Theory Questions

## The GVL (Global VM Lock)

**Q1. What is the GVL (Global VM Lock) in MRI Ruby?**

The GVL (also called GIL — Global Interpreter Lock) is a mutex inside the MRI Ruby runtime that prevents more than one thread from executing Ruby bytecode at the same time.

```ruby
# Even with multiple threads, only ONE runs Ruby code at a time
threads = 4.times.map do |i|
  Thread.new { loop { i * i } }  # CPU-bound: threads take turns
end

# MRI threads do NOT achieve true parallelism for CPU-bound work
# The GVL ensures at most one thread runs Ruby instructions at a time

# For CPU-bound work: GVL means no speedup from multiple threads
require 'benchmark'

def cpu_work
  1_000_000.times { |n| n * n }
end

Benchmark.bm do |b|
  b.report("1 thread:") { cpu_work }
  b.report("4 threads:") {
    4.times.map { Thread.new { cpu_work } }.each(&:join)
  }
end
# 1 thread:  ~0.15s
# 4 threads: ~0.15s  (no speedup — GVL!)
```

**Q2. When does the GVL get released?**

```ruby
# The GVL IS released during blocking I/O operations
# This means Ruby threads ARE useful for I/O-bound concurrency

# Scenario: fetching 5 URLs concurrently
require 'net/http'
require 'benchmark'

urls = [
  "http://httpbin.org/delay/1",
  "http://httpbin.org/delay/1",
  "http://httpbin.org/delay/1"
]

Benchmark.bm do |b|
  b.report("sequential:") {
    urls.each { |url| Net::HTTP.get(URI(url)) }
  }
  b.report("threaded:") {
    urls.map { |url|
      Thread.new { Net::HTTP.get(URI(url)) }
    }.each(&:join)
  }
end
# sequential: ~3.0s
# threaded:   ~1.0s  (GVL released during I/O! threads overlap)

# GVL is released for:
# - File I/O (read, write)
# - Network I/O (HTTP, database, sockets)
# - sleep
# - C extensions that explicitly release it (e.g., pg gem)
```

---

## Thread Class

**Q3. How do you create and manage threads in Ruby?**

```ruby
# Thread.new — creates and immediately starts a thread
t = Thread.new do
  puts "Thread started"
  sleep 0.5
  "thread result"   # last expression is thread value
end

puts "Main thread continues"
value = t.value    # blocks until thread finishes, returns result
                   # also re-raises exceptions from the thread

# Thread.join — wait for thread without getting value
t = Thread.new { sleep 1 }
t.join   # main thread blocks here until t completes

# Multiple threads
threads = (1..5).map do |i|
  Thread.new do
    sleep rand(0.1..0.5)
    "Thread #{i} done"
  end
end

results = threads.map(&:value)
puts results.inspect
```

**Q4. How do you handle exceptions in threads?**

```ruby
# By default, exceptions in threads are silently swallowed
Thread.new {
  raise "Something went wrong"
}.join
# => RuntimeError: Something went wrong  (re-raised on join)

# Thread.value also re-raises
t = Thread.new { 1 / 0 }
t.value  # => ZeroDivisionError: divided by 0

# Thread.abort_on_exception — crash the main thread
Thread.abort_on_exception = true
Thread.new { raise "Fatal thread error" }
# Now the main process exits immediately

# Per-thread setting:
t = Thread.new { raise "error" }
t.abort_on_exception = true

# Rescue inside the thread (safest):
t = Thread.new do
  begin
    risky_operation
  rescue => e
    log_error(e)
    nil  # return nil as thread's value
  end
end
```

---

## Thread Safety: Mutex

**Q5. What is a Mutex and how do you use it?**

```ruby
# Mutex — mutual exclusion lock
# Only one thread can hold it at a time

counter = 0
mutex   = Mutex.new

# UNSAFE — race condition
threads = 100.times.map do
  Thread.new do
    100.times { counter += 1 }  # read, increment, write = not atomic!
  end
end
threads.each(&:join)
puts counter  # WRONG: might be 9843, 9917, etc. instead of 10000

# SAFE — mutex protects critical section
counter = 0
mutex   = Mutex.new

threads = 100.times.map do
  Thread.new do
    100.times do
      mutex.synchronize { counter += 1 }  # atomic
    end
  end
end
threads.each(&:join)
puts counter  # CORRECT: always 10000
```

**Q6. What is the difference between Mutex#lock, Mutex#unlock, and Mutex#synchronize?**

```ruby
mutex = Mutex.new

# Manual lock/unlock (use with caution — easy to forget unlock)
mutex.lock
begin
  # critical section
  counter += 1
ensure
  mutex.unlock  # MUST unlock in ensure!
end

# synchronize — automatically locks and unlocks (preferred)
mutex.synchronize do
  counter += 1  # always unlocked, even if exception raised
end

# try_lock — non-blocking attempt
if mutex.try_lock
  begin
    # got the lock
  ensure
    mutex.unlock
  end
else
  # someone else has the lock — do something else
end

# Mutex is NOT reentrant — calling lock twice in the same thread deadlocks!
mutex = Mutex.new
mutex.synchronize do
  mutex.synchronize { }  # => ThreadError: deadlock (thread already owns lock)
end

# Use Monitor for reentrant locking:
require 'monitor'
monitor = Monitor.new
monitor.synchronize do
  monitor.synchronize { }  # works — reentrant
end
```

---

## Thread Local Variables

**Q7. How do thread-local variables work in Ruby?**

```ruby
# Thread[] accessor — per-thread key/value storage
Thread.current[:user_id] = 42
Thread.current[:user_id]    # => 42

# Each thread has its own storage
t1 = Thread.new do
  Thread.current[:name] = "Thread 1"
  sleep 0.1
  puts Thread.current[:name]   # => "Thread 1"
end

t2 = Thread.new do
  Thread.current[:name] = "Thread 2"
  sleep 0.1
  puts Thread.current[:name]   # => "Thread 2"
end

t1.join; t2.join

# Practical use: per-request context in web servers
class ApplicationController
  before_action :set_current_user

  private

  def set_current_user
    Thread.current[:current_user] = User.find(session[:user_id])
  end
end

# Access from anywhere in the request:
class AuditLog
  def self.record(action)
    user = Thread.current[:current_user]
    puts "#{user&.name} performed: #{action}"
  end
end
```

---

## Fiber: Cooperative Concurrency

**Q8. What is a Fiber and how does it differ from a Thread?**

```ruby
# Fiber — cooperative, manually scheduled coroutine
# No parallelism — runs in the current thread, you control scheduling

fiber = Fiber.new do
  puts "Fiber: step 1"
  Fiber.yield "result from step 1"  # pause, return value to caller

  puts "Fiber: step 2"
  Fiber.yield "result from step 2"

  puts "Fiber: step 3"
  "final result"  # last value is returned by final resume
end

puts fiber.resume  # => "Fiber: step 1", returns "result from step 1"
puts fiber.resume  # => "Fiber: step 2", returns "result from step 2"
puts fiber.resume  # => "Fiber: step 3", returns "final result"
fiber.resume       # => FiberError: dead fiber called

# Fibers are useful for:
# 1. Generators (infinite sequences)
# 2. Coroutines / cooperative concurrency
# 3. Async I/O (Ruby 3.x fiber scheduler)
```

**Q9. How do you implement a generator using Fibers?**

```ruby
# Fibonacci generator using Fiber
fib_fiber = Fiber.new do
  a, b = 0, 1
  loop do
    Fiber.yield a
    a, b = b, a + b
  end
end

10.times { print "#{fib_fiber.resume} " }
# => 0 1 1 2 3 5 8 13 21 34

# Compared to Enumerator (which uses Fibers internally):
fib_enum = Enumerator.new do |y|
  a, b = 0, 1
  loop do
    y << a
    a, b = b, a + b
  end
end

fib_enum.take(10)  # => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

---

## Ractor: True Parallelism (Ruby 3.0+)

**Q10. What is a Ractor and how does it differ from a Thread?**

```ruby
# Ractor — actor-model based parallelism, NO shared mutable state
# Multiple Ractors run truly in parallel (each has its own GVL)

# Simple Ractor example:
ractor = Ractor.new do
  "Hello from Ractor!"
end
puts ractor.take   # => "Hello from Ractor!"

# CPU-bound parallelism with Ractors:
def cpu_intensive(n)
  (1..n).inject(:+)
end

results = 4.times.map do |i|
  Ractor.new(i) { |idx| cpu_intensive(1_000_000) }
end.map(&:take)

# This DOES use multiple cores! Unlike threads with GVL.
```

**Q11. What are Ractor isolation rules?**

```ruby
# Ractors CANNOT share mutable objects (Ractor::IsolationError)
shared_array = [1, 2, 3]

Ractor.new { shared_array << 4 }
# => Ractor::IsolationError: can not access non-shareable objects by non-main Ractor

# Objects that CAN be shared between Ractors:
# - Frozen objects
# - Integers, Symbols, nil, true, false (always immutable)
# - Immutable objects (numbers, frozen strings, frozen arrays)

SHARED_CONFIG = { timeout: 30, retries: 3 }.freeze

r = Ractor.new { SHARED_CONFIG[:timeout] }  # works — frozen
r.take   # => 30

# Sending objects MOVES or COPIES them:
r = Ractor.new do
  data = Ractor.receive   # blocks until main ractor sends
  data.sum
end

r.send([1, 2, 3, 4, 5])   # sends and moves the array (main ractor loses access)
puts r.take                # => 15
```

---

## Process.fork

**Q12. How does fork provide true parallelism and what are its limitations?**

```ruby
# fork creates a child process — full copy of parent's memory
# True parallelism — completely separate GVL
pid = Process.fork do
  puts "Child process: #{Process.pid}"
  # CPU-intensive work here
  exit
end

puts "Parent process: #{Process.pid}"
Process.wait(pid)   # wait for child to finish

# Practical: parallel processing
results_pipe = IO.pipe  # communication channel

pids = (1..4).map do |i|
  Process.fork do
    result = compute_something(i)
    results_pipe[1].write("#{result}\n")
    results_pipe[1].close
    exit
  end
end

results_pipe[1].close  # parent closes write end
pids.each { |pid| Process.wait(pid) }

results = results_pipe[0].read.split("\n").map(&:to_i)

# Limitations:
# - fork is expensive (copies entire process memory — CoW helps but still slow)
# - fork + threads is dangerous: only the forking thread survives in the child
# - not available on Windows
# - database connections must be re-established after fork
```

---

## Race Conditions and Deadlocks

**Q13. What is a race condition and how do you prevent it?**

```ruby
# Race condition: outcome depends on thread execution order

# Example: bank account without synchronization
class UnsafeBankAccount
  attr_reader :balance

  def initialize(balance)
    @balance = balance
  end

  def withdraw(amount)
    if @balance >= amount         # ← thread 1 reads balance=100
                                  # ← THREAD SWITCH HERE
      @balance -= amount          # ← thread 1 withdraws 100
    end                           # ← thread 2 also read 100, also withdraws!
  end
end

account = UnsafeBankAccount.new(100)
threads = 10.times.map { Thread.new { account.withdraw(20) } }
threads.each(&:join)
# balance might be -100! (both threads saw balance=100 before either withdrew)

# Fix: use Mutex
class SafeBankAccount
  def initialize(balance)
    @balance = balance
    @mutex   = Mutex.new
  end

  def withdraw(amount)
    @mutex.synchronize do
      raise "Insufficient funds" if @balance < amount
      @balance -= amount
    end
  end

  def balance
    @mutex.synchronize { @balance }
  end
end
```

**Q14. What is a deadlock?**

```ruby
# Deadlock: two threads each hold a lock the other needs

mutex_a = Mutex.new
mutex_b = Mutex.new

# Thread 1: locks A, then tries to lock B
t1 = Thread.new do
  mutex_a.synchronize do
    sleep 0.01  # force context switch
    mutex_b.synchronize { puts "Thread 1 has both" }
  end
end

# Thread 2: locks B, then tries to lock A
t2 = Thread.new do
  mutex_b.synchronize do
    sleep 0.01  # force context switch
    mutex_a.synchronize { puts "Thread 2 has both" }
  end
end

t1.join
t2.join
# Both threads wait forever — deadlock!

# Prevention strategies:
# 1. Always acquire locks in the same ORDER across all threads
# 2. Use try_lock with timeout and retry
# 3. Minimize lock scope (hold as briefly as possible)
# 4. Use higher-level abstractions (Monitor, Queue, etc.)
```

---

## Queue for Thread-Safe Communication

**Q15. How does Queue enable safe producer-consumer patterns?**

```ruby
require 'thread'

# Queue is thread-safe — no mutex needed for basic operations
queue = Queue.new

# Producer thread
producer = Thread.new do
  10.times do |i|
    queue << "item_#{i}"
    sleep 0.1
  end
  queue << :done  # sentinel value
end

# Consumer thread
consumer = Thread.new do
  loop do
    item = queue.pop   # blocks if queue is empty (cooperative wait)
    break if item == :done
    puts "Processed: #{item}"
  end
end

producer.join
consumer.join

# Multiple consumers:
NUM_WORKERS = 4
queue = Queue.new

workers = NUM_WORKERS.times.map do
  Thread.new do
    loop do
      job = queue.pop
      break if job == :poison_pill
      process_job(job)
    end
  end
end

# Enqueue work
100.times { |i| queue << i }

# Send poison pills to stop workers
NUM_WORKERS.times { queue << :poison_pill }

workers.each(&:join)
```

---

## concurrent-ruby gem

**Q16. What does the concurrent-ruby gem provide?**

```ruby
require 'concurrent'

# Atomic reference — thread-safe single value
counter = Concurrent::AtomicFixnum.new(0)
threads = 100.times.map do
  Thread.new { counter.increment }
end
threads.each(&:join)
puts counter.value   # => 100 (always correct)

# Future — asynchronous computation
future = Concurrent::Future.execute { expensive_computation }
# runs in background threadpool
puts future.value   # blocks until result available (or raises exception)

# Promise — chainable async operations
result = Concurrent::Promise
  .execute { fetch_user(id) }
  .then    { |user| fetch_permissions(user) }
  .then    { |perms| perms.map(&:name) }

result.value  # => ["read", "write"]

# Thread-safe data structures
hash = Concurrent::Hash.new
array = Concurrent::Array.new
```
