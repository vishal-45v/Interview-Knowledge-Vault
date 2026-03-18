# Chapter 04 — Collections & Enumerable: Theory Questions

## Array Core Methods

**Q1. What are the most important Array transformation methods?**

```ruby
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# map/collect — transform each element, return new array
arr.map { |n| n ** 2 }     # => [1, 4, 9, 16, 25, ...]

# select/filter — keep elements where block returns truthy
arr.select { |n| n.even? } # => [2, 4, 6, 8, 10]

# reject — remove elements where block returns truthy
arr.reject { |n| n.even? } # => [1, 3, 5, 7, 9]

# find/detect — first element where block returns true
arr.find { |n| n > 5 }     # => 6
arr.detect { |n| n > 5 }   # => 6  (alias)

# find_index / index
arr.find_index { |n| n > 5 }  # => 5 (index of first element > 5)
arr.index(7)                   # => 6 (index of value 7)

# reduce/inject — fold to a single value
arr.reduce(:+)                    # => 55 (sum)
arr.reduce(0) { |sum, n| sum + n } # => 55
arr.inject { |product, n| product * n }  # => 3628800 (factorial of 10)

# each_with_index
arr.each_with_index do |val, idx|
  puts "#{idx}: #{val}"
end

# each_with_object — thread an accumulator through
arr.each_with_object([]) do |n, acc|
  acc << n * 2 if n.odd?
end
# => [2, 6, 10, 14, 18]
```

**Q2. What are flat_map, tally, combination, and permutation?**

```ruby
# flat_map — map then flatten one level
[[1,2], [3,4], [5,6]].flat_map { |a| a.map { |n| n * 2 } }
# => [2, 4, 6, 8, 10, 12]
# Equivalent to .map { ... }.flatten(1)

# tally — count occurrences (Ruby 2.7+)
["red", "blue", "red", "green", "blue", "red"].tally
# => {"red"=>3, "blue"=>2, "green"=>1}

# combination — all combinations of given size
[1,2,3,4].combination(2).to_a
# => [[1,2],[1,3],[1,4],[2,3],[2,4],[3,4]]

# permutation — all ordered arrangements of given size
[1,2,3].permutation(2).to_a
# => [[1,2],[1,3],[2,1],[2,3],[3,1],[3,2]]

# zip — interleave arrays
[1,2,3].zip([4,5,6], [7,8,9])
# => [[1,4,7],[2,5,8],[3,6,9]]

# take / drop
arr.take(3)       # => [1, 2, 3]  (first 3)
arr.drop(7)       # => [8, 9, 10] (all after first 7)
arr.take_while { |n| n < 5 }  # => [1, 2, 3, 4]
arr.drop_while { |n| n < 5 }  # => [5, 6, 7, 8, 9, 10]
```

---

## Hash Core Methods

**Q3. What are the most important Hash transformation methods?**

```ruby
h = { alice: 95, bob: 87, carol: 92, dave: 78 }

# transform_values — modify each value
h.transform_values { |score| score >= 90 ? "A" : "B" }
# => {alice: "A", bob: "B", carol: "A", dave: "B"}

# transform_keys — modify each key
h.transform_keys { |k| k.to_s.upcase }
# => {"ALICE"=>95, "BOB"=>87, "CAROL"=>92, "DAVE"=>78}

# filter_map — select + transform in one pass (Ruby 2.7+)
h.filter_map { |name, score| "#{name}: #{score}" if score >= 90 }
# => ["alice: 95", "carol: 92"]

# group_by — group elements by a key
scores = [45, 82, 91, 65, 88, 73, 95, 52]
scores.group_by { |s| s >= 90 ? "A" : s >= 80 ? "B" : s >= 70 ? "C" : "F" }
# => {"F"=>[45, 52], "B"=>[82, 88], "A"=>[91, 95], "C"=>[65, 73]}

# min_by / max_by
h.min_by { |name, score| score }  # => [:dave, 78]
h.max_by { |name, score| score }  # => [:alice, 95]

# sort_by
h.sort_by { |name, score| score }.to_h
# => {dave: 78, bob: 87, carol: 92, alice: 95}

# count with block
h.count { |name, score| score >= 90 }  # => 2

# sum — Ruby 2.4+
h.sum { |name, score| score }  # => 352
```

