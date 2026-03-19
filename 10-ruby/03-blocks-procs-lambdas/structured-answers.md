# Chapter 03 — Blocks, Procs & Lambdas: Structured Answers

---

## Answer 1 — "Explain the difference between a Proc and a Lambda in detail"

Procs and Lambdas are both instances of the `Proc` class, but they have two fundamental behavioral differences.

**Difference 1: Argument count (arity)**

```ruby
# Lambda — strict arity (like a method)
strict = lambda { |x, y| "#{x} and #{y}" }
strict.call(1, 2)    # => "1 and 2"
strict.call(1)       # => ArgumentError: wrong number of arguments (given 1, expected 2)
strict.call(1, 2, 3) # => ArgumentError: wrong number of arguments (given 3, expected 2)

# Proc — lenient arity (missing args = nil, extra args dropped)
lenient = proc { |x, y| "#{x} and #{y}" }
lenient.call(1, 2)    # => "1 and 2"
lenient.call(1)       # => "1 and "  (y is nil)
lenient.call(1, 2, 3) # => "1 and 2"  (3 silently dropped)
```

**Difference 2: return semantics**

```ruby
def demo_lambda
  l = lambda { return "lambda return" }
  l.call                    # return exits the lambda
  "method continues"        # this runs
end

def demo_proc
  p = Proc.new { return "proc return" }
  p.call                    # return exits the ENCLOSING METHOD
  "method continues"        # this NEVER runs
end

demo_lambda  # => "method continues"
demo_proc    # => "proc return"
```

**When to use each:**
- Use `lambda` (or `->`) when you need strict argument checking and want `return` to exit only the lambda — e.g., callbacks, dispatchers, validators
- Use `proc` when you want flexible argument handling — e.g., event handlers where you might not always have all arguments
- Use blocks when you just need to pass code to a method and don't need to store or pass it around

---

## Answer 2 — "How do closures work in Ruby? What is a binding?"

A closure is a function that carries its surrounding scope with it. In Ruby, blocks, Procs, and Lambdas are all closures.

```ruby
def make_multiplier(factor)
  # `factor` is a local variable in make_multiplier
  # The lambda below captures it — this is a closure
  ->(n) { n * factor }
end

triple = make_multiplier(3)
double = make_multiplier(2)

# make_multiplier has already returned, but the lambdas still hold
# references to their respective `factor` variables
triple.call(5)  # => 15 (uses factor=3)
double.call(5)  # => 10 (uses factor=2)
```

```ruby
# Closures share variable bindings — they close over the REFERENCE
def shared_state
  count = 0

  increment = -> { count += 1 }
  decrement = -> { count -= 1 }
  current   = -> { count }

  [increment, decrement, current]
end

inc, dec, cur = shared_state
inc.call; inc.call; inc.call
dec.call
cur.call  # => 2  (all three lambdas share the same `count` binding)
```

**Binding in Ruby:**
```ruby
# Kernel#binding captures the current execution context
def context_demo
  x = 42
  b = binding  # captures: local vars, self, etc.
  b
end

b = context_demo
eval("x", b)  # => 42  (evaluates in the captured context)
eval("x * 2", b)  # => 84
```

---

## Answer 3 — "How does &:symbol work and what is Symbol#to_proc?"

```ruby
# The long way:
[1, 2, 3].map { |n| n.to_s }  # => ["1", "2", "3"]

# The short way:
[1, 2, 3].map(&:to_s)  # => ["1", "2", "3"]

# What happens:
# 1. :to_s.to_proc is called
# 2. It creates roughly: proc { |obj, *args| obj.send(:to_s, *args) }
# 3. & converts it to a block for map
```

**Implementing Symbol#to_proc from scratch:**
```ruby
class Symbol
  def to_proc
    method_name = self
    proc { |receiver, *args| receiver.send(method_name, *args) }
  end
end

# Now these all work:
[1, 2, 3].map(&:to_s)           # => ["1", "2", "3"]
["hello", "world"].map(&:upcase) # => ["HELLO", "WORLD"]
[1, 2, 3, 4].select(&:even?)    # => [2, 4]
["  hi  ", " there "].map(&:strip)  # => ["hi", "there"]
```

**The general & pattern:**
```ruby
# & calls to_proc on whatever follows it
# Works for: Symbol, Method, Proc/Lambda

# Symbol:
[1,2,3].map(&:to_s)

# Method:
[1,2,3].map(&method(:puts))   # prints each, returns [nil, nil, nil]

# Proc:
formatter = proc { |n| "$#{'%.2f' % n}" }
[9.5, 24.99].map(&formatter)  # => ["$9.50", "$24.99"]
```

---

## Answer 4 — "Explain lazy enumerators and when you'd use them"

