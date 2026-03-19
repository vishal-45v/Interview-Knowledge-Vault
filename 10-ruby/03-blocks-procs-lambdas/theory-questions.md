# Chapter 03 — Blocks, Procs & Lambdas: Theory Questions

## Blocks

**Q1. What is a block in Ruby?**

A block is an anonymous chunk of code that can be passed to a method. It is not an object by itself (unlike Procs), but it can be captured into a Proc object.

```ruby
# Block with do...end syntax:
[1, 2, 3].each do |n|
  puts n * 2
end

# Block with { } syntax (conventional for single-line):
[1, 2, 3].each { |n| puts n * 2 }

# Blocks can have multiple lines with either syntax:
result = [1, 2, 3].map { |n|
  doubled = n * 2
  doubled + 1
}
# => [3, 5, 7]
```

**Q2. What is the precedence difference between do...end and { }?**

```ruby
# { } binds TIGHTLY to the nearest method
# do...end binds LOOSELY — to the farthest method in the expression

def foo(x)
  puts "foo called with: #{x.inspect}"
end

# This calls: foo( [1,2,3].map { |n| n * 2 } )
# { } binds to .map, result [2,4,6] is passed to foo
foo [1, 2, 3].map { |n| n * 2 }
# foo called with: [2, 4, 6]

# This calls: foo([1,2,3].map) do |n| n * 2 end
# do...end binds to foo, .map receives no block (returns Enumerator)
foo [1, 2, 3].map do |n| n * 2 end
# foo called with: #<Enumerator:...>
```

---

## yield

**Q3. What is yield and how does it work?**

`yield` transfers control to the block passed to the current method. If no block is given, it raises `LocalJumpError`.

```ruby
def twice
  yield
  yield
end

twice { puts "hello" }
# hello
# hello

# yield with arguments — passes values to the block
def transform(value)
  yield(value)
end

transform(5) { |n| n * 3 }  # => 15

# yield with a return value — the block's return value is yield's return value
def apply(value)
  result = yield(value)
  "Result: #{result}"
end

apply(10) { |n| n + 5 }  # => "Result: 15"
```

**Q4. What is block_given? and when do you use it?**

```ruby
def log(message)
  if block_given?
    formatted = yield(message)
    puts formatted
  else
    puts message
  end
end

log("hello")                      # => hello
log("hello") { |m| m.upcase }    # => HELLO
log("hello") { |m| "[LOG] #{m}" } # => [LOG] hello

# Common pattern: optional block for customization
def with_retry(max_attempts: 3)
  attempts = 0
  begin
    result = yield
    return result
  rescue => e
    attempts += 1
    retry if attempts < max_attempts
    raise
  end
end

with_retry(max_attempts: 5) { risky_api_call }
```

---

## Proc

**Q5. What is a Proc and how do you create one?**

```ruby
# Proc.new — explicit creation
square = Proc.new { |n| n ** 2 }
square.call(5)   # => 25
square.(5)       # => 25  (syntactic sugar)
square[5]        # => 25  (also works)

# proc — shorthand for Proc.new in a method context
double = proc { |n| n * 2 }
double.call(4)   # => 8

# &block parameter — captures the block as a Proc object
def capture_block(&block)
  block           # block is now a Proc object
end

my_proc = capture_block { |x| x + 1 }
my_proc.call(5)   # => 6
my_proc.class     # => Proc

# Storing and passing blocks:
def build_formatter(template)
  proc { |value| template % value }
end

money_formatter = build_formatter("$%.2f")
money_formatter.call(9.5)   # => "$9.50"
```

---

## Lambda

**Q6. What is a Lambda and how do you create one?**

```ruby
# lambda keyword
greet = lambda { |name| "Hello, #{name}!" }
greet.call("Alice")  # => "Hello, Alice!"
greet.lambda?        # => true

# -> (stabby lambda) syntax — preferred in modern Ruby
greet = ->(name) { "Hello, #{name}!" }
greet.call("Alice")  # => "Hello, Alice!"

# Multi-line stabby lambda
transform = ->(x, y) {
  sum     = x + y
  product = x * y
  [sum, product]
}
transform.call(3, 4)   # => [7, 12]

# Lambda with default arguments
greet = ->(name, greeting = "Hello") { "#{greeting}, #{name}!" }
greet.call("Alice")         # => "Hello, Alice!"
greet.call("Alice", "Hi")   # => "Hi, Alice!"
```