**Q4. What does Hash#merge do with a conflict block?**

```ruby
defaults = { timeout: 30, retries: 3, debug: false }
overrides = { timeout: 60, verbose: true }

# Without block: overrides win
defaults.merge(overrides)
# => {timeout: 60, retries: 3, debug: false, verbose: true}

# With block: you decide who wins per key
defaults.merge(overrides) { |key, old_val, new_val|
  puts "Conflict: #{key} = #{old_val} vs #{new_val}"
  old_val  # keep the default
}
# Conflict: timeout = 30 vs 60
# => {timeout: 30, retries: 3, debug: false, verbose: true}

# Practical: deep merge
def deep_merge(h1, h2)
  h1.merge(h2) { |_key, v1, v2|
    if v1.is_a?(Hash) && v2.is_a?(Hash)
      deep_merge(v1, v2)
    else
      v2  # override wins for non-hash values
    end
  }
end
```

---

## Enumerable Module

**Q5. What does the Enumerable module provide?**

```ruby
# Enumerable requires that your class defines #each
# Then you get 50+ methods for free

# Existence checks
[1,2,3].all? { |n| n > 0 }    # => true  (all positive)
[1,2,3].any? { |n| n > 2 }    # => true  (at least one > 2)
[1,2,3].none? { |n| n > 5 }   # => true  (none > 5)
[1,2,3].one? { |n| n > 2 }    # => true  (exactly one > 2)

# Counting
[1,2,2,3,3,3].count               # => 6
[1,2,2,3,3,3].count(3)            # => 3
[1,2,2,3,3,3].count { |n| n > 1 } # => 5

# Aggregation
[1..5].flat_map(&:to_a).sum   # => 15
["a","bb","ccc"].sum(&:length) # => 6

# Partitioning
[1,2,3,4,5,6].partition { |n| n.even? }
# => [[2,4,6],[1,3,5]]  — [truthy, falsy]

# min/max
[3,1,4,1,5,9].min     # => 1
[3,1,4,1,5,9].max     # => 9
[3,1,4,1,5,9].minmax  # => [1, 9]
[3,1,4,1,5,9].min(3)  # => [1, 1, 3]  (first 3 smallest)

# Chunking
[1,1,2,2,3,1,1].chunk { |n| n }.map { |k, v| [k, v.size] }
# => [[1, 2], [2, 2], [3, 1], [1, 2]]
```

**Q6. What are chunk_while, slice_when, and each_cons/each_slice?**

```ruby
# each_slice — groups into fixed-size chunks
(1..10).each_slice(3).to_a
# => [[1,2,3],[4,5,6],[7,8,9],[10]]

# each_cons — sliding window
(1..6).each_cons(3).to_a
# => [[1,2,3],[2,3,4],[3,4,5],[4,5,6]]

# chunk_while — keep consecutive elements together while condition holds
[1,2,3,5,6,10,11,12].chunk_while { |a, b| b == a + 1 }.to_a
# => [[1,2,3],[5,6],[10,11,12]]  (consecutive runs)

# slice_when — break when condition holds
[1,2,3,5,6,10,11,12].slice_when { |a, b| b != a + 1 }.to_a
# => [[1,2,3],[5,6],[10,11,12]]  (same result, different logic)

# tally_by
["apple", "banana", "apricot", "blueberry"].tally
# => {"apple"=>1, "banana"=>1, "apricot"=>1, "blueberry"=>1}
```

---

## Set

**Q7. What is Set and when should you use it over Array?**

```ruby
require 'set'

# Set: unordered collection of unique elements, O(1) lookup
s = Set.new([1, 2, 3, 2, 1])
s.to_a      # => [1, 2, 3]  (duplicates removed)

s.add(4)           # add element
s.include?(3)      # => true  (O(1) hash-based lookup)
s.delete(2)        # remove element

# Set operations:
a = Set.new([1, 2, 3, 4])
b = Set.new([3, 4, 5, 6])

a | b              # => Set {1,2,3,4,5,6}  union
a & b              # => Set {3,4}           intersection
a - b              # => Set {1,2}           difference
a ^ b              # => Set {1,2,5,6}       symmetric difference

a.subset?(Set.new([1,2,3,4,5]))   # => true
a.superset?(Set.new([1,2]))       # => true

# When to use Set vs Array:
# Array#include? is O(n) — searches linearly
# Set#include? is O(1) — hash-based lookup

# SLOW for large collections:
valid_statuses = ["active", "inactive", "suspended"]
valid_statuses.include?("active")  # O(n)

# FAST:
valid_statuses = Set.new(["active", "inactive", "suspended"])
valid_statuses.include?("active")  # O(1)
```

