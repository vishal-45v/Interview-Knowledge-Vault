# Chapter 10: Ruby Internals — Structured Answers

## Answer 1: Ruby's Complete Execution Pipeline

**Question:** "Walk me through what happens when Ruby executes a `.rb` file, from source text to running code."

**Answer:**

```ruby
# STEP 1: Lexing (tokenization)
# Source: "def greet(name)\n  puts \"Hello, #{name}!\"\nend"
# Tokens: [:kDEF, :tIDENTIFIER "greet", "(", :tIDENTIFIER "name", ")",
#           :tNL, :tIDENTIFIER "puts", :tSTRING_BEG, ...]

# STEP 2: Parsing (Ruby 3.3+: Prism parser)
# Tokens → AST (Abstract Syntax Tree)
ast = RubyVM::AbstractSyntaxTree.parse(<<~RUBY)
  def greet(name)
    puts "Hello, #{'"'}#{'}name}!"
  end
RUBY
# AST: SCOPE → DEFN :greet → args:[name] → body:[CALL puts, [DSTR...]]

# STEP 3: Compilation to YARV bytecode
iseq = RubyVM::InstructionSequence.compile(<<~RUBY)
  def greet(name)
    puts "Hello, \#{name}!"
  end
RUBY
puts iseq.disasm
# == disasm: #<ISeq:<compiled>@<compiled>:1>
# 0000 definemethod :greet, <ISeq:greet>
# 0003 putobject :greet
# 0005 leave

# STEP 4: YARV VM executes bytecodes
# The VM maintains:
# - Program counter (which instruction is next)
# - Stack (VALUE array for passing operands)
# - Local table (named variables)
# - Control frame stack (call stack)

# STEP 5: YJIT (if enabled) compiles hot bytecode to native
# After a method is called ~threshold times:
# YARV bytecodes → LLVM-like IR → native x86_64/arm64 machine code
# Stored in a "code cache" — subsequent calls go directly to native code

# TIMELINE:
# .rb file → [Prism] → AST → [compile] → ISeq (bytecode) → [YARV] → execution
#                                                      → [YJIT] → native code → execution
```

---

## Answer 2: Deep Dive into Ruby GC — How the Tri-Color Algorithm Works

**Question:** "Explain the Ruby GC in detail. How does the write barrier prevent bugs in incremental GC?"

**Answer:**

```ruby
# The fundamental GC problem: find all "live" objects and free the "dead" ones
# "Live" = reachable from any root (local vars, globals, constants, stack)
# "Dead" = not reachable = can be freed

# MARK PHASE (tri-color algorithm):
# Start: ALL objects are WHITE (potentially garbage)
# GC Roots → colored GREY (marked but children unscanned)

# Processing loop:
# Take one GREY object:
#   → Color it BLACK (fully processed)
#   → Color all its DIRECT REFERENCES GREY (to be processed)
# Repeat until no GREY objects remain

# Result:
# BLACK = definitely live (reachable from roots)
# WHITE = definitely dead (unreachable) → free them

# SWEEP PHASE:
# Walk the heap pages, collect WHITE objects' memory

# WHY TRI-COLOR ENABLES INCREMENTAL GC:
# After marking some objects, GC can PAUSE and let Ruby code run
# Then GC resumes where it left off

# THE WRITE BARRIER PROBLEM:
# What if Ruby code RUNS between GC increments and creates:
# BLACK_obj → new WHITE_obj (BLACK already processed → won't re-scan!)
# Result: WHITE_obj looks unreachable → FREED even though BLACK_obj holds reference!

# WRITE BARRIER SOLUTION:
# When a BLACK object gains a reference to a WHITE object:
# → Automatically "un-mark" the BLACK object back to GREY
# → Scheduler re-processes it, following the new reference
# → WHITE object is then also marked GREY → BLACK → survives GC

# Ruby C code for write barrier (simplified):
# void rb_gc_write_barrier(VALUE a, VALUE b) {
#   if (RB_FL_TEST(a, FL_WB_PROTECTED) && is_black(a) && is_white(b)) {
#     promote_to_grey(a);  // un-black the holder, re-schedule it
#   }
# }

# In Ruby code, you can observe GC behavior:
class Observer
  def initialize(ref)
    @ref = ref  # Write barrier protects this assignment
  end
end

# Force minor vs major GC:
GC.start(minor_only: true)   # minor GC only (young generation)
GC.start(minor_only: false)  # full GC (major)
GC.start                     # let Ruby decide

# Check if GC is running:
GC.count  # total GC count since start

# WB_PROTECTED objects (have write barriers): Hash, Array, Object
# NOT WB_PROTECTED: String, Float, Bignum (treated as shady objects)
puts RUBY_DESCRIPTION  # tells you the exact Ruby implementation
```