---

## Proc vs Lambda: Key Differences

**Q7. What are the differences between a Proc and a Lambda?**

There are two critical differences: **arity checking** and **return behavior**.

```ruby
# DIFFERENCE 1: Argument count (arity)
strict_lambda = lambda { |x, y| x + y }
loose_proc    = Proc.new { |x, y| x.to_i + y.to_i }

strict_lambda.call(1, 2)       # => 3
strict_lambda.call(1)          # => ArgumentError! (too few args)
strict_lambda.call(1, 2, 3)    # => ArgumentError! (too many args)

loose_proc.call(1, 2)          # => 3
loose_proc.call(1)             # => 1  (y is nil, nil.to_i == 0)
loose_proc.call(1, 2, 3, 4)    # => 3  (extra args ignored silently)
```

```ruby
# DIFFERENCE 2: return behavior
def proc_return_demo
  p = Proc.new { return "from proc" }
  p.call
  "after proc"  # NEVER reached!
end

def lambda_return_demo
  l = lambda { return "from lambda" }
  l.call
  "after lambda"  # IS reached — lambda return exits lambda only
end

proc_return_demo    # => "from proc"
lambda_return_demo  # => "after lambda"
```

---

## &block, &method, &:symbol

**Q8. What does the & operator do in method signatures and method calls?**

```ruby
# In method definition: & captures block as Proc
def run(&block)
  puts block.class   # => Proc
  block.call(10)
end

run { |n| n * 2 }  # => 20

# In method call: & converts Proc/Method/Symbol to a block
double = proc { |n| n * 2 }
[1, 2, 3].map(&double)    # => [2, 4, 6]
# &double calls double.to_proc and passes it as a block

# &:symbol — Symbol#to_proc
# :upcase.to_proc creates: proc { |obj| obj.upcase }
["hello", "world"].map(&:upcase)     # => ["HELLO", "WORLD"]
[1, 2, 3, 4].select(&:odd?)         # => [1, 3]
["a", nil, "b", nil].compact.map(&:upcase)  # => ["A", "B"]

# &method(:name) — converts a method to a block
def double(n)
  n * 2
end

[1, 2, 3].map(&method(:double))  # => [2, 4, 6]
```

**Q9. How does Symbol#to_proc work under the hood?**

```ruby
# :upcase.to_proc is roughly equivalent to:
# proc { |obj, *args| obj.send(:upcase, *args) }

# So &:upcase on "hello" does:
# "hello".send(:upcase) => "HELLO"

# This works for any method name:
[1, 2, 3].map(&:to_s)           # => ["1", "2", "3"]
[1, -2, 3, -4].select(&:positive?)  # => [1, 3]
["  hello  ", "  world  "].map(&:strip)  # => ["hello", "world"]

# With methods that take arguments (you need a lambda instead):
# &:method doesn't pass arguments
# Use a lambda: ->(n) { n.round(2) }
[1.234, 2.567].map { |n| n.round(2) }  # => [1.23, 2.57]
```

---

## Closures

**Q10. What is a closure in Ruby and how does it capture scope?**

A closure is a block/Proc/Lambda that captures the surrounding variables at the time of creation — and can access them even after the enclosing scope is gone.

```ruby
def make_counter(start = 0)
  count = start

  increment = -> { count += 1; count }
  decrement = -> { count -= 1; count }
  value     = -> { count }

  [increment, decrement, value]
end

inc, dec, val = make_counter(10)
inc.call   # => 11
inc.call   # => 12
dec.call   # => 11
val.call   # => 11

# The closures SHARE the same binding — they all reference the same `count`
```

```ruby
# Closure captures BINDING, not VALUE — mutation trap!
lambdas = []
5.times do |i|
  lambdas << -> { i }
end

lambdas.map(&:call)   # => [0, 1, 2, 3, 4]  (each captures its own i)
# This is fine because each block iteration creates a new `i` binding

# But:
callbacks = []
i = 0
while i < 5
  callbacks << -> { i }  # all capture the SAME i variable
  i += 1
end

callbacks.map(&:call)   # => [5, 5, 5, 5, 5]  (all see final value of i)
```

---

## Enumerators and Lazy Enumerators

**Q11. What is an Enumerator and how do you use it?**