---

## Comparable

**Q8. How do you implement custom sorting with Comparable?**

```ruby
class Employee
  include Comparable

  attr_reader :name, :salary, :tenure_years

  def initialize(name, salary, tenure_years)
    @name          = name
    @salary        = salary
    @tenure_years  = tenure_years
  end

  # Primary sort: salary descending, secondary: tenure descending
  def <=>(other)
    result = other.salary <=> salary    # note: reversed for descending
    result = other.tenure_years <=> tenure_years if result == 0
    result
  end

  def to_s
    "#{name} ($#{salary}, #{tenure_years}yrs)"
  end
end

employees = [
  Employee.new("Alice", 90_000, 5),
  Employee.new("Bob",   90_000, 3),
  Employee.new("Carol", 110_000, 2),
  Employee.new("Dave",  75_000, 8)
]

employees.sort.map(&:to_s)
# => ["Carol ($110000, 2yrs)", "Alice ($90000, 5yrs)",
#     "Bob ($90000, 3yrs)", "Dave ($75000, 8yrs)"]

employees.min   # => Dave (lowest salary)
employees.max   # => Carol (highest salary)
```

---

## Lazy Enumerators

**Q9. How do lazy enumerators work internally?**

```ruby
# Regular enumerator — eager
result = (1..10).map { |n| n * 2 }.select { |n| n > 10 }
# Creates two full arrays before returning

# Lazy enumerator — each element flows through the full pipeline
result = (1..10).lazy.map { |n| n * 2 }.select { |n| n > 10 }.to_a

# force or to_a materializes lazy results
(1..Float::INFINITY).lazy.map { |n| n * 2 }.first(5)
# => [2, 4, 6, 8, 10]

# Chain operations without creating intermediate arrays:
numbers = (1..1_000_000).lazy

result = numbers
  .select { |n| n % 3 == 0 }
  .map    { |n| n * n }
  .reject { |n| n % 2 == 0 }
  .first(5)
# => [9, 225, 441, 1089, 1521]  (only processes enough elements)
```

---

## filter_map

**Q10. What is filter_map and how does it improve code clarity?**

```ruby
# filter_map — select + map in one pass (Ruby 2.7+)
# Equivalent to .map { }.compact (but faster, single pass)

users = [
  { name: "Alice", age: 25, admin: true },
  { name: "Bob",   age: 17, admin: false },
  { name: "Carol", age: 32, admin: true },
  { name: "Dave",  age: 15, admin: false }
]

# Old way — two passes
admins = users
  .select { |u| u[:admin] }
  .map    { |u| u[:name] }
# => ["Alice", "Carol"]

# filter_map — one pass, returns mapped value if truthy
admins = users.filter_map { |u| u[:name] if u[:admin] }
# => ["Alice", "Carol"]

# More complex example:
data = ["1", "abc", "2", nil, "3", "xyz"]
numbers = data.filter_map { |s| Integer(s) rescue nil }
# => [1, 2, 3]
```

---

## each_with_object vs inject

**Q11. When do you use each_with_object vs inject/reduce?**

```ruby
words = ["apple", "banana", "cherry", "avocado"]

# each_with_object: accumulator is passed and returned automatically
# Best when accumulator is a mutable container (Hash, Array)
grouped = words.each_with_object(Hash.new { |h,k| h[k] = [] }) do |word, h|
  h[word[0]] << word
end
# => {"a"=>["apple", "avocado"], "b"=>["banana"], "c"=>["cherry"]}

# inject/reduce: memo is the block's return value
# Best when accumulating to a scalar or when building immutable structures
total_length = words.inject(0) { |sum, w| sum + w.length }
# => 25

# Trap with inject and Hash:
result = words.inject({}) do |memo, word|
  memo[word[0]] = word  # last word wins (not what we want for grouping)
  memo  # MUST return memo from the block!
end
# => {"a"=>"avocado", "b"=>"banana", "c"=>"cherry"}

# compare: each_with_object does NOT require returning the accumulator:
result = words.each_with_object({}) do |word, memo|
  memo[word[0]] = word  # no need to return memo
end
```