---

## Answer 3: Understanding `BasicObject`, `Object`, and the Kernel Mix-in

**Question:** "Why does Ruby have both `BasicObject` and `Object`? What's the design philosophy?"

**Answer:**

```ruby
# The DESIGN PHILOSOPHY:
# BasicObject = absolute minimum interface any Ruby object must have
# Object = BasicObject + Kernel = "normal" Ruby objects
# Kernel = utility methods mixed into every Object

# BasicObject's ~8 methods (intentionally minimal):
BasicObject.public_instance_methods
# => [:equal?, :!, :==, :instance_eval, :instance_exec, :__send__, :__id__, :!=]

# Notably ABSENT from BasicObject:
# - nil?       (in Object via Kernel)
# - class      (in Object)
# - respond_to? (in Object via Kernel)
# - is_a?      (in Object)
# - send       (Kernel#send)
# - object_id  (Kernel#object_id)

# WHY this split?

# 1. Proxy objects: you don't want proxy.class to return Proxy
#    You want proxy.class to delegate to the proxied object
class Proxy < BasicObject
  def initialize(target) = @target = target
  def method_missing(name, *args, &block) = @target.__send__(name, *args, &block)
  def class = @target.class  # override with delegation, not Object's version
end

arr_proxy = Proxy.new([1, 2, 3])
arr_proxy.class    # => Array (correctly delegates!)
arr_proxy.length   # => 3

# 2. DSL builders: with Object's methods, things like `puts`, `require`, `p`
#    are available. For a DSL, you may want a blank slate.
class DSLEvaluator < BasicObject
  def initialize(rules)
    @rules = rules
    @result = {}
  end

  def method_missing(name, *args)
    @result[name] = args.first
  end

  def evaluate(block)
    instance_eval(&block)
    @result
  end
end

# 3. The alternative: removing methods (bad pattern)
# Some old Ruby code would do:
# class BlankSlate < Object
#   instance_methods.each { |m| undef_method(m) unless m =~ /^__/ }
# end
# This is fragile and slow. BasicObject is the RIGHT way.

# Hierarchy visualization:
# BasicObject (the root — NO superclass)
#      │
#   Object (includes Kernel)
#      │
#  String, Array, Hash, Integer, ...
```

---

## Answer 4: Pattern Matching — Complete Feature Guide

**Question:** "Show me all the pattern matching syntax in Ruby 3.0-3.3 with practical examples."

**Answer:**

```ruby
# 1. LITERAL PATTERNS:
case value
in 42         then "exact integer"
in "hello"    then "exact string"
in true       then "exact boolean"
in nil        then "nil"
end

# 2. TYPE PATTERNS:
case value
in Integer    then "any integer"
in String     then "any string"
in NilClass   then "nil"
end

# 3. VARIABLE BINDING (capture):
case { name: "Alice", age: 30 }
in { name: String => name }   # bind match to variable `name`
  puts name  # => "Alice"
end

# 4. ARRAY PATTERNS:
case [1, 2, 3, 4, 5]
in [first, second, *rest]   # capture first, second, rest
  [first, second, rest]     # => [1, 2, [3, 4, 5]]
in [Integer, Integer, *]    # type check without capture
  "array of integers"
end

# 5. HASH PATTERNS:
case { status: 200, body: "ok", headers: {} }
in { status: 200, body: String => body }  # only required keys needed
  "Success: #{body}"
end

# 6. FIND PATTERN (Ruby 3.0+):
case [1, 2, "needle", 4, 5]
in [*, String => found, *]
  "Found: #{found}"  # => "Found: needle"
end

# 7. GUARD CLAUSES:
case temperature
in Integer => t if t > 100
  "boiling"
in Integer => t if t < 0
  "freezing"
in Integer => t
  "normal: #{t}"
end

# 8. PIN OPERATOR (^) — match against variable's value:
expected_status = 200
case response
in { status: ^expected_status }  # matches only if status == 200
  "Expected response"
end

# 9. DECONSTRUCT protocols for custom objects:
class Rectangle
  attr_reader :width, :height
  def initialize(w, h) = @width, @height = w, h
  def deconstruct         = [@width, @height]
  def deconstruct_keys(k) = { width: @width, height: @height }
end

rect = Rectangle.new(10, 5)
case rect
in [w, h] if w > h
  "landscape: #{w}x#{h}"
in { width:, height: } if width == height
  "square: #{width}"
end

# 10. RIGHTWARD ASSIGNMENT (Ruby 3.1+):
# Extracts variables in one line — raises NoMatchingPatternError if fails
{ name: "Alice", scores: [90, 85, 92] } => { name:, scores: [first, *] }
puts "#{name}'s first score: #{first}"  # name and first are now local vars

# 11. ONE-LINE in (returns true/false, Ruby 3.1+):
data = { role: :admin }
data in { role: :admin }  # => true (doesn't raise, returns bool)
data in { role: :viewer } # => false
```