```ruby
# Eager vs Lazy evaluation:

# EAGER — processes all elements before moving to next step
words = ["the", "quick", "brown", "fox", "jumps"]

# Creates full intermediate arrays at each step:
result = words
  .select { |w| w.length > 3 }      # => ["quick", "brown", "jumps"]
  .map    { |w| w.capitalize }       # => ["Quick", "Brown", "Jumps"]
  .first(2)                          # => ["Quick", "Brown"]

# LAZY — processes element-by-element, stops early
result = words
  .lazy
  .select { |w| w.length > 3 }
  .map    { |w| w.capitalize }
  .first(2)                          # stops after finding 2 matches
# => ["Quick", "Brown"]
```

**Key use cases:**

```ruby
# 1. Infinite sequences (impossible without lazy)
fibonacci = Enumerator.new do |y|
  a, b = 0, 1
  loop { y << a; a, b = b, a + b }
end

fibonacci.lazy.select { |n| n.even? }.first(10)
# => [0, 2, 8, 34, 144, 610, 2584, 10946, 46368, 196418]

# 2. Large file processing
File.foreach("huge_log.txt")
  .lazy
  .select  { |line| line.include?("ERROR") }
  .map     { |line| line.strip }
  .first(100)
# Reads only enough lines to find 100 errors — doesn't load whole file

# 3. Performance with pipeline of operations on large collections
(1..1_000_000)
  .lazy
  .map    { |n| n * n }
  .select { |n| n.odd? }
  .first(5)
# Only processes ~10 elements instead of all 1 million
```

---

## Answer 5 — "How would you implement memoization using closures?"

```ruby
# Basic memoize using a closure over a cache hash
def memoize(&fn)
  cache = {}
  ->(n) {
    unless cache.key?(n)
      cache[n] = fn.call(n)
    end
    cache[n]
  }
end

# Memoized fibonacci
fib = memoize { |n|
  n <= 1 ? n : fib.call(n - 1) + fib.call(n - 2)
}

fib.call(40)  # fast because results are cached
```

**Module-based memoize:**
```ruby
module Memoizable
  def memoize(method_name)
    original_method = instance_method(method_name)
    cache_var       = :"@_memo_#{method_name}"

    define_method(method_name) do |*args|
      cache = instance_variable_get(cache_var) ||
              instance_variable_set(cache_var, {})
      cache[args] ||= original_method.bind(self).call(*args)
    end
  end
end

class Calculator
  extend Memoizable

  def expensive_computation(n)
    puts "Computing #{n}..."
    n * n
  end

  memoize :expensive_computation
end

calc = Calculator.new
calc.expensive_computation(5)  # Computing 5... => 25
calc.expensive_computation(5)  # (no output — cached)    => 25
calc.expensive_computation(6)  # Computing 6... => 36
```

---

## Answer 6 — "What is currying and how would you use it in a real application?"

```ruby
# Currying converts f(a, b, c) into f(a).(b).(c)
# Enables partial application — pre-fill some arguments

# Generic validator factory
validate_range = ->(min, max, value) { value.between?(min, max) }

valid_age        = validate_range.curry.(0).(130)
valid_percentage = validate_range.curry.(0).(100)
valid_score      = validate_range.curry.(0).(10)

# Use as predicates:
ages = [25, 200, -5, 65]
ages.select(&valid_age)        # => [25, 65]

percentages = [50, 110, -1, 99]
percentages.select(&valid_percentage)  # => [50, 99]

# Real application: building configurable formatters
format_with = ->(template, value) { template % value }
money_format = format_with.curry.("$%.2f")
pct_format   = format_with.curry.("%.1f%%")

[9.5, 24.99, 0.75].map(&money_format)
# => ["$9.50", "$24.99", "$0.75"]

[0.15, 0.08, 0.22].map { |r| pct_format.(r * 100) }
# => ["15.0%", "8.0%", "22.0%"]
```

---

## Answer 7 — "Explain the tap, then/yield_self, and tee pattern"

```ruby
# tap — yields self, returns self (for debugging or side effects)
user = User.new
  .tap { |u| u.name = "Alice" }
  .tap { |u| u.email = "alice@example.com" }
  .tap { |u| puts "Building user: #{u.name}" }

# then / yield_self (Ruby 2.6+) — yields self, returns block result
# Useful for transforming values in a pipeline without intermediate variables
result = "  hello world  "
  .then { |s| s.strip }
  .then { |s| s.split(" ") }
  .then { |arr| arr.map(&:capitalize) }
  .then { |arr| arr.join("-") }
# => "Hello-World"

# Without then — nested or intermediate variables:
s = "  hello world  ".strip
arr = s.split(" ")
capitalized = arr.map(&:capitalize)
result = capitalized.join("-")

# Practical then example: conditional transformation
def process(value)
  value
    .then { |v| v.is_a?(String) ? v.upcase : v }
    .then { |v| v.respond_to?(:to_a) ? v.to_a : [v] }
end
```