---

## sort_by (Schwartzian Transform)

**Q12. What is sort_by and why is it more efficient than sort with a block?**

```ruby
# sort with block — comparison function called O(n log n) times
# Each comparison recalculates the key
words = ["banana", "apple", "cherry", "date"]
words.sort { |a, b| a.length <=> b.length }
# Calls a.length and b.length repeatedly during comparison

# sort_by — Schwartzian transform
# Computes the key ONCE per element, then sorts by the cached keys
words.sort_by { |w| w.length }
# => ["date", "apple", "banana", "cherry"]
# Each word's length is computed ONCE, not once per comparison

# Multi-key sort_by using array comparison
employees.sort_by { |e| [-e.salary, e.name] }
# Sort by salary descending (negated), then name ascending

# Performance matters with expensive key computation:
files.sort_by { |f| File.mtime(f) }  # reads mtime ONCE per file
# vs:
files.sort { |a, b| File.mtime(a) <=> File.mtime(b) }
# reads mtime TWICE per comparison = 2 * n * log(n) file system calls
```

---

## Array#zip and Hash building

**Q13. How do you efficiently build hashes from arrays?**

```ruby
keys   = [:name, :age, :role]
values = ["Alice", 30, "admin"]

# Method 1: zip + to_h
keys.zip(values).to_h
# => {name: "Alice", age: 30, role: "admin"}

# Method 2: Hash[] constructor
Hash[keys.zip(values)]
# => {name: "Alice", age: 30, role: "admin"}

# Method 3: transpose
[keys, values].transpose.to_h
# => {name: "Alice", age: 30, role: "admin"}

# Build lookup from array
products = [
  { id: 1, name: "Widget", price: 9.99 },
  { id: 2, name: "Gadget", price: 24.99 }
]

lookup = products.each_with_object({}) { |p, h| h[p[:id]] = p }
lookup[1]  # => {id: 1, name: "Widget", price: 9.99}

# Or with group_by:
by_price_range = products.group_by { |p| p[:price] < 15 ? "cheap" : "expensive" }
```

---

## Enumerable#tally

**Q14. What does tally do and what did we use before it?**

```ruby
# tally — counts occurrences of each element (Ruby 2.7+)
["a", "b", "a", "c", "b", "a"].tally
# => {"a"=>3, "b"=>2, "c"=>1}

# Before tally (Ruby < 2.7):
arr = ["a", "b", "a", "c", "b", "a"]
arr.group_by { |x| x }.transform_values(&:count)
# => {"a"=>3, "b"=>2, "c"=>1}

# Or with inject:
arr.inject(Hash.new(0)) { |h, x| h[x] += 1; h }
# => {"a"=>3, "b"=>2, "c"=>1}

# tally_by (tally with a key function):
words = ["apple", "banana", "avocado", "blueberry", "cherry"]
words.tally  # exact match
words.group_by { |w| w[0] }.transform_values(&:count)
# => {"a"=>2, "b"=>2, "c"=>1}
```

---

## Struct as Value Object

**Q15. How do you use Struct in a collection context?**

```ruby
# Struct with Comparable for sorting collections
Person = Struct.new(:name, :age) do
  include Comparable

  def <=>(other)
    age <=> other.age
  end

  def to_s
    "#{name}(#{age})"
  end
end

people = [
  Person.new("Alice", 30),
  Person.new("Bob",   25),
  Person.new("Carol", 35)
]

people.sort.map(&:to_s)  # => ["Bob(25)", "Alice(30)", "Carol(35)"]
people.min               # => Bob(25)
people.max               # => Carol(35)

# Struct with Enumerable for custom collection
require 'set'
people.to_set  # works because Struct implements eql? and hash
```
