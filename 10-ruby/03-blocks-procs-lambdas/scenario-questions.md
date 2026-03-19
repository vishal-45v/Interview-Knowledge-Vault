# Chapter 03 — Blocks, Procs & Lambdas: Scenario Questions

## Scenario 1 — What does this code print?

```ruby
def method_with_return
  p = Proc.new { return "from proc" }
  p.call
  "after proc call"
end

def method_with_lambda
  l = lambda { return "from lambda" }
  l.call
  "after lambda call"
end

puts method_with_return
puts method_with_lambda
```

**Answer:**
```
from proc
after lambda call
```

The Proc's `return` returns from the enclosing method `method_with_return`. The lambda's `return` exits only the lambda, so `method_with_lambda` continues and returns `"after lambda call"`.

---

## Scenario 2 — Closure mutation trap

```ruby
multipliers = []

[1, 2, 3].each do |i|
  multipliers << proc { |n| n * i }
end

puts multipliers.map { |m| m.call(10) }.inspect
```

**Answer:** `[10, 20, 30]`.

Each iteration of `each` creates a new binding for `i` (each `do |i|` block gets its own `i`). So the three procs capture i=1, i=2, and i=3 respectively. This is different from a `while` loop where the same variable is mutated.

**Contrast with the mutation trap:**
```ruby
multipliers = []
i = 1
while i <= 3
  multipliers << proc { |n| n * i }  # all capture SAME i variable
  i += 1
end
# After loop, i == 4
puts multipliers.map { |m| m.call(10) }.inspect
# => [40, 40, 40]  (all see final i = 4... wait, i is 4 after the loop)
# Actually => [40, 40, 40] because i ends at 4 after the while exits
```

---

## Scenario 3 — Arity difference: Proc vs Lambda

```ruby
lam = lambda { |x, y| "#{x}, #{y}" }
prc = proc   { |x, y| "#{x}, #{y}" }

begin
  puts lam.call(1)
rescue ArgumentError => e
  puts "Lambda error: #{e.message}"
end

puts prc.call(1).inspect
puts prc.call(1, 2, 3).inspect
```

**Answer:**
```
Lambda error: wrong number of arguments (given 1, expected 2)
"1, "
"1, 2"
```

Lambda enforces arity strictly. Proc silently sets missing args to `nil` (`nil.to_s == ""`) and ignores extra args.

---

## Scenario 4 — &:symbol with collect

```ruby
class Product
  attr_reader :name, :price

  def initialize(name, price)
    @name  = name
    @price = price
  end

  def discounted_price
    (@price * 0.9).round(2)
  end
end

products = [
  Product.new("Widget", 9.99),
  Product.new("Gadget", 24.99),
  Product.new("Doohickey", 4.99)
]

puts products.map(&:name).inspect
puts products.map(&:discounted_price).inspect
puts products.sort_by(&:price).map(&:name).inspect
```

**Answer:**
```
["Widget", "Gadget", "Doohickey"]
[8.99, 22.49, 4.49]
["Doohickey", "Widget", "Gadget"]
```

`&:method_name` works with any instance method that takes no arguments.

---

## Scenario 5 — Building a DSL with blocks

```ruby
class HtmlBuilder
  def initialize
    @html = []
  end

  def div(&block)
    @html << "<div>"
    instance_eval(&block) if block_given?
    @html << "</div>"
    self
  end

  def p(text)
    @html << "<p>#{text}</p>"
    self
  end

  def h1(text)
    @html << "<h1>#{text}</h1>"
    self
  end

  def build
    @html.join("\n")
  end
end

html = HtmlBuilder.new
html.div do
  h1 "Welcome"
  p "Hello, World!"
  p "This is Ruby DSL"
end

puts html.build
```

**Answer:**
```html
<div>
<h1>Welcome</h1>
<p>Hello, World!</p>
<p>This is Ruby DSL</p>
</div>
```

`instance_eval(&block)` executes the block in the context of `self` (the HtmlBuilder instance), so method calls inside the block resolve to the builder's methods without `builder.` prefix.

---

## Scenario 6 — Lazy enumerator with infinite sequence

```ruby
natural_numbers = Enumerator.new do |y|
  n = 1
  loop do
    y << n
    n += 1
  end
end

result = natural_numbers
  .lazy
  .select { |n| n % 3 == 0 }
  .map    { |n| n ** 2 }
  .first(5)

puts result.inspect
```

**Answer:** `[9, 36, 81, 144, 225]` (squares of 3, 6, 9, 12, 15).

The lazy enumerator only processes elements until it has collected 5. Without `.lazy`, the `select` would try to enumerate all natural numbers forever.

---

## Scenario 7 — Passing a block to another method

```ruby
def with_logging(name, &block)
  puts "Starting: #{name}"
  result = block.call
  puts "Finished: #{name}"
  result
end

def process_data(data)
  with_logging("process") do
    data.map { |n| n * 2 }
  end
end

result = process_data([1, 2, 3])
puts result.inspect
```

**Answer:**
```
Starting: process
Finished: process
[2, 4, 6]
```

The `&block` parameter captures the block as a Proc. You can then pass it down with `block.call` or re-pass it as `&block` to another method.

---

## Scenario 8 — Currying for partial application