---

## Answer 5: YARV Bytecode and YJIT — How JIT Compilation Works

**Question:** "Explain how YJIT compiles Ruby code to native machine code. What are the limitations?"

**Answer:**

```ruby
# YJIT is a method-based, lazy JIT compiler

# HOW IT WORKS:
# 1. Ruby code runs through YARV interpreter normally at first
# 2. YJIT monitors execution frequency and type information
# 3. When a method becomes "hot" (called many times):
#    → YJIT compiles the YARV bytecodes to native x86_64/arm64 code
#    → Stores compiled code in a "code cache"
#    → Future calls jump directly to compiled code

# 4. YJIT uses type specialization:
#    If it observes that `a + b` always receives Integers,
#    it compiles to: add rax, rbx  (single CPU instruction)
#    Instead of: full method dispatch → Integer#+ → type check → add

# VIEWING YJIT STATS:
if defined?(RubyVM::YJIT) && RubyVM::YJIT.enabled?
  stats = RubyVM::YJIT.runtime_stats

  puts "YJIT enabled: #{RubyVM::YJIT.enabled?}"
  puts "Code pages allocated: #{stats[:code_page_count]}"
  puts "Ratio in YJIT: #{(stats[:ratio_in_yjit] * 100).round(1)}%"
  puts "Exits (deopt): #{stats[:total_exit_count]}"
end

# DEOPTIMIZATION (exit from JIT):
# If YJIT's assumptions are wrong (unexpected type, method redefinition):
# → "side exit" back to the interpreter
# → YJIT may try to recompile with less aggressive assumptions

# LIMITATIONS:
# 1. Type instability causes deoptimizations:
def process(x)
  x * 2  # if x is sometimes Integer, sometimes Float → YJIT can't specialize
end
process(1)    # Integer
process(1.5)  # Float → different compiled path needed

# 2. Method redefinition invalidates compiled code:
Integer.class_eval { def +(other) = 0 }  # all Integer#+ JIT code invalidated!

# 3. method_missing — hard to JIT (unknown dispatch target):
class Proxy
  def method_missing(name, *args)
    target.send(name, *args)  # YJIT gives up here — too dynamic
  end
end

# 4. C extensions: YJIT can't compile inside C code, only the Ruby-level boundary

# 5. First N calls: always interpreted (JIT warmup)
# YJIT_EXEC_MEM_SIZE: max memory for compiled code (default 64MB)

# ENABLING AND TUNING:
# ruby --yjit app.rb
# RUBY_YJIT_EXEC_MEM_SIZE=128  # increase code cache (MB)

# Testing YJIT impact:
require 'benchmark/ips'

def hot_method(n)
  total = 0
  n.times { |i| total += i * i }
  total
end

# Warmup YJIT:
100.times { hot_method(100) }

Benchmark.ips do |x|
  x.config(warmup: 3, time: 5)
  x.report("hot_method") { hot_method(1000) }
end
```

---

## Answer 6: Ruby Version Feature Timeline

**Question:** "What are the most important Ruby language features added since 2.7? How do you use them?"

**Answer:**