```ruby
# Methods called without a block return an Enumerator
enum = [1, 2, 3].each
enum.class   # => Enumerator

enum.next    # => 1
enum.next    # => 2
enum.next    # => 3
enum.next    # => StopIteration

# Enumerator.new for custom iteration
fib = Enumerator.new do |yielder|
  a, b = 0, 1
  loop do
    yielder << a     # same as yielder.yield(a)
    a, b = b, a + b
  end
end

fib.first(10)  # => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
fib.take(5)    # => [0, 1, 1, 2, 3]
fib.lazy.select { |n| n.even? }.first(5)
# => [0, 2, 8, 34, 144]
```

**Q12. What is a lazy enumerator and why does it matter for performance?**

```ruby
# Without lazy — creates full intermediate arrays
result = (1..Float::INFINITY)
  .select { |n| n.odd? }      # infinite! would run forever
  .map    { |n| n ** 2 }
  .first(5)                   # never reached

# With lazy — only processes what's needed
result = (1..Float::INFINITY)
  .lazy
  .select { |n| n.odd? }
  .map    { |n| n ** 2 }
  .first(5)
# => [1, 9, 25, 49, 81]  — processed only enough elements to get 5

# Performance comparison with large datasets:
large_range = (1..1_000_000)

# Eager: creates 1M array, then 1M array, then takes 5
large_range.map { |n| n * 2 }.select { |n| n > 100 }.first(5)

# Lazy: processes only until 5 elements found
large_range.lazy.map { |n| n * 2 }.select { |n| n > 100 }.first(5)
```

---

## Currying

**Q13. What is currying and how does Ruby support it?**

```ruby
# Currying transforms a multi-argument function into a chain of
# single-argument functions

add = ->(a, b) { a + b }
add.call(3, 4)    # => 7

# Curried version:
curried_add = add.curry
add5 = curried_add.call(5)   # partial application — returns a lambda
add5.call(3)                  # => 8
add5.call(10)                 # => 15

# One-liner:
add5 = ->(a, b) { a + b }.curry.call(5)
[1, 2, 3].map(&add5)   # => [6, 7, 8]

# More complex example:
multiply = ->(factor, n) { n * factor }
triple   = multiply.curry.(3)
double   = multiply.curry.(2)

[1, 2, 3, 4].map(&triple)  # => [3, 6, 9, 12]
[1, 2, 3, 4].map(&double)  # => [2, 4, 6, 8]
```

---

## Method Objects

**Q14. What is a Method object in Ruby?**

```ruby
# method(:name) returns a bound Method object
class Calculator
  def multiply(a, b)
    a * b
  end
end

calc = Calculator.new
m    = calc.method(:multiply)

m.class     # => Method
m.arity     # => 2
m.call(3,4) # => 12
m.(3, 4)    # => 12

# Convert to Proc for use with & operator
[1, 2, 3].map(&calc.method(:multiply).curry.call(10))
# => [10, 20, 30]

# UnboundMethod — not bound to any instance
unbound = Calculator.instance_method(:multiply)
unbound.class   # => UnboundMethod
# Must bind before calling:
bound = unbound.bind(Calculator.new)
bound.call(5, 6)  # => 30

# method() on built-in methods:
m = method(:puts)
m.call("hello")   # => "hello"
[1, 2, 3].each(&method(:puts))   # prints each number
```

---

## Proc#arity

**Q15. What does arity return for different block/lambda signatures?**

```ruby
# arity returns the number of required arguments
# Negative: -(n+1) means at least n required args, rest optional

Proc.new { }.arity              # => 0
Proc.new { |x| }.arity         # => 1
Proc.new { |x, y| }.arity      # => 2
Proc.new { |x, y, z| }.arity   # => 3

# Optional arguments — negative arity
Proc.new { |*args| }.arity      # => -1  (0 required, rest)
Proc.new { |x, *args| }.arity   # => -2  (1 required, rest)
Proc.new { |x, y, *args| }.arity  # => -3  (2 required, rest)

lambda { |x, y = 1| }.arity    # => -2  (1 required, 1 optional)
lambda { |x, y:| }.arity       # => 1   (1 keyword required)

# Practical use: safely call a block regardless of arity
def flexible_call(block, *args)
  if block.arity >= 0 && args.length != block.arity
    block.call(*args.first(block.arity))
  else
    block.call(*args)
  end
end
```
