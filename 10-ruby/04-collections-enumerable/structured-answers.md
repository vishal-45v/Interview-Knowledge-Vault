# Chapter 04 — Collections & Enumerable: Structured Answers

---

## Answer 1 — "Walk me through how Enumerable works and why it's powerful"

Enumerable is a module that provides ~50 collection methods to any class that defines `each`. It's one of Ruby's best examples of the mixin pattern.

```ruby
# Minimal implementation: define only #each
class NumberBag
  include Enumerable

  def initialize(*numbers)
    @numbers = numbers
  end

  def each(&block)
    @numbers.each(&block)
  end
end

bag = NumberBag.new(5, 3, 8, 1, 9, 2, 7)

# Now we get ALL Enumerable methods for free:
bag.sort              # => [1, 2, 3, 5, 7, 8, 9]
bag.min               # => 1
bag.max               # => 9
bag.select(&:odd?)    # => [5, 3, 1, 9, 7]
bag.map { |n| n * 2 } # => [10, 6, 16, 2, 18, 4, 14]
bag.sum               # => 35
bag.count { |n| n > 5 }  # => 3
bag.include?(8)       # => true
bag.all? { |n| n > 0 }   # => true
bag.any? { |n| n > 8 }   # => true
bag.none? { |n| n > 10 } # => true
bag.first(3)          # => [5, 3, 8]
bag.to_a              # => [5, 3, 8, 1, 9, 2, 7]
```

**The "each is enough" principle:** Ruby's standard library is built around this contract. `Array`, `Hash`, `Range`, `IO` — they all implement `each` and include `Enumerable`. Your custom classes can join this ecosystem with just one method.

---

## Answer 2 — "Explain the difference between map, flat_map, and each_with_object"

```ruby
data = [
  { user: "Alice", tags: ["ruby", "rails"] },
  { user: "Bob",   tags: ["python", "django"] },
  { user: "Carol", tags: ["ruby", "go"] }
]

# map — one-to-one transformation, returns array of same size
names = data.map { |d| d[:user] }
# => ["Alice", "Bob", "Carol"]

# flat_map — map then flatten one level
# Use when each element maps to an array and you want one flat result
all_tags = data.flat_map { |d| d[:tags] }
# => ["ruby", "rails", "python", "django", "ruby", "go"]

# vs map (returns array of arrays):
data.map { |d| d[:tags] }
# => [["ruby", "rails"], ["python", "django"], ["ruby", "go"]]

# flat_map is equivalent to .map { }.flatten(1)
# but more efficient (single pass)

# each_with_object — builds up a result object
tag_users = data.each_with_object(Hash.new { |h,k| h[k] = [] }) do |d, index|
  d[:tags].each { |tag| index[tag] << d[:user] }
end
# => {"ruby"=>["Alice","Carol"], "rails"=>["Alice"], "python"=>["Bob"],
#     "django"=>["Bob"], "go"=>["Carol"]}
```

---

## Answer 3 — "How would you efficiently group, sort, and aggregate a large dataset?"

```ruby
# Given: array of order hashes
orders = [
  { id: 1, customer: "Alice", product: "Widget",  qty: 3,  price: 9.99 },
  { id: 2, customer: "Bob",   product: "Gadget",  qty: 1,  price: 24.99 },
  { id: 3, customer: "Alice", product: "Gadget",  qty: 2,  price: 24.99 },
  { id: 4, customer: "Carol", product: "Widget",  qty: 5,  price: 9.99 },
  { id: 5, customer: "Bob",   product: "Doohickey",qty: 1, price: 4.99 }
]

# Task: customer revenue report, sorted by total descending
report = orders
  .each_with_object(Hash.new(0)) { |order, totals|
    totals[order[:customer]] += order[:qty] * order[:price]
  }
  .sort_by { |customer, total| -total }
  .map { |customer, total| "#{customer}: $#{'%.2f' % total}" }

puts report.join("\n")
# => Alice: $79.95
# => Carol: $49.95
# => Bob: $29.98

# Task: products by total units sold
product_sales = orders
  .group_by  { |o| o[:product] }
  .transform_values { |product_orders| product_orders.sum { |o| o[:qty] } }
  .sort_by   { |_, qty| -qty }

product_sales.each { |product, qty| puts "#{product}: #{qty} units" }
# Widget: 8 units
# Gadget: 3 units
# Doohickey: 1 unit
```

---

## Answer 4 — "Explain the Set class and when O(1) lookup matters"