```ruby
# RUBY 2.7 (2019):
# Pattern matching (experimental):
case [1, 2, 3]
in [Integer => a, *rest]
  "first: #{a}"
end

# Numbered parameters:
[1, 2, 3].map { _1 * 2 }

# Enumerable#tally:
["a", "b", "a", "c", "b", "a"].tally
# => {"a"=>3, "b"=>2, "c"=>1}

# Hash#transform_keys:
{ "name" => "Alice" }.transform_keys(&:to_sym)  # => { name: "Alice" }

# RUBY 3.0 (2020):
# Keyword argument separation (BREAKING):
def hello(name:); end
hello(name: "Alice")        # OK
hello(**{ name: "Alice" })  # OK
hello({ name: "Alice" })    # ArgumentError! (was OK in 2.x)

# Hash#except:
{ a: 1, b: 2, c: 3 }.except(:b)  # => { a: 1, c: 3 }

# Array#intersect?:
[1, 2, 3].intersect?([3, 4, 5])  # => true

# RUBY 3.1 (2021):
# YJIT (production-ready)
# Hash shorthand:
x, y = 1, 2
{ x:, y: }  # => { x: 1, y: 2 }  (instead of { x: x, y: y })

# Endless method (stable):
def double(x) = x * 2
def square(x) = x ** 2

# Refinements work in #include:
module M
  using SomeRefinement
  # refinement active in M and descendants
end

# Integer#ceildiv:
7.ceildiv(3)  # => 3  (ceiling division: Math.ceil(7.0/3))

# RUBY 3.2 (2022):
# Data class:
Point = Data.define(:x, :y)
p = Point.new(x: 1, y: 2)
p.frozen?  # => true

# Regexp.timeout (prevent ReDoS):
Regexp.timeout = 5.0
/^a*$/.match("a" * 1_000_000 + "!")  # raises after 5 seconds

# Object#frozen? for Integer, Float, Symbol (always true):
1.frozen?       # => true (was always frozen, now documented)
:foo.frozen?    # => true
nil.frozen?     # => true

# Array#intersect? (Ruby 3.1)
# String#byteindex, String#bytesplice (byte-level manipulation)

# RUBY 3.3 (2023):
# Prism parser (new default):
# - Better error messages (points to exact token)
# - 25-45% faster parsing

# YJIT: RubyVM::YJIT.enable  at runtime
RubyVM::YJIT.enable  # enable after boot (after loading gems)

# Range#overlap?:
(1..5).overlap?(3..7)   # => true
(1..5).overlap?(6..10)  # => false

# Fiber#kill:
fiber = Fiber.new { loop { Fiber.yield } }
fiber.resume
fiber.kill  # terminate the fiber

# ObjectSpace::WeakMap (improved) and WeakKeyMap
# Module#set_temporary_name for anonymous classes
klass = Class.new
klass.set_temporary_name("MyTemporaryClass")
klass.name  # => "MyTemporaryClass"
```

---

## Answer 7: Understanding `ObjectSpace` for Production Debugging

**Question:** "How do you use ObjectSpace to debug memory issues in a production-like setting safely?"

**Answer:**

```ruby
# SAFE PRODUCTION USE: aggregate statistics only (no iteration)
require 'objspace'

def memory_snapshot
  stats = GC.stat
  count_by_type = ObjectSpace.count_objects

  {
    heap_pages:        stats[:heap_allocated_pages],
    live_objects:      stats[:heap_live_slots],
    free_slots:        stats[:heap_free_slots],
    minor_gcs:         stats[:minor_gc_count],
    major_gcs:         stats[:major_gc_count],
    total_allocated:   stats[:total_allocated_objects],
    total_freed:       stats[:total_freed_objects],
    strings:           count_by_type[:T_STRING],
    arrays:            count_by_type[:T_ARRAY],
    hashes:            count_by_type[:T_HASH],
    objects:           count_by_type[:T_OBJECT],
    rss_kb:            `ps -o rss= -p #{Process.pid}`.to_i
  }
end

# Run snapshot before and after suspected leak:
before = memory_snapshot

# ... run 1000 requests ...

after = memory_snapshot
growth = after.merge(before) { |k, a, b| a - b }
         .reject { |_, v| v == 0 }

if growth[:strings] > 1000 || growth[:live_objects] > 5000
  Rails.logger.warn "Potential memory growth: #{growth}"
end

