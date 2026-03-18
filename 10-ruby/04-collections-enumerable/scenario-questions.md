# Chapter 04 — Collections & Enumerable: Scenario Questions

## Scenario 1 — map vs each: what's the difference?

```ruby
result_each = [1, 2, 3].each { |n| n * 2 }
result_map  = [1, 2, 3].map  { |n| n * 2 }

puts result_each.inspect
puts result_map.inspect
```

**Answer:**
```
[1, 2, 3]   # each returns the RECEIVER (original array)
[2, 4, 6]   # map returns a NEW array with block results
```

This is a common interview mistake. `each` is for side effects; `map` is for transformations.

---

## Scenario 2 — inject gotcha: must return memo

```ruby
result = ["apple", "banana", "cherry"].inject({}) do |memo, word|
  memo[word] = word.length
  # What if we forget to return memo here?
end

puts result.inspect
```

**Answer:** `3` (not a hash!). The last assignment `memo[word] = word.length` returns the length (an integer), so `memo` becomes `3` after the last iteration, and inject returns `3`. The fix:

```ruby
result = ["apple", "banana", "cherry"].inject({}) do |memo, word|
  memo[word] = word.length
  memo   # must return memo explicitly!
end
# => {"apple"=>5, "banana"=>6, "cherry"=>6}

# Or use each_with_object (no need to return accumulator):
result = ["apple", "banana", "cherry"].each_with_object({}) do |word, memo|
  memo[word] = word.length
end
# => {"apple"=>5, "banana"=>6, "cherry"=>6}
```

---

## Scenario 3 — select vs find

```ruby
numbers = [1, 3, 5, 7, 2, 4, 6, 8]

evens = numbers.select { |n| n.even? }
first_even = numbers.find { |n| n.even? }

puts evens.inspect
puts first_even.inspect
```

**Answer:**
```
[2, 4, 6, 8]   # select returns ALL matching elements
2              # find returns the FIRST matching element
```

`find` also returns `nil` if no element matches (not an empty array).

---

## Scenario 4 — Hash merge conflict: later key wins

```ruby
h1 = { a: 1, b: 2, c: 3 }
h2 = { b: 99, d: 4 }

puts h1.merge(h2).inspect
puts h2.merge(h1).inspect
```

**Answer:**
```
{a: 1, b: 99, c: 3, d: 4}  # h2 wins on :b (h2 is the argument)
{b: 2, d: 4, a: 1, c: 3}   # h1 wins on :b (h1 is the argument)
```

In `receiver.merge(argument)`, argument values win for conflicting keys.

---

## Scenario 5 — flatten vs flatten(1)

```ruby
nested = [1, [2, [3, [4, [5]]]]]
puts nested.flatten.inspect
puts nested.flatten(1).inspect
puts nested.flatten(2).inspect
```

**Answer:**
```
[1, 2, 3, 4, 5]          # complete flattening
[1, 2, [3, [4, [5]]]]    # one level only
[1, 2, 3, [4, [5]]]      # two levels
```

---

## Scenario 6 — Lazy evaluation: performance demonstration

```ruby
require 'benchmark'

n = 1_000_000

eager_time = Benchmark.realtime do
  (1..n).select { |x| x.odd? }.map { |x| x * x }.first(10)
end

lazy_time = Benchmark.realtime do
  (1..n).lazy.select { |x| x.odd? }.map { |x| x * x }.first(10)
end

puts "Eager: #{'%.4f' % eager_time}s"
puts "Lazy:  #{'%.4f' % lazy_time}s"
puts "Speedup: #{(eager_time / lazy_time).round(1)}x"
```

**Expected output (approximate):**
```
Eager: 0.2500s
Lazy:  0.0001s
Speedup: ~2500x
```

The lazy version processes only ~19 elements (the first 19 odd numbers to get 10 odd squares), while the eager version processes all 1 million elements twice.

---

## Scenario 7 — group_by for categorization

```ruby
transactions = [
  { id: 1, type: :credit,  amount: 100 },
  { id: 2, type: :debit,   amount: 50  },
  { id: 3, type: :credit,  amount: 200 },
  { id: 4, type: :debit,   amount: 75  },
  { id: 5, type: :fee,     amount: 5   }
]

grouped = transactions.group_by { |t| t[:type] }

grouped.each do |type, txns|
  total = txns.sum { |t| t[:amount] }
  puts "#{type}: #{txns.size} transactions, total $#{total}"
end
```

**Answer:**
```
credit: 2 transactions, total $300
debit: 2 transactions, total $125
fee: 1 transactions, total $5
```

---

## Scenario 8 — chunk_while for finding runs

```ruby
temps = [23, 24, 24, 25, 21, 22, 23, 30, 29, 28]

# Find consecutive increasing temperature runs
increasing_runs = temps.each_cons(2)
  .chunk_while { |(a, _), (_, b)| }  # hmm, is this right?

# Correct approach:
runs = temps.chunk_while { |a, b| b >= a }.to_a
puts runs.map(&:inspect).join("\n")
```

**Answer (corrected):**
```
[23, 24, 24, 25]
[21, 22, 23, 30]
[29]
[28]
```

Wait: `chunk_while { |a, b| b >= a }` keeps consecutive pairs where each next is >= previous. So `25 → 21` breaks (21 < 25), `30 → 29` breaks (29 < 30).

