# Chapter 04 — Collections & Enumerable: Follow-Up Traps

---

## Trap 1 — map returns new array; map! mutates in-place

```ruby
arr = [1, 2, 3]
result = arr.map { |n| n * 2 }

puts arr.inspect     # => [1, 2, 3]  (original unchanged)
puts result.inspect  # => [2, 4, 6]  (new array)

# map! mutates the original
arr.map! { |n| n * 2 }
puts arr.inspect     # => [2, 4, 6]  (original changed)

# This applies to all bang methods: select!, reject!, map!, sort!, etc.
# map! returns nil if no change (like gsub!), but actually always returns self for map!

# Frozen array:
arr = [1, 2, 3].freeze
arr.map! { |n| n * 2 }  # => FrozenError
arr.map  { |n| n * 2 }  # => [2, 4, 6] (works — creates new array)
```

---

## Trap 2 — inject(:+) vs inject { |sum, n| sum + n }

```ruby
# inject with symbol: uses the method name as operator
[1, 2, 3, 4].inject(:+)   # => 10  (calls +)
[1, 2, 3, 4].inject(:*)   # => 24  (calls *)
["a","b","c"].inject(:+)   # => "abc"  (String#+)

# inject with block: your own logic
[1, 2, 3, 4].inject { |sum, n| sum + n }  # => 10

# inject with initial value + symbol
[1, 2, 3].inject(10, :+)   # => 16  (starts from 10)

# Trap 1: inject(:+) with empty array
[].inject(:+)    # => nil  (no initial value, no elements)
[].inject(0, :+) # => 0    (has initial value, safe)

# Trap 2: inject(:+) with single element
[5].inject(:+)   # => 5  (no call needed, returns the element)

# Trap 3: inject with no initial value uses first element as initial
[1, 2, 3].inject { |sum, n| sum + n }
# iteration 1: sum=1, n=2 → sum=3
# iteration 2: sum=3, n=3 → sum=6
```

---

## Trap 3 — select vs find: return type difference

```ruby
numbers = [1, 3, 5, 7, 2, 4]

# select always returns an ARRAY
numbers.select { |n| n.even? }   # => [2, 4]
numbers.select { |n| n > 100 }   # => []   (empty array, not nil)

# find always returns ONE element or nil
numbers.find { |n| n.even? }     # => 2    (first match)
numbers.find { |n| n > 100 }     # => nil  (no match)

# Common bug: treating find result as array
result = numbers.find { |n| n.even? }
result.map { |n| n * 2 }  # => NoMethodError: undefined method 'map' for 2:Integer

# Safe pattern:
result = numbers.select { |n| n.even? }
result.map { |n| n * 2 }  # => [4, 8]  (always an array)

# Or:
numbers.filter_map { |n| n * 2 if n.even? }  # => [4, 8]
```

---

## Trap 4 — Hash#merge — later key wins; merge with block for conflict resolution

```ruby
# receiver.merge(argument) — argument wins for conflicts
{ a: 1 }.merge({ a: 2 })   # => {a: 2}  (argument wins)
{ a: 2 }.merge({ a: 1 })   # => {a: 1}  (argument wins again)

# Common trap: assuming merge is commutative
config = { timeout: 30, debug: false }
overrides = { timeout: 60 }

# You want overrides to win:
config.merge(overrides)   # => {timeout: 60, debug: false}  ✓

# Accidentally reversed:
overrides.merge(config)   # => {timeout: 30, debug: false}  ✗ (config wins)

# merge! (alias: update) mutates the receiver:
config.merge!(overrides)
config  # => {timeout: 60, debug: false}  (mutated!)

# merge returns new hash; merge! modifies receiver
original = { a: 1 }
original.merge({ b: 2 })   # original is unchanged
original.merge!({ b: 2 })  # original is now {a: 1, b: 2}
```

---

## Trap 5 — Array#flatten! returns nil if nothing to flatten

```ruby
already_flat = [1, 2, 3]
result = already_flat.flatten!   # => nil  (nothing to flatten)
already_flat                     # => [1, 2, 3]  (unchanged)

nested = [1, [2, 3]]
result = nested.flatten!         # => [1, 2, 3]  (changed)
nested                           # => [1, 2, 3]  (mutated)

# Bug: chaining on flatten! when nothing to flatten
arr = [1, 2, 3]
arr.flatten!.map { |n| n * 2 }   # => NoMethodError: undefined method 'map' for nil

# Safe patterns:
arr.flatten.map { |n| n * 2 }    # non-bang is safe
(arr.flatten! || arr).map { |n| n * 2 }  # fallback to arr if nil
```

---

## Trap 6 — Lazy evaluation: .lazy.map.first(5) vs .map.first(5)

```ruby
# Critical difference: lazy stops early; eager builds full array

# Eager: materializes the ENTIRE intermediate array
(1..1_000_000).map { |n| puts "computing #{n}"; n * 2 }.first(5)
# prints "computing 1" through "computing 1000000" — ALL 1M!
# then takes first 5 from the 1M element array

# Lazy: processes only as many elements as needed
(1..1_000_000).lazy.map { |n| puts "computing #{n}"; n * 2 }.first(5)
# prints "computing 1" through "computing 5" — only 5!

# The lazy enumerator doesn't evaluate until forced by first/to_a/force
lazy = (1..Float::INFINITY).lazy.select { |n| n.odd? }
# nothing computed yet!
lazy.first(3)  # NOW it computes: [1, 3, 5]

# Trap: mixing lazy with eager operations
(1..10).lazy.map { |n| n * 2 }.sort
# sort forces evaluation of everything: [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
# sort is not lazy-compatible — it needs all elements
```