# UNSAFE IN PRODUCTION (causes GC pause, memory spike):
# ObjectSpace.each_object(String) { }  ← holds ALL strings in memory during iteration

# SAFE: count without iteration
ObjectSpace.count_objects[:T_STRING]

# SEMI-SAFE: use with care, only in staging/debug mode
if ENV['MEMORY_PROFILE']
  require 'memory_profiler'
  report = MemoryProfiler.report { process_batch(batch) }
  report.pretty_print(to_file: 'tmp/memory_profile.txt')
end

# PRODUCTION-SAFE LEAK DETECTION: track allocations in middleware
class AllocationTrackingMiddleware
  def initialize(app)
    @app = app
    @window = []
  end

  def call(env)
    before = GC.stat[:total_allocated_objects]
    result = @app.call(env)
    after  = GC.stat[:total_allocated_objects]

    allocated = after - before
    @window << allocated
    @window.shift if @window.size > 100

    avg = @window.sum / @window.size.to_f
    StatsD.gauge('ruby.allocations.per_request', allocated)
    StatsD.gauge('ruby.allocations.rolling_avg', avg)

    result
  end
end
```

---

## Answer 8: The Ruby Object Model — Eigenclasses and Method Dispatch

**Question:** "Explain the Ruby object model including eigenclasses, and trace exactly how `Dog.bark` is found."

**Answer:**

```ruby
# THE FULL RUBY OBJECT MODEL:

module Walkable
  def walk = "walking"
end

module Domestic
  def pet = "pets you"
end

class Animal
  def breathe = "breathing"
end

class Dog < Animal
  include Domestic
  extend Walkable  # adds to Dog's singleton class

  def bark = "woof"

  class << self
    def species = "Canis lupus familiaris"  # singleton method
  end
end

# When you call Dog.bark or Dog.new.bark, here's the ACTUAL lookup:

# DOG INSTANCE METHOD LOOKUP (dog.bark):
# 1. dog's singleton class (eigenclass)
# 2. Prepended modules of Dog (none in this example)
# 3. Dog class itself → finds :bark ✓

# DOG CLASS METHOD LOOKUP (Dog.species):
# Dog is an OBJECT — an instance of Class
# 1. Dog's singleton class → finds :species ✓
# (class << self methods are stored here)

# Where does Walkable go with extend?
# extend adds Walkable to Dog's SINGLETON CLASS as included module
# Dog's singleton class ancestors:
Dog.singleton_class.ancestors
# => [#<Class:Dog>, Walkable, #<Class:Animal>, #<Class:Object>,
#     #<Class:BasicObject>, Class, Module, Object, Kernel, BasicObject]

Dog.walk  # => "walking" (found via Walkable in singleton class ancestors)

# Full ancestor chain for dog.bark:
Dog.ancestors
# => [Dog, Domestic, Animal, Object, Kernel, BasicObject]

dog = Dog.new
dog.singleton_class.ancestors
# => [#<Class:dog_instance>, Dog, Domestic, Animal, Object, Kernel, BasicObject]

# VISUAL:
# [Dog's singleton class] ← Dog class methods live here
#       ↑ (superclass)
# [Dog]                   ← instance methods: bark
# [Domestic]              ← included module
#       ↑
# [Animal]
# [Object]
# [Kernel]
# [BasicObject]
```

---

## Answer 9: Ruby's Encoding System and Multibyte String Handling

**Question:** "How does Ruby handle multibyte strings? What are the common encoding pitfalls?"

**Answer:**

```ruby
# CORE CONCEPT: Ruby strings are sequences of BYTES with an encoding label
# The encoding tells Ruby how to interpret those bytes as characters

str = "Héllo"                 # UTF-8 string
str.encoding                  # => UTF-8
str.length                    # => 5 (characters)
str.bytesize                  # => 6 (bytes: é is 2 bytes in UTF-8)
str.bytes                     # => [72, 195, 169, 108, 108, 111]

# PITFALL 1: Length vs bytesize confusion
emoji = "Hello 👋"
emoji.length    # => 7 (7 characters, including emoji)
emoji.bytesize  # => 10 (emoji is 4 bytes in UTF-8)

# PITFALL 2: String slicing is character-based (correct):
"Héllo"[0..1]   # => "Hé" (first 2 CHARACTERS, not bytes)

# vs byte slicing:
"Héllo".byteslice(0, 2)  # => "H\xC3" (2 bytes — might be invalid UTF-8!)