```ruby
# Explanation:
temps.chunk_while { |a, b| b >= a }.map(&:inspect).each { |r| puts r }
# [23, 24, 24, 25]   (each >= previous)
# [21, 22, 23, 30]   (each >= previous)
# [29, 28]           (wait, 28 < 29, so breaks)
# Actually:
# [23, 24, 24, 25]  (25 → 21: break)
# [21, 22, 23, 30]  (30 → 29: break)
# [29]              (29 → 28: break)
# [28]
```

---

## Scenario 9 — Set lookup performance

```ruby
require 'set'
require 'benchmark'

valid_codes = (1..10_000).map { |n| "CODE#{n}" }
test_code   = "CODE9999"

arr_time = Benchmark.realtime { 10_000.times { valid_codes.include?(test_code) } }
set_data  = Set.new(valid_codes)
set_time  = Benchmark.realtime { 10_000.times { set_data.include?(test_code) } }

puts "Array: #{'%.4f' % arr_time}s"
puts "Set:   #{'%.4f' % set_time}s"
```

**Expected:** Set is significantly faster (O(1) vs O(n)). For 10,000 checks on a 10,000 element collection, Set will be 100-1000x faster.

---

## Scenario 10 — tally and most frequent element

```ruby
votes = ["ruby", "python", "ruby", "go", "ruby", "python", "go", "ruby", "python"]

tally = votes.tally
puts tally.inspect
winner = tally.max_by { |_, count| count }
puts "Winner: #{winner[0]} with #{winner[1]} votes"

# Also find top 3:
top3 = tally.sort_by { |_, count| -count }.first(3)
top3.each_with_index do |(lang, count), i|
  puts "#{i+1}. #{lang}: #{count}"
end
```

**Answer:**
```
{"ruby"=>4, "python"=>3, "go"=>2}
Winner: ruby with 4 votes
1. ruby: 4
2. python: 3
3. go: 2
```

---

## Scenario 11 — each_slice for batch processing

```ruby
user_ids = (1..25).to_a

user_ids.each_slice(10) do |batch|
  puts "Processing batch of #{batch.size}: #{batch.first}-#{batch.last}"
  # process_batch(batch) — e.g., bulk DB query
end
```

**Answer:**
```
Processing batch of 10: 1-10
Processing batch of 10: 11-20
Processing batch of 5: 21-25
```

`each_slice` is the idiomatic Ruby way to process large collections in fixed-size batches — commonly used for bulk API calls, database batch inserts, or rate-limited processing.

---

## Scenario 12 — filter_map for safe type conversion

```ruby
raw_data = ["42", "not_a_number", "100", nil, "7", "", "55"]

numbers = raw_data.filter_map do |s|
  Integer(s) rescue nil
end

puts numbers.inspect
puts numbers.sum
```

**Answer:**
```
[42, 100, 7, 55]
204
```

`filter_map` is perfect for this pattern: attempt conversion, return `nil` on failure (via `rescue nil`), and `filter_map` automatically removes `nil` values.

---

## Scenario 13 — sort_by with multiple criteria

```ruby
students = [
  { name: "Alice", grade: "B", age: 22 },
  { name: "Bob",   grade: "A", age: 25 },
  { name: "Carol", grade: "A", age: 22 },
  { name: "Dave",  grade: "B", age: 20 }
]

sorted = students.sort_by { |s| [s[:grade], s[:age], s[:name]] }
sorted.each { |s| puts "#{s[:name]}: #{s[:grade]}, #{s[:age]}" }
```

**Answer:**
```
Carol: A, 22
Bob: A, 25
Dave: B, 20
Alice: B, 22
```

Array comparison in Ruby compares element by element: first by grade (alphabetical), then by age, then by name — all ascending.

---

## Scenario 14 — Enumerator chaining

```ruby
result = (1..Float::INFINITY)
  .lazy
  .each_with_object([]) do |n, arr|  # DOES NOT WORK with lazy
    arr << n * 2
    arr
  end

# Why doesn't this work?
```

**Answer:** `each_with_object` does not work with lazy enumerators because it's not a lazy method — it would need to consume the infinite sequence. The fix:

```ruby
# Use take + map instead:
(1..Float::INFINITY).lazy.map { |n| n * 2 }.first(10)
# => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# Or use take to limit first:
(1..Float::INFINITY).lazy.take(10).map { |n| n * 2 }.to_a
# => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

---

## Scenario 15 — Building an inverted index

```ruby
documents = {
  doc1: "the quick brown fox",
  doc2: "the lazy dog",
  doc3: "quick brown dog"
}

# Build an inverted index: word => [doc_ids that contain it]
inverted_index = documents.each_with_object(Hash.new { |h,k| h[k] = [] }) do
  |(doc_id, text), index|
  text.split.each { |word| index[word] << doc_id }
end

puts inverted_index["quick"].inspect
puts inverted_index["dog"].inspect
puts inverted_index["the"].inspect
```

**Answer:**
```
[:doc1, :doc3]
[:doc2, :doc3]
[:doc1, :doc2]
```

This demonstrates `each_with_object` with a `Hash.new` default block, and nested iteration over both the hash and each word in the text.