```ruby
validate = ->(min, max, value) {
  value >= min && value <= max
}

valid_percentage = validate.curry.(0).(100)
valid_age        = validate.curry.(0).(150)
valid_score      = validate.curry.(1).(10)

puts valid_percentage.(50)   # ?
puts valid_percentage.(150)  # ?
puts valid_age.(25)          # ?
puts valid_score.(0)         # ?
puts [85, 102, 73, 95].select(&valid_percentage).inspect  # ?
```

**Answer:**
```
true
false
true
false
[85, 73, 95]
```

Currying enables creating specialized validators from a generic one. `valid_percentage` is a lambda that checks `0 <= value <= 100`.

---

## Scenario 9 — block_given? for optional formatting

```ruby
class Report
  def initialize(data)
    @data = data
  end

  def summary
    total = @data.sum
    avg   = total.to_f / @data.size

    result = { total: total, average: avg, count: @data.size }

    if block_given?
      yield result
    else
      result
    end
  end
end

r = Report.new([10, 20, 30, 40, 50])

puts r.summary.inspect
puts "---"
r.summary do |s|
  puts "Total: #{s[:total]}"
  puts "Avg:   #{s[:average]}"
  puts "Count: #{s[:count]}"
end
```

**Answer:**
```
{:total=>150, :average=>30.0, :count=>5}
---
Total: 150
Avg:   30.0
Count: 5
```

---

## Scenario 10 — Method reference as block

```ruby
class Formatter
  def currency(amount)
    "$#{'%.2f' % amount}"
  end

  def percentage(value)
    "#{'%.1f' % (value * 100)}%"
  end
end

f = Formatter.new

prices = [9.5, 24.99, 0.75]
rates  = [0.05, 0.125, 0.875]

puts prices.map(&f.method(:currency)).inspect
puts rates.map(&f.method(:percentage)).inspect
```

**Answer:**
```
["$9.50", "$24.99", "$0.75"]
["5.0%", "12.5%", "87.5%"]
```

`&f.method(:currency)` converts the bound method to a block, allowing it to be passed to `map`.

---

## Scenario 11 — Proc.new without a block inside a method

```ruby
def capture
  Proc.new  # no block given explicitly
end

p1 = capture { |x| x * 2 }
puts p1.call(5)

p2 = capture
puts p2.class
```

**Answer:** First prints `10`. Second raises `ArgumentError: tried to create Proc object without a block`. `Proc.new` inside a method with no explicit block argument will capture the method's own block. If the method is called without a block, it raises.

---

## Scenario 12 — Enumerator as external iterator

```ruby
enum = ["a", "b", "c"].each

loop do
  item = enum.next
  print item.upcase + " "
end

puts ""
```

**Answer:** Prints `A B C `. The `loop` construct automatically rescues `StopIteration`, which is raised by `enum.next` when the enumerator is exhausted — a clean idiom for external iteration.

---

## Scenario 13 — Lambda in hash for dispatch table

```ruby
OPERATIONS = {
  add:      ->(a, b) { a + b },
  subtract: ->(a, b) { a - b },
  multiply: ->(a, b) { a * b },
  divide:   ->(a, b) { b.zero? ? Float::INFINITY : a.to_f / b }
}.freeze

def calculate(op, a, b)
  operation = OPERATIONS[op]
  raise ArgumentError, "Unknown operation: #{op}" unless operation
  operation.call(a, b)
end

puts calculate(:add, 10, 5)
puts calculate(:divide, 10, 0)
puts calculate(:multiply, 3, 7)
```

**Answer:**
```
15
Infinity
21
```

Using lambdas in a dispatch table is a clean alternative to case/when statements. It's also open for extension without modifying the calculate method.

---

## Scenario 14 — Proc composed with >> and << operators (Ruby 2.6+)

```ruby
double    = ->(n)    { n * 2 }
increment = ->(n)    { n + 1 }
stringify = ->(n)    { n.to_s }

# >> composes left-to-right: double, then increment, then stringify
pipeline = double >> increment >> stringify

puts pipeline.call(5)   # ?
puts pipeline.call(10)  # ?

# << composes right-to-left
reverse_pipeline = stringify << increment << double
puts reverse_pipeline.call(5)   # ?
```

**Answer:**
```
11       # (5 * 2) + 1 = 11, then to_s
21       # (10 * 2) + 1 = 21, then to_s
11       # same result: double first, then increment, then stringify
```

Lambda/Proc composition with `>>` and `<<` enables functional pipelines without chaining `.call` explicitly.

---

## Scenario 15 — Closure over mutable state for memoization

```ruby
def memoize(fn)
  cache = {}
  ->(n) {
    cache[n] ||= fn.call(n)
  }
end

fib_raw = ->(n) {
  return n if n <= 1
  fib_raw.call(n - 1) + fib_raw.call(n - 2)
}

fast_fib = memoize(fib_raw)

puts fast_fib.call(10)
puts fast_fib.call(10)  # returns from cache
```

**Answer:** Both print `55`. The `memoize` function returns a new lambda that closes over a private `cache` hash. First call computes and stores; second call retrieves from cache. This is a functional pattern for adding caching to any pure function.

Note: The example above has limited benefit because `fib_raw` doesn't go through `fast_fib` recursively. A production memoized fibonacci would define itself through the memoized version.