# PITFALL 3: ASCII compatibility:
ascii = "hello".encode("ASCII")
utf8  = "hello".encode("UTF-8")
ascii == utf8   # => true (both contain the same ASCII characters)
ascii.encoding  # => ASCII
utf8.encoding   # => UTF-8

# PITFALL 4: Encoding errors during database reads:
# Database returns Latin-1 encoded data, Ruby assumes UTF-8
bad_string = "\xE9".force_encoding("UTF-8")  # ← wrong encoding
bad_string.valid_encoding?  # => false → will cause issues with JSON, puts, etc.

# Fix: specify correct encoding on read
correct = "\xE9".force_encoding("ISO-8859-1").encode("UTF-8")
# => "é" (properly converted)

# PITFALL 5: Binary strings and encoding:
binary = File.binread("image.png")  # reads as ASCII-8BIT (binary)
binary.encoding  # => ASCII-8BIT
binary[0..3]     # => first 4 bytes (might not be valid UTF-8)

# SAFE conversion pattern:
def safe_to_utf8(str)
  return str if str.encoding == Encoding::UTF_8 && str.valid_encoding?

  str.encode("UTF-8",
    invalid: :replace,
    undef:   :replace,
    replace: "?"
  )
end

# Database (PostgreSQL): always use client_encoding = 'UTF8' in database.yml
# HTTP: always set Content-Type: application/json; charset=utf-8
# File: use File.open(path, "r:UTF-8") or File.read(path, encoding: "UTF-8")
```

---

## Answer 10: The Impact of Ruby 3.x Changes on Real Applications

**Question:** "Which Ruby 3.x features have you used or seen provide real practical value in production?"

**Answer:**

```ruby
# 1. HASH SHORTHAND (Ruby 3.1) — reduces boilerplate significantly
# BEFORE:
def create_user(name, email, role)
  User.create(name: name, email: email, role: role)
end

# AFTER:
def create_user(name, email, role)
  User.create(name:, email:, role:)
end

# Also useful in specs:
describe "user creation" do
  let(:name)  { "Alice" }
  let(:email) { "alice@example.com" }

  it "creates" do
    result = UserService.create(name:, email:)  # concise
    expect(result).to be_a(User)
  end
end

# 2. PATTERN MATCHING FOR API RESPONSES (Ruby 3.0+)
# BEFORE: nested conditionals
def process_api_response(response)
  if response.is_a?(Hash) && response[:status] == 200 && response[:data].is_a?(Array)
    response[:data].map { |item| item[:name] if item.is_a?(Hash) }.compact
  elsif response.is_a?(Hash) && response[:status] == 401
    raise AuthError
  else
    []
  end
end

# AFTER: expressive and exhaustive
def process_api_response(response)
  case response
  in { status: 200, data: [*, { name: String }, *] => items }
    items.filter_map { _1[:name] }
  in { status: 401 }
    raise AuthError
  in { status: (500..) => status }
    raise ServerError, "Status #{status}"
  else
    []
  end
end

# 3. DATA CLASS FOR VALUE OBJECTS (Ruby 3.2+)
# BEFORE: Struct with manual freeze or separate freeze calls
Coordinate = Struct.new(:lat, :lon) do
  def initialize(*)
    super
    freeze
  end
end

# AFTER: Data is always frozen, cleaner
Coordinate = Data.define(:lat, :lon) do
  def to_s = "#{lat},#{lon}"
  def valid? = lat.between?(-90, 90) && lon.between?(-180, 180)
end

# 4. YJIT FOR CPU-HEAVY JOBS (Ruby 3.1+)
# Real gains in: report generation, data processing, template rendering
# Enable in production: ruby --yjit config.ru (Rack/Puma)
# Puma config:
if defined?(RubyVM::YJIT)
  on_worker_boot { RubyVM::YJIT.enable rescue nil }
end

# 5. Endless methods for simple value objects and one-liner utilities
class Product
  attr_reader :name, :price, :category

  def initialize(name:, price:, category:)
    @name = name; @price = price; @category = category
  end

  def discounted_price(pct) = (price * (1 - pct / 100.0)).round(2)
  def affordable?           = price < 50
  def electronics?          = category == :electronics
  def to_s                  = "#{name} ($#{price})"
end
```
