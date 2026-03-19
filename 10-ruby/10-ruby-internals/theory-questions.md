# Chapter 10: Ruby Internals — Theory Questions

## YARV and the Ruby VM

**Q1. What is YARV and how does it differ from the original Ruby interpreter?**

YARV (Yet Another Ruby VM) is the bytecode virtual machine that powers MRI (Matz's Ruby Interpreter) since Ruby 1.9. Before YARV, Ruby used a tree-walking interpreter that directly evaluated the AST on every execution.

```ruby
# Ruby source code compilation pipeline (Ruby 3.3):
# Source → Prism Parser (new) → AST → Instruction Sequences (YARV bytecode) → YARV VM

# View YARV bytecode with RubyVM::InstructionSequence:
code = RubyVM::InstructionSequence.compile(<<~RUBY)
  def add(a, b)
    a + b
  end
  add(1, 2)
RUBY

puts code.disasm
# == disasm: #<ISeq:<compiled>@<compiled>:1 (1,0)-(5,0)>
# 0000 definemethod :add, <ISeq:add@<compiled>:1 (1,0)-(3,1)>  ( 1)[Li]
# 0003 putself
# 0004 putobject 1
# 0006 putobject 2
# 0008 opt_send_without_block <calldata!mid:add, argc:1>
# 0010 leave

# YARV advantages over tree-walking interpreter:
# 1. Bytecode is more compact than AST
# 2. Bytecode can be analyzed for optimization (YJIT operates on bytecode)
# 3. Faster dispatch (VM loop vs recursive tree evaluation)
# 4. Allows techniques like constant folding, inlining
```

---

**Q2. What is the Ruby object model? How are `RObject`, `RClass`, and `RString` structured in C?**

Every Ruby object is a C structure on the heap. The main types are:

```ruby
# Conceptual representation of Ruby's C structures:

# RBasic — common header for ALL Ruby objects:
# struct RBasic {
#   VALUE flags;  // type tag, frozen flag, GC mark bits
#   VALUE klass;  // pointer to class object
# };

# RObject — for regular user-defined objects:
# struct RObject {
#   struct RBasic basic;  // klass, flags
#   uint32_t numiv;       // number of instance variables
#   VALUE *ivptr;         // pointer to instance variable array
# };

# RClass — for class and module objects:
# struct RClass {
#   struct RBasic basic;
#   rb_classext_t *ptr;   // methods, constants, instance variables, superclass
# };

# RString — for string objects:
# struct RString {
#   struct RBasic basic;
#   union {
#     struct { long len; char *ptr; long capa; } heap;  // heap-allocated
#     char ary[RSTRING_EMBED_LEN_MAX + 1];               // short string embed
#   } as;
# };

# In Ruby code, you can observe this structure through:
obj = Object.new
obj.instance_variable_set(:@x, 1)
puts ObjectSpace.memsize_of(obj)  # => ~40 bytes (RObject + 1 ivar)

class Small; end
s = Small.new
puts s.instance_variables.size  # => 0

s.instance_variable_set(:@a, 1)
puts ObjectSpace.memsize_of(s)  # slightly larger after adding ivar
```

---

**Q3. Explain Ruby's tri-color mark-and-sweep GC. What are the three colors?**

```ruby
# Tri-color incremental GC (Ruby 2.2+):
# Objects are colored to track GC progress:
#
# WHITE: not yet visited by GC (potentially garbage)
# GREY:  visited but children not yet scanned
# BLACK: fully processed (definitely reachable, children also scanned)
#
# At start of GC: all objects are WHITE
# GC roots (stack variables, globals, constants): marked GREY
# GC processes GREY objects:
#   - Colors the GREY object BLACK
#   - Colors all referenced children GREY
# After processing all GREY: only WHITE (garbage) and BLACK (live) remain
# WHITE objects are freed

# Why tri-color instead of bi-color?
# Bi-color (mark/unmark) requires stopping the world while marking
# Tri-color allows INCREMENTAL GC:
#   - GC marks some GREY objects, then yields back to Ruby code
#   - Ruby code runs, potentially creating new objects (marked WHITE)
#   - GC continues marking
#   - Issue: if Ruby code creates a reference from BLACK → new WHITE object,
#     that object would be freed! → WRITE BARRIER prevents this

# Write barrier: when a BLACK object gets a new reference to a WHITE object,
# it "notches" the black object back to GREY so it's re-scanned

# Ruby objects that use write barriers:
# T_ARRAY, T_HASH, T_OBJECT (user objects) — these are WB-protected
# T_STRING, T_BIGNUM — NOT WB-protected (treated specially)

# Observable in GC.stat:
GC.stat[:minor_gc_count]  # incremental minor GCs
GC.stat[:major_gc_count]  # full major GCs
```

---

**Q4. What are Object Shapes in Ruby 3.2+? How do they optimize instance variable access?**

```ruby
# Before Object Shapes:
# Instance variables were stored in a hash-like table
# Looking up @name required hashing the symbol and searching the table
# Each object could have ANY combination of instance variables → hard to optimize

# Object Shapes (Ruby 3.2+):
# Objects that have the same sequence of instance variable ASSIGNMENTS
# share the same "shape" — a unique identifier
# The shape encodes: which ivars, in what order they were set

class User
  def initialize(name, email)
    @name = name     # shape transition: {} → {:@name}
    @email = email   # shape transition: {:@name} → {:@name, :@email}
  end
end

# All User instances follow the SAME shape sequence
# → YJIT can specialize: "if shape=X, @name is at index 0, @email at index 1"
# → Direct array index access (no hash lookup!)

# Observable shape effects:
user1 = User.new("Alice", "a@b.com")
user2 = User.new("Bob",   "b@b.com")
# Both have same shape → YJIT can use same compiled specialization for both

# Shape divergence (bad for performance):
class FlexUser
  def set_field(name, val)
    instance_variable_set(:"@#{name}", val)  # dynamic ivars → many shapes
  end
end

f1 = FlexUser.new; f1.set_field(:x, 1); f1.set_field(:y, 2)
f2 = FlexUser.new; f2.set_field(:y, 2); f2.set_field(:x, 1)  # different order!
# f1 and f2 have DIFFERENT shapes (different assignment order) → YJIT can't specialize

# Best practice: always set the same ivars in the same order in initialize
```

---

**Q5. What is `$LOAD_PATH` and how does `require` use it?**

```ruby
# $LOAD_PATH (also accessible as $:) is an array of directories
# Ruby searches for .rb and .so files when require is called

puts $LOAD_PATH
# => [
#   "/home/user/.rbenv/versions/3.2.0/lib/ruby/gems/3.2.0/gems/bundler-2.4.10/lib",
#   "/home/user/.rbenv/versions/3.2.0/lib/ruby/site_ruby/3.2.0",
#   "/home/user/.rbenv/versions/3.2.0/lib/ruby/3.2.0",
#   "/home/user/.rbenv/versions/3.2.0/lib/ruby/3.2.0/x86_64-linux"
# ]

# require 'json' search process:
# For each path in $LOAD_PATH:
#   check path/json.rb → found! → load it, return true
# If not found → LoadError: cannot load such file -- json

# require_relative uses file's directory, not $LOAD_PATH:
require_relative 'my_module'  # loads ./my_module.rb relative to current file

# Adding to load path (common in gems and test setups):
$LOAD_PATH.unshift(File.join(__dir__, 'lib'))
# or:
$LOAD_PATH.prepend(File.join(__dir__, 'lib'))

# $LOADED_FEATURES ($") tracks what's already been loaded:
puts $LOADED_FEATURES.grep(/json/)
# => ["/usr/lib/ruby/3.2.0/json.rb", ...]

# require vs require_relative vs load:
# require:          searches $LOAD_PATH, caches in $LOADED_FEATURES
# require_relative: relative to current file, also cached
# load:             always reloads (useful in Rails dev mode, console reloading)
```

---

**Q6. How does Ruby handle string encoding? What is the `# encoding:` pragma?**

```ruby
# Default encoding: Ruby 2.0+ defaults to UTF-8
# Ruby 1.9: each file has its own encoding, default ASCII

# Declare file encoding:
# encoding: UTF-8  (at top of file, alternative comment styles)
# -*- coding: utf-8 -*-

# Check encoding:
"hello".encoding  # => #<Encoding:UTF-8>
"hello".encode("ISO-8859-1").encoding  # => #<Encoding:ISO-8859-1>

# Common encoding operations:
str = "Héllo"
str.encoding          # => UTF-8
str.bytes.to_a        # => [72, 195, 169, 108, 108, 111]  (UTF-8 bytes)
str.length            # => 6  (character count, not byte count)
str.bytesize          # => 7  (byte count — é is 2 bytes in UTF-8)

# Encoding conversion:
utf8  = "Héllo"
latin = utf8.encode("ISO-8859-1")
back  = latin.encode("UTF-8")

# Handling encoding errors:
# Replace invalid/undefined characters:
str.encode("ASCII", invalid: :replace, undef: :replace, replace: "?")
# => "H?llo"

# Force encoding (change the label without converting):
binary = "\xC3\xA9"
binary.force_encoding("UTF-8")
binary.valid_encoding?  # => true

# Encoding::UTF_8 constant:
"hello".encode(Encoding::UTF_8)

# Practical: reading files with specific encoding
File.read("data.csv", encoding: "ISO-8859-1")
File.open("data.csv", "r:ISO-8859-1:UTF-8")  # read as ISO, convert to UTF-8
```

---

**Q7. What is the difference between `Comparable`, `Enumerable`, and their C-level implementations?**

```ruby
# Comparable and Enumerable are modules implemented in C for performance
# They provide rich interfaces based on a single method you define

# Comparable:
# Requires: define <=> (spaceship operator)
# Provides: <, >, <=, >=, ==, between?, clamp

class Temperature
  include Comparable

  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  def <=>(other)
    degrees <=> other.degrees  # the ONLY method you must define
  end
end

temps = [Temperature.new(30), Temperature.new(10), Temperature.new(20)]
temps.sort            # uses <=> to sort
temps.min             # uses <=>
temps.max             # uses <=>
Temperature.new(25).between?(Temperature.new(20), Temperature.new(30))  # => true
Temperature.new(35).clamp(Temperature.new(0), Temperature.new(40))      # works

# Enumerable:
# Requires: define each (yield elements)
# Provides: map, select, reject, find, reduce, count, min, max, sort, group_by, etc.

class NumberRange
  include Enumerable

  def initialize(start, stop)
    @start = start
    @stop  = stop
  end

  def each  # the ONLY method you must define
    current = @start
    while current <= @stop
      yield current
      current += 1
    end
  end
end

range = NumberRange.new(1, 10)
range.map { |n| n * 2 }                    # [2,4,6,8,10,12,14,16,18,20]
range.select { |n| n.even? }               # [2,4,6,8,10]
range.reduce(:+)                            # 55
range.group_by { |n| n % 3 }              # {1=>[1,4,7,10], 2=>[2,5,8], 0=>[3,6,9]}
range.min_by { |n| -n }                    # 10
range.lazy.select { |n| n > 5 }.first(3)  # [6,7,8]
```

---

**Q8. Explain the key changes from Ruby 2.7 to Ruby 3.0, 3.1, 3.2, and 3.3.**

```ruby
# Ruby 2.7 (2019):
# - Pattern matching (experimental): case/in
# - Numbered block parameters: _1, _2
# - Warning for keyword argument separation (preparing for 3.0 change)
# - Enumerable#tally
[1, 2, 1, 3, 2, 1].tally  # => {1=>3, 2=>2, 3=>1}

# Numbered params:
[1, 2, 3].map { _1 * 2 }    # => [2, 4, 6]
[[1, 2], [3, 4]].map { _1 + _2 }  # => [3, 7]

# Ruby 3.0 (2020): "3x faster than Ruby 2.0" goal
# - Keyword argument separation (BREAKING): positional and keyword args fully separated
def greet(name:); end
greet({name: "Alice"})  # Ruby 2.x: worked (hash auto-converted to kwargs)
greet({name: "Alice"})  # Ruby 3.0: ArgumentError! Must use: greet(name: "Alice")

# - Pattern matching (stable):
case { name: "Alice", age: 30 }
in { name: String => name, age: (18..) => age }
  "Adult: #{name}, #{age}"
end

# - Ractor (experimental parallelism)
# - Fiber::Scheduler for async I/O

# Ruby 3.1 (2021):
# - YJIT (production-ready JIT compiler)
# - error_highlight gem (shows exact token in error)
# - Hash shorthand: { x:, y: } instead of { x: x, y: y }
x, y = 1, 2
{ x:, y: }  # => { x: 1, y: 2 }

# - Endless method one-liner (stable)
def double(x) = x * 2

# Ruby 3.2 (2022):
# - Object Shapes (major ivar performance improvement)
# - Data class (immutable Struct):
Point = Data.define(:x, :y)
p = Point.new(x: 1, y: 2)
p.x       # => 1
p.frozen? # => true (Data instances are frozen)
p.with(x: 5)  # => Point(x: 5, y: 2)  (non-destructive update)

# - YJIT improvements (50-60% faster than 3.1 YJIT)
# - Production-ready Ractors
# - Regexp timeout to prevent ReDoS:
Regexp.timeout = 1.0  # regex that takes > 1s raises Regexp::TimeoutError

# Ruby 3.3 (2023):
# - Prism parser (new default parser, better errors)
# - YJIT improvements (15-25% faster)
# - RubyVM::YJIT.enable at runtime
# - Fiber improvements
# - Range#overlap? method
(1..5).overlap?(3..7)  # => true
```

---

**Q9. Explain Ruby's pattern matching introduced in 3.0. What is `deconstruct` and `deconstruct_keys`?**

```ruby
# Pattern matching uses case/in (not case/when)

# 1. Value patterns:
case 42
in Integer => n if n > 0
  "positive integer: #{n}"
in String
  "it's a string"
end

# 2. Array patterns:
case [1, 2, 3]
in [Integer => first, *rest]
  "first: #{first}, rest: #{rest}"
end

# 3. Hash patterns:
case { name: "Alice", role: :admin }
in { name: String => name, role: :admin }
  "Admin user: #{name}"
in { name: String => name }
  "Regular user: #{name}"
end

# 4. Find pattern (Ruby 3.0+):
case [1, 2, 3, "hello", 4, 5]
in [*, String => s, *]
  "Found string: #{s}"
end

# deconstruct — for array-like pattern matching on custom objects:
class Point
  attr_reader :x, :y

  def initialize(x, y)
    @x = x
    @y = y
  end

  def deconstruct
    [@x, @y]  # allows: case point in [x, y]
  end

  def deconstruct_keys(keys)
    # keys is nil (all) or an array of requested keys
    h = { x: @x, y: @y }
    keys ? h.slice(*keys) : h  # allows: case point in {x:, y:}
  end
end

point = Point.new(1, 2)

case point
in [x, y]
  "Array pattern: #{x}, #{y}"
end

case point
in { x: 0, y: }
  "On y-axis: #{y}"
in { x:, y: }
  "Point at #{x}, #{y}"
end

# Pinning operator (^) — match against a variable's value:
expected = 42
case some_value
in ^expected  # matches only if some_value == 42
  "matched expected"
end

# One-line pattern matching (Ruby 3.1+):
{ name: "Alice" } => { name: }  # deconstruct into local variable `name`
[1, 2, 3] => [first, *]         # deconstruct first element
```

---

**Q10. What are Ractors and how do they achieve true parallelism in Ruby?**

```ruby
# MRI Ruby has the GIL (Global Interpreter Lock) / GVL (Global VM Lock)
# Only ONE thread can execute Ruby code at a time
# Threads are concurrent but NOT parallel for CPU-bound work

# Ractors (Ruby 3.0+) bypass the GIL:
# Each Ractor has its OWN GVL
# Multiple Ractors can run ON MULTIPLE CPU CORES simultaneously

# Simple parallel computation:
numbers = (1..1_000_000).to_a.freeze  # must be frozen/shareable

r1 = Ractor.new(numbers[0..499_999]) do |nums|
  nums.sum { |n| n * n }
end

r2 = Ractor.new(numbers[500_000..]) do |nums|
  nums.sum { |n| n * n }
end

total = r1.take + r2.take  # wait for both Ractors to finish
# Each Ractor runs on a separate thread → true parallel execution

# Ractor isolation rules:
# - Only SHAREABLE objects can be passed between Ractors:
#   → Frozen objects (Ractor.make_shareable or .freeze)
#   → Immutable types: Integer, Float, Symbol, true, false, nil
#   → Ractor-created objects

# - NOT shareable:
#   → Mutable objects (regular Hash, Array, etc.)
#   → Objects from most standard library classes

# Sending messages between Ractors:
producer = Ractor.new do
  5.times do |i|
    Ractor.yield i * 10  # sends value to any receiver
  end
end

5.times do
  puts producer.take  # receive from producer: 0, 10, 20, 30, 40
end

# Ractor with input channel:
worker = Ractor.new do
  loop do
    work = Ractor.receive  # wait for work
    break if work == :stop
    Ractor.yield work * 2
  end
end

worker.send(5)
worker.take   # => 10
worker.send(:stop)
```

---

**Q11. What are numbered block parameters (`_1`, `_2`) and what are their limitations?**

```ruby
# Ruby 2.7+: numbered parameters
# _1 = first block argument
# _2 = second block argument, etc.

# Instead of:
[1, 2, 3].map { |n| n * 2 }

# You can write:
[1, 2, 3].map { _1 * 2 }  # => [2, 4, 6]

# With two arguments:
[[1, 2], [3, 4]].map { _1 + _2 }  # => [3, 7]

# With hash iteration:
{ a: 1, b: 2 }.map { "#{_1}: #{_2}" }  # => ["a: 1", "b: 2"]

# LIMITATIONS:
# 1. Cannot combine with explicit block parameters
[1, 2, 3].map { |n| _1 * n }  # SyntaxError: numbered parameter used in outer block

# 2. Cannot use in nested blocks:
[1, 2].map do
  [3, 4].map { _1 * _1 }  # _1 refers to inner block's arg, outer is inaccessible
end

# 3. Only _1 through _9 are available (more than 9 args: use explicit params)

# When to use:
# Good: short one-line blocks where the parameter name adds no clarity
[1, 2, 3].map { _1 ** 2 }
events.sort_by { _1[:timestamp] }
users.select { _1.active? }

# Bad: longer blocks, multiple params with meaning
users.map do |user|  # explicit name 'user' is clearer
  "#{user.name} (#{user.email})"
end
```

---

**Q12. Explain Ruby's `Kernel`, `BasicObject`, and `Object` hierarchy.**

```ruby
# Hierarchy (bottom to top):
# BasicObject  ← root, ~8 methods, no Kernel
#   └── Object  ← BasicObject + Kernel module
#         └── [all user-defined classes by default]

BasicObject.superclass   # => nil (no parent)
Object.superclass        # => BasicObject
String.superclass        # => Object

# BasicObject — minimal, ~8 methods:
BasicObject.instance_methods
# => [:equal?, :!, :==, :__id__, :__send__, :instance_eval, :instance_exec, :!=]
# Intentionally MISSING: nil?, is_a?, class, respond_to?, send, object_id

# Object adds Kernel's methods:
Object.include?(Kernel)   # => true
# Kernel provides: puts, print, p, require, require_relative,
#                  raise, loop, proc, lambda, rand, exit, etc.

# Kernel methods are available everywhere:
puts "hello"  # Kernel#puts
require 'json' # Kernel#require
rand           # Kernel#rand

# When to subclass BasicObject:
# - Blank-slate objects for method_missing proxies
# - DSL builders where you don't want Object's methods to interfere
# - Objects that delegate EVERYTHING to a target

class BlankSlate < BasicObject
  def initialize(target)
    @target = target
  end

  def method_missing(name, *args, &block)
    @target.__send__(name, *args, &block)
  end
end

# object_id still works on BasicObject instances (it's built-in):
blank = BlankSlate.new([1,2,3])
blank.length  # => 3 (delegated)

# Checking ancestors:
String.ancestors
# => [String, Comparable, Object, Kernel, BasicObject]
```

---

**Q13. What are the GIL (Global Interpreter Lock) and its effect on threads in MRI Ruby?**

```ruby
# The GIL (also called GVL - Global VM Lock) in MRI ensures
# only ONE thread executes Ruby bytecode at a time

# Threads ARE useful in Ruby for:
# - I/O-bound work (GIL released during blocking I/O)
# - Concurrency (multiple things in progress, not necessarily parallel)

require 'net/http'

# CONCURRENT (useful — threads released GIL during network wait):
threads = urls.map do |url|
  Thread.new { fetch(url) }  # each thread waits for I/O independently
end
results = threads.map(&:value)  # collected results

# NOT PARALLEL for CPU-bound work:
# Two CPU-intensive threads still take ~2x as long as one (GIL prevents parallelism)
t1 = Thread.new { 10_000_000.times { } }
t2 = Thread.new { 10_000_000.times { } }
t1.join; t2.join
# Duration: ~2x single-threaded (GIL serializes them)

# GIL is released for:
# - Blocking I/O (File.read, socket reads, HTTP calls)
# - sleep
# - C extensions that explicitly release it: rb_thread_blocking_region

# Achieving true CPU parallelism in Ruby:
# Option 1: Ractors (Ruby 3.0+) — experimental, limited gem support
# Option 2: Separate processes (fork, spawn)
# Option 3: JRuby or TruffleRuby (no GIL, true thread parallelism)

# Thread-safe patterns in MRI:
require 'mutex_m'

class ThreadSafeCounter
  def initialize
    @mutex = Mutex.new
    @count = 0
  end

  def increment
    @mutex.synchronize { @count += 1 }
  end

  def value
    @mutex.synchronize { @count }
  end
end
```

---

**Q14. What is `GC.compact` and object compaction?**

```ruby
# Ruby 2.7+: GC.compact

# Problem without compaction:
# GC frees objects, leaving "holes" (fragmentation) in the heap
# New objects fill random holes → bad cache locality
# The process's virtual memory stays large even after many GC runs

# GC.compact:
# - Moves live objects to be contiguous in memory
# - Updates all references to point to new locations
# - Allows heap to shrink (pages at the end can be returned to OS)
# - Improves cache locality → faster object access

# Usage:
GC.compact   # triggers compaction

# Results:
stats = GC.compact
puts stats
# => { considered: 100000, moved: 45000 }
# 45000 objects moved to compact the heap

# When to call:
# - After a large batch job completes
# - After loading many classes (boot time)
# - In Puma's on_worker_boot hook (Rails):
# puma.rb:
on_worker_boot do
  GC.compact
end

# Copy-on-write friendly forking (Ruby 3.2+):
# Before forking (after loading the app), compact the heap:
# This means worker processes share more memory pages with the master
GC.compact
workers = 4.times.map { fork { run_worker } }

# After compaction, ObjectSpace shows moved objects:
GC.latest_compact_info
# => { considered: {...}, moved: {...} }
```

---

**Q15. Explain Ruby's Fiber and how the Fiber scheduler enables non-blocking I/O.**

```ruby
# Fibers are cooperative, lightweight concurrency primitives
# Unlike threads: Fibers yield control explicitly (non-preemptive)

fiber = Fiber.new do
  puts "Step 1"
  Fiber.yield  # suspend this fiber, return control
  puts "Step 2"
  Fiber.yield
  puts "Step 3"
end

fiber.resume  # "Step 1"
fiber.resume  # "Step 2"
fiber.resume  # "Step 3"

# Fibers with values:
counter = Fiber.new do
  i = 0
  loop do
    Fiber.yield(i)  # yield a value
    i += 1
  end
end

counter.resume  # => 0
counter.resume  # => 1
counter.resume  # => 2

# Fiber Scheduler (Ruby 3.0+) — for non-blocking I/O:
# The scheduler allows blocking I/O operations to yield the Fiber
# instead of blocking the entire thread

require 'async'  # implements Fiber::Scheduler

Async do |task|
  # These run CONCURRENTLY on a single thread via Fiber switching:
  task.async { fetch_data("https://api1.example.com") }
  task.async { fetch_data("https://api2.example.com") }
  task.async { fetch_data("https://api3.example.com") }
end
# When one Fiber waits for I/O, the scheduler runs another Fiber

# Standard library hooks (Ruby 3.0+):
# TCPSocket, IO, Mutex — all Fiber scheduler-aware
# When a non-blocking socket would block, it yields the current Fiber

# Practical: 1000 concurrent HTTP requests on one thread
Async do
  1000.times.map do |i|
    Async { HTTP.get("https://api.example.com/item/#{i}") }
  end.map(&:wait)
end
```

---

**Q16. What is the difference between `Kernel#eval`, `Binding#eval`, and `BasicObject#instance_eval`?**

```ruby
# eval — execute arbitrary Ruby code in a string
# With optional binding argument to set the context
eval("1 + 1")              # => 2
eval("x * 2", binding)    # uses current binding (x must be in scope)

# Binding#eval — evaluate in a specific captured binding
def make_binding(x, y)
  binding  # captures x, y from this scope
end

b = make_binding(10, 20)
b.eval("x + y")  # => 30 (accesses x and y from the captured scope)

# instance_eval — evaluate in the context of a specific object
class Config
  def initialize
    @db_host = "localhost"
    @db_port = 5432
  end
end

config = Config.new
config.instance_eval { @db_host }   # => "localhost" (accesses ivar directly)
config.instance_eval { @db_port = 9999 }  # sets ivar on config

# class_eval (module_eval) — evaluate as if inside a class body
String.class_eval do
  def shout
    upcase + "!!!"
  end
end
"hello".shout  # => "HELLO!!!"

# Security note:
# eval with user input is extremely dangerous
user_input = "system('rm -rf /')"
eval(user_input)  # NEVER DO THIS

# Performance note:
# eval is slow (parses and compiles a new string each call)
# instance_eval with a block is fast (block is compiled once)
```

---

**Q17. How does Ruby implement `method_missing` at the C level?**

```ruby
# At the C level (simplified):
#
# 1. rb_call0() tries to find the method in the method table
# 2. If not found → calls rb_method_missing()
# 3. rb_method_missing() looks for 'method_missing' in the class hierarchy
# 4. If method_missing is overridden → calls it with (method_name, *args)
# 5. If method_missing is NOT overridden → raises NoMethodError

# You can observe the performance cost:
require 'benchmark/ips'

class WithMM
  def method_missing(name, *args)
    "handled: #{name}"
  end
end

class WithReal
  define_method(:dynamic_method) { "handled" }
end

obj_mm   = WithMM.new
obj_real = WithReal.new

Benchmark.ips do |x|
  x.report("method_missing") { obj_mm.dynamic_method }
  x.report("real method")    { obj_real.dynamic_method }
  x.compare!
end
# method_missing: 2-10x slower (full lookup chain + method_missing dispatch)
# real method:    normal speed (direct method table lookup)

# The "ghostbusting" optimization: define the method on first miss
def method_missing(name, *args)
  if valid_method?(name)
    self.class.define_method(name) { |*a| handle(name, *a) }
    send(name, *args)  # call the newly defined real method
  else
    super
  end
end
# First call: method_missing overhead
# Subsequent calls: direct method table lookup (fast)
```

---

**Q18. What is `autoload` and how does it differ from `require`?**

```ruby
# autoload: defers loading until the constant is first referenced
# require: loads immediately (or records as already loaded)

# Without autoload: all files loaded at boot
require 'heavy_parser'    # loads immediately, even if never used
require 'pdf_generator'   # loads immediately

# With autoload: only loaded when first used
autoload :HeavyParser,   'heavy_parser'    # registered but NOT loaded
autoload :PdfGenerator,  'pdf_generator'   # registered but NOT loaded

# First time HeavyParser is referenced:
HeavyParser.new  # NOW heavy_parser.rb is loaded

# Class-level autoload (common in gems):
module MyGem
  autoload :Parser,    'my_gem/parser'
  autoload :Generator, 'my_gem/generator'
  autoload :Formatter, 'my_gem/formatter'
end

# Loading is triggered by:
MyGem::Parser.new       # loads my_gem/parser.rb
MyGem::Generator.parse  # loads my_gem/generator.rb
# MyGem::Formatter never loaded if never referenced!

# Rails uses Zeitwerk for autoloading (more sophisticated than autoload):
# - Maps file paths to constants automatically
# - Supports reloading in development mode
# - Thread-safe

# Ruby's autoload vs Zeitwerk:
# Ruby autoload: built-in, simple, NOT reloadable
# Zeitwerk: gem, convention-based, supports reload for development
```

---

**Q19. What are Ruby's frozen string literals and how do they interact with `dup`, `clone`, and `+@`?**

```ruby
# frozen_string_literal: true

str = "hello"
str.frozen?  # => true

# dup creates an unfrozen copy:
copy = str.dup
copy.frozen?  # => false
copy << " world"  # works!
copy  # => "hello world"

# clone preserves the frozen state:
clone = str.clone
clone.frozen?  # => true
clone << " world"  # => FrozenError!

# +@ (unary plus) creates an unfrozen copy (alias for String#-dup):
mutable = +"hello"  # or: "hello".dup
mutable.frozen?     # => false
mutable << " world"  # works!

# -@ (unary minus) returns a frozen, deduplicated string:
frozen1 = -"hello"
frozen2 = -"hello"
frozen1.equal?(frozen2)   # => true (same interned object!)
frozen1.frozen?           # => true

# Use cases:
# Building strings from frozen literals:
PREFIX = "User: ".freeze

def build_display_name(name)
  result = +PREFIX  # unfrozen copy of frozen constant
  result << name    # mutate safely
  result
end

# Or just use interpolation (which creates new unfrozen string):
def build_display_name(name)
  "User: #{name}"  # creates new string, avoids FrozenError
end
```

---

**Q20. How does Ruby implement `Comparable`'s `clamp` and why is it a common interview question for custom objects?**

```ruby
# Comparable#clamp clamps a value to a range [min, max]
# Requires: <=> method defined

class Version
  include Comparable

  attr_reader :major, :minor, :patch

  def initialize(version_string)
    parts = version_string.split('.').map(&:to_i)
    @major = parts[0] || 0
    @minor = parts[1] || 0
    @patch = parts[2] || 0
  end

  def <=>(other)
    return major <=> other.major unless major == other.major
    return minor <=> other.minor unless minor == other.minor
    patch <=> other.patch
  end

  def to_s
    "#{major}.#{minor}.#{patch}"
  end
end

v = Version.new("2.5.0")
min_v = Version.new("1.0.0")
max_v = Version.new("3.0.0")

v.clamp(min_v, max_v)             # => Version 2.5.0 (within range)
Version.new("0.1.0").clamp(min_v, max_v)  # => Version 1.0.0 (clamped to min)
Version.new("5.0.0").clamp(min_v, max_v)  # => Version 3.0.0 (clamped to max)

# Clamp with Range (Ruby 2.7+):
v.clamp(min_v..max_v)  # range form

# This works because Comparable#clamp is implemented using <=>, < and >
# which are all derived from your <=> implementation

# The interview trap: forgetting <=> must handle nil gracefully:
class FlexVersion
  include Comparable

  def <=>(other)
    return nil unless other.is_a?(Version)  # return nil for incomparable types
    # ...
  end
end
```