---

## Trap 7 — each returns the receiver; map returns new array

```ruby
arr = [1, 2, 3]

# each — always returns the ORIGINAL receiver
result = arr.each { |n| n * 2 }
result.equal?(arr)   # => true  (same object)
result.inspect       # => "[1, 2, 3]"  (unchanged)

# map — returns NEW array
result = arr.map { |n| n * 2 }
result.equal?(arr)   # => false  (different object)
result.inspect       # => "[2, 4, 6]"

# Common mistake: expecting each to return transformed values
doubled = [1,2,3].each { |n| n * 2 }
doubled.sum  # => 6  (original array summed, not doubled!)

# Use map when you want transformed results
doubled = [1,2,3].map { |n| n * 2 }
doubled.sum  # => 12  (doubled values summed)
```

---

## Trap 8 — sort vs sort_by performance (Schwartzian transform)

```ruby
# sort with block: key function called MULTIPLE TIMES per element
words = ["banana", "apple", "cherry", "date", "elderberry"]

# O(n log n) comparisons, each calls the key function TWICE:
words.sort { |a, b| a.length <=> b.length }
# a.length called: 2 * n * log(n) times ≈ 50 calls for 5 elements

# sort_by: key function called ONCE per element
words.sort_by { |w| w.length }
# w.length called: exactly n times = 5 calls

# The difference matters when the key is expensive:
# Slow:
files.sort { |a, b| File.stat(a).mtime <=> File.stat(b).mtime }
# Fast (Schwartzian transform — precompute keys):
files.sort_by { |f| File.stat(f).mtime }

# When to use sort vs sort_by:
# sort — simple spaceship operator, or when comparison logic is complex
# sort_by — when there's a single key to compute (always prefer for performance)
```

---

## Trap 9 — Hash.new with default block vs default value

```ruby
# Default VALUE (shared across all keys — dangerous!)
h = Hash.new([])
h[:a] << 1
h[:b] << 2
h[:a]   # => [1, 2]  ← both keys share the SAME array!
h[:b]   # => [1, 2]  ← same object

# Default BLOCK (creates new object per key — correct)
h = Hash.new { |hash, key| hash[key] = [] }
h[:a] << 1
h[:b] << 2
h[:a]   # => [1]   (only a's values)
h[:b]   # => [2]   (only b's values)

# Verify:
h[:a].object_id == h[:b].object_id  # => false  (different arrays)

# Also note: default block with no assignment just returns (doesn't store)
h = Hash.new { [] }
h[:a] << 1
h[:a]   # => []  (nothing was stored! fresh [] each time)
# You need: Hash.new { |h, k| h[k] = [] } to store the default
```

---

## Trap 10 — Enumerable#min with empty collection

```ruby
[].min         # => nil  (no error!)
[].max         # => nil
[].min_by { |x| x }  # => nil

[].inject(:+)  # => nil  (no initial value, no elements)
[].inject(0, :+)  # => 0  (initial value used)
[].sum         # => 0   (sum default is 0)
[].count       # => 0

# Watch for nil propagation:
scores = []
average = scores.sum.to_f / scores.size  # => NaN (0.0 / 0)
# Fix:
average = scores.empty? ? 0.0 : scores.sum.to_f / scores.size

# Or:
average = scores.then { |s| s.empty? ? 0.0 : s.sum.to_f / s.size }
```

---

## Trap 11 — uniq uses eql? (not ==)

```ruby
# For basic types, == and eql? agree, so uniq works as expected
[1, 1.0, 1].uniq   # => [1, 1.0]  ← 1 and 1.0 are NOT eql?

# For custom objects:
class Point
  attr_reader :x, :y
  def initialize(x, y); @x, @y = x, y; end
  def ==(other)  # define == but not eql? and hash
    x == other.x && y == other.y
  end
end

p1 = Point.new(1, 2)
p2 = Point.new(1, 2)
p1 == p2      # => true
[p1, p2].uniq # => [p1, p2]  ← NOT deduplicated! eql? and hash not defined

# Fix: also define eql? and hash
class Point
  def eql?(other)
    x.eql?(other.x) && y.eql?(other.y)
  end

  def hash
    [x, y].hash
  end
end

[p1, p2].uniq  # => [p1]  ← now works
```

---

## Trap 12 — group_by vs tally: use cases

```ruby
data = ["a", "b", "a", "c", "b", "a"]

# tally: count only
data.tally  # => {"a"=>3, "b"=>2, "c"=>1}

# group_by: full groups
data.group_by { |x| x }
# => {"a"=>["a","a","a"], "b"=>["b","b"], "c"=>["c"]}

# When you need counts: use tally (cleaner, Ruby 2.7+)
# When you need the actual grouped elements: use group_by

# Trap: doing group_by + transform_values(&:count) is SLOWER than tally
# Because group_by stores all elements before counting
data.group_by { |x| x }.transform_values(&:count)  # slower
data.tally                                           # faster
```