```ruby
require 'set'

# Problem: membership testing in a large collection
# Array#include? is O(n) — linear scan
# Set#include? is O(1) — hash-based lookup

# Benchmark demonstrates the difference:
require 'benchmark'

n = 100_000
data = (1..n).to_a.shuffle
lookup_val = n / 2  # somewhere in the middle

arr = data.dup
set = Set.new(data)

Benchmark.bm(15) do |b|
  b.report("Array#include?:") { 10_000.times { arr.include?(lookup_val) } }
  b.report("Set#include?:  ") { 10_000.times { set.include?(lookup_val) } }
end

# Real-world use cases:
# 1. Deduplication during import
seen_ids = Set.new
records.select { |r| seen_ids.add?(r[:id]) }  # add? returns nil if already present

# 2. Access control whitelist
ALLOWED_ROLES = Set.new(["admin", "moderator", "editor"]).freeze
def authorized?(role)
  ALLOWED_ROLES.include?(role)  # O(1)
end

# 3. Stop-word filtering in text processing
STOP_WORDS = Set.new(%w[the a an is are was were be been]).freeze
words.reject { |w| STOP_WORDS.include?(w) }  # O(1) per word
```

---

## Answer 5 — "How would you implement a word frequency counter using Ruby collections?"

```ruby
text = """
  to be or not to be that is the question
  whether tis nobler in the mind to suffer
  the slings and arrows of outrageous fortune
"""

word_freq = text
  .downcase
  .scan(/\b[a-z]+\b/)          # extract all words
  .tally                        # count each word
  .sort_by { |_, count| -count } # sort by frequency descending
  .first(10)                    # top 10

puts word_freq.map { |word, count| "#{word}: #{count}" }.join("\n")
# the: 3
# to: 3
# be: 2
# or: 1
# ...

# More detailed analysis with each_with_object:
analysis = text
  .downcase
  .scan(/\b[a-z]+\b/)
  .each_with_object({ total: 0, unique: Set.new, freq: Hash.new(0) }) do |word, acc|
    acc[:total] += 1
    acc[:unique].add(word)
    acc[:freq][word] += 1
  end

puts "Total words: #{analysis[:total]}"
puts "Unique words: #{analysis[:unique].size}"
puts "Most frequent: #{analysis[:freq].max_by { |_, c| c }}"
```

---

## Answer 6 — "Explain lazy enumerators with a real pipeline example"

```ruby
# Real scenario: processing a large log file
# Find the first 10 unique IPs that caused errors

class LogProcessor
  def initialize(file_path)
    @file_path = file_path
  end

  def error_ips(limit: 10)
    seen = Set.new

    File.foreach(@file_path)
      .lazy
      .select  { |line| line.include?("ERROR") }
      .map     { |line| extract_ip(line) }
      .reject  { |ip|   ip.nil? }
      .select  { |ip|   seen.add?(ip) }  # add? returns nil if already in set
      .first(limit)
  end

  private

  def extract_ip(line)
    line.match(/\b(\d{1,3}(?:\.\d{1,3}){3})\b/)&.captures&.first
  end
end

# This approach:
# 1. Reads the file line-by-line (doesn't load the whole file)
# 2. Stops after finding 10 unique error IPs
# 3. Deduplicates with Set (O(1) lookup)
# 4. Never creates large intermediate arrays

# Without lazy:
# File.readlines(path).select { }.map { }.reject { }.select { }.first(10)
# Reads the ENTIRE file into memory before any filtering
```

---

## Answer 7 — "When would you choose inject over each_with_object?"

```ruby
# Use inject when:
# - Result is a single scalar value
# - You prefer functional style
# - The accumulator changes TYPE between iterations

# Scalar accumulation:
[1, 2, 3, 4, 5].inject(:+)        # => 15 (sum)
[1, 2, 3, 4, 5].inject(:*)        # => 120 (product)
["a","b","c"].inject("") { |s, c| s + c }  # => "abc" (string concat)

# Factorial:
(1..10).inject(:*)   # => 3628800

# Maximum without #max (educational):
[3,1,4,1,5,9,2,6].inject { |max, n| n > max ? n : max }  # => 9

# Use each_with_object when:
# - Building a mutable container (Hash, Array)
# - You hate returning memo explicitly

# Building index:
words = ["apple", "banana", "apricot", "avocado"]
index = words.each_with_object(Hash.new { |h,k| h[k] = [] }) do |word, h|
  h[word[0]] << word
end
# => {"a"=>["apple","apricot","avocado"], "b"=>["banana"]}

# Same with inject (must return memo):
index = words.inject(Hash.new { |h,k| h[k] = [] }) do |h, word|
  h[word[0]] << word
  h  # must return h!
end

# Rule of thumb:
# - Building a container? Use each_with_object (can't forget memo)
# - Computing a scalar? Use inject (cleaner with symbol shorthand)
```
