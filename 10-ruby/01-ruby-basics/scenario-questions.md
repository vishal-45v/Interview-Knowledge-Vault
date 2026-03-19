# Chapter 01 — Ruby Basics: Scenario Questions

## Scenario 1 — What does this code print?

```ruby
x = 10
def test
  x = 20
  puts x
end
test
puts x
```

**Answer:** Prints `20` then `10`. Methods create a new scope. The `x` inside `test` is a completely different local variable from the outer `x`. Ruby does not have block-level scoping for methods — methods always create a fresh local scope.

---

## Scenario 2 — Symbol vs String hash key trap

```ruby
user = { "name" => "Alice", "age" => 30 }
puts user[:name]
puts user["name"]
```

**Answer:** First puts prints `nil`. Second puts prints `"Alice"`. Symbol `:name` and string `"name"` are different hash keys. This is a common source of bugs when mixing JSON-parsed data (string keys) with Ruby hashes (often symbol keys).

**Fix:**
```ruby
# Convert all keys to symbols
user.transform_keys(&:to_sym)[:name]  # => "Alice"

# Or use HashWithIndifferentAccess (Rails)
require 'active_support/hash_with_indifferent_access'
user = ActiveSupport::HashWithIndifferentAccess.new("name" => "Alice")
user[:name]    # => "Alice"
user["name"]   # => "Alice"
```

---

## Scenario 3 — What is the output?

```ruby
arr = [1, 2, nil, 3, nil, 4]
puts arr.compact.sum
```

**Answer:** Prints `10`. `compact` removes all `nil` values, returning `[1, 2, 3, 4]`. Then `sum` adds them: `1 + 2 + 3 + 4 = 10`.

---

## Scenario 4 — Integer division trap

```ruby
total = 7
count = 2
average = total / count
puts average
puts average.class
```

**Answer:** Prints `3` and `Integer`. Ruby performs integer division when both operands are integers. The decimal part is truncated, not rounded. To get a float:

```ruby
average = total.to_f / count   # => 3.5
average = total / count.to_f   # => 3.5
average = total.fdiv(count)    # => 3.5  — cleanest Ruby idiom
```

---

## Scenario 5 — Frozen string mutation

```ruby
# frozen_string_literal: true

greeting = "Hello"
greeting << ", World!"
puts greeting
```

**Answer:** This raises `FrozenError: can't modify frozen String: "Hello"`. The magic comment `# frozen_string_literal: true` at the top of the file freezes all string literals. To build strings dynamically:

```ruby
# frozen_string_literal: true

greeting = +"Hello"    # + prefix creates a mutable string
greeting << ", World!"
puts greeting  # => "Hello, World!"

# Or use string interpolation (creates new unfrozen string)
base = "Hello"
result = "#{base}, World!"
```

---

## Scenario 6 — nil.to_s vs p nil vs puts nil

```ruby
puts nil.inspect
puts nil.to_s.inspect
p nil
puts nil
print nil
```

**Answer:**
```
"nil"        # nil.inspect returns the string "nil"
""           # nil.to_s returns empty string ""
nil          # p nil prints nil (using inspect)
             # puts nil prints blank line (using to_s which is "")
             # print nil prints nothing
```

---

## Scenario 7 — String gsub! returning nil

```ruby
def sanitize(str)
  str.gsub!(/[^a-z]/i, "")
  str
end

result = sanitize("Hello, World! 123")
puts result
```

**Answer:** Prints `HelloWorld`. `gsub!` mutates the string in-place and returns the modified string... **unless no substitutions were made**, in which case it returns `nil`. This can cause `NoMethodError` if you chain methods on the result:

```ruby
# Dangerous pattern:
"hello".gsub!(/x/, "y").upcase  # => NoMethodError (nil.upcase)

# Safe alternatives:
str.gsub!(/pattern/, "replacement") || str  # returns str if no match
str.gsub(/pattern/, "replacement")           # non-bang always returns string
```

---

## Scenario 8 — Understanding each_with_object

```ruby
words = ["apple", "banana", "cherry", "avocado", "blueberry"]

grouped = words.each_with_object(Hash.new([])) do |word, memo|
  memo[word[0]] += [word]
end

puts grouped.inspect
```

**Answer:** This produces unexpected results with `Hash.new([])` because all keys share the same default array object. The correct approach:

```ruby
# Correct: Hash.new { |h, k| h[k] = [] }
grouped = words.each_with_object(Hash.new { |h, k| h[k] = [] }) do |word, memo|
  memo[word[0]] << word
end
# => {"a"=>["apple", "avocado"], "b"=>["banana", "blueberry"], "c"=>["cherry"]}

# Alternative using group_by
grouped = words.group_by { |w| w[0] }
# => {"a"=>["apple", "avocado"], "b"=>["banana", "blueberry"], "c"=>["cherry"]}
```

---

## Scenario 9 — case/when uses ===

```ruby
value = "hello"

case value
when String  then puts "it's a String"
when Integer then puts "it's an Integer"
when /^h/    then puts "starts with h"
end
```

**Answer:** Prints `it's a String`. The `when` clause uses `===` (the case equality operator), not `==`. For classes, `String === "hello"` calls `String.===("hello")` which is equivalent to `"hello".is_a?(String)`. Since the first match wins, "starts with h" is never reached (even though it would also match).

```ruby
# === is polymorphic:
String  === "hi"    # => true   (is_a? check)
/^h/    === "hi"    # => true   (regex match)
(1..10) === 5       # => true   (range inclusion)
:foo    === :foo    # => true   (equality)
```

---

## Scenario 10 — Variable scope in blocks

```ruby
total = 0
[1, 2, 3, 4, 5].each do |n|
  total += n
end
puts total
```

**Answer:** Prints `15`. Unlike methods, blocks share the enclosing scope — `total` is the same variable inside and outside the block. This is the core of how Ruby's `reduce`/`inject` pattern works with an external accumulator.

---

## Scenario 11 — Array multiplication

```ruby
arr = [1, 2, 3]
puts (arr * 3).inspect
puts (arr * ", ")
```

**Answer:**
```
[1, 2, 3, 1, 2, 3, 1, 2, 3]  # Array * Integer repeats the array
1, 2, 3                        # Array * String joins with separator
```

---

## Scenario 12 — Multiple return values

```ruby
def minmax(arr)
  [arr.min, arr.max]
end

low, high = minmax([3, 1, 4, 1, 5, 9, 2, 6])
puts "#{low} to #{high}"
```

**Answer:** Prints `1 to 9`. Ruby supports multiple return values via array decomposition (destructuring assignment). This is idiomatic Ruby — methods often return arrays for multiple values.

---

## Scenario 13 — Chaining string operations

```ruby
result = "  Hello, World!  "
  .strip
  .downcase
  .gsub(",", "")
  .split(" ")
  .map(&:capitalize)
  .join(" ")

puts result
```

**Answer:** Prints `Hello World`. Walking through the chain:
1. `strip` → `"Hello, World!"`
2. `downcase` → `"hello, world!"`
3. `gsub(",", "")` → `"hello world!"`
4. `split(" ")` → `["hello", "world!"]`
5. `map(&:capitalize)` → `["Hello", "World!"]`
6. `join(" ")` → `"Hello World!"`

---

## Scenario 14 — Hash merge with conflict block

```ruby
user_prefs  = { theme: "dark", font_size: 14, language: "en" }
admin_prefs = { theme: "light", notifications: true }

merged = user_prefs.merge(admin_prefs) do |key, user_val, admin_val|
  puts "Conflict on #{key}: user=#{user_val}, admin=#{admin_val}"
  user_val  # user preference wins
end

puts merged.inspect
```

**Answer:**
```
Conflict on theme: user=dark, admin=light
{:theme=>"dark", :font_size=>14, :language=>"en", :notifications=>true}
```

The block is called only for conflicting keys. User value `"dark"` is kept for `:theme`. The non-conflicting `:notifications` key is added without calling the block.

---

## Scenario 15 — Range and Array conversion

```ruby
r = (1..10)
puts r.select(&:odd?).inspect
puts r.sum
puts r.include?(10)
puts r.to_a.length
```

**Answer:**
```
[1, 3, 5, 7, 9]
55
true
10
```

Ranges are lazy by default — `select` enumerates them. `sum` computes 1+2+...+10 = 55. `include?` is O(1) for ranges (just boundary check). `to_a` materializes the range into an array.

---

## Scenario 16 — Truthiness in conditions

```ruby
[0, "", [], {}, false, nil].each do |val|
  if val
    puts "#{val.inspect} is truthy"
  else
    puts "#{val.inspect} is falsy"
  end
end
```

**Answer:**
```
0 is truthy
"" is truthy
[] is truthy
{} is truthy
false is falsy
nil is falsy
```

Only `false` and `nil` are falsy. This surprises developers from Python, JavaScript, or C backgrounds.

---

## Scenario 17 — Heredoc with interpolation

```ruby
name = "Alice"
age  = 30

message = <<~MSG
  Dear #{name},

  You are #{age} years old.
  In 5 years you'll be #{age + 5}.
MSG

puts message.lines.count
puts message.include?("Alice")
```

**Answer:** Prints `5` (4 content lines + trailing newline creates 5 lines including the empty line). Prints `true`. The squiggly heredoc `<<~MSG` strips leading whitespace based on the least-indented line, making it work cleanly inside indented code.

---

## Scenario 18 — Chained array operations with nil

```ruby
data = ["alice", nil, "bob", "", "carol", nil]
result = data
  .compact
  .reject(&:empty?)
  .map(&:capitalize)
  .sort

puts result.inspect
```

**Answer:** `["Alice", "Bob", "Carol"]`. Step by step:
1. `compact` → `["alice", "bob", "", "carol"]`
2. `reject(&:empty?)` → `["alice", "bob", "carol"]`
3. `map(&:capitalize)` → `["Alice", "Bob", "Carol"]`
4. `sort` → `["Alice", "Bob", "Carol"]` (already sorted)

---

## Scenario 19 — Conditional assignment operators

```ruby
a = nil
b = false
c = 0
d = "hello"

a ||= "default_a"
b ||= "default_b"
c ||= "default_c"
d ||= "default_d"

puts a.inspect  # ?
puts b.inspect  # ?
puts c.inspect  # ?
puts d.inspect  # ?
```

**Answer:**
```
"default_a"    # nil is falsy, gets assigned
"default_b"    # false is falsy, gets assigned
0              # 0 is TRUTHY, NOT reassigned
"hello"        # "hello" is truthy, NOT reassigned
```

`||=` only assigns when the left side is falsy (nil or false). This is a critical distinction from languages where 0 is falsy.

---

## Scenario 20 — puts vs p on arrays

```ruby
arr = ["Alice", "Bob", "Carol"]
puts arr
puts "---"
p arr
puts "---"
print arr
```

**Answer:**
```
Alice          # puts on array prints each element on its own line
Bob
Carol
---
["Alice", "Bob", "Carol"]   # p uses inspect — shows the array structure
---
["Alice", "Bob", "Carol"]   # print calls to_s on array (same as inspect for arrays)
```

---

## Scenario 21 — String scan vs match

```ruby
text = "The price is $12.99 and $8.50 for two items"

prices_scan  = text.scan(/\$\d+\.\d+/)
first_match  = text.match(/\$(\d+\.\d+)/)

puts prices_scan.inspect
puts first_match[0]
puts first_match[1]
```

**Answer:**
```
["$12.99", "$8.50"]    # scan returns ALL matches as array
"$12.99"               # match[0] is full match
"12.99"                # match[1] is first capture group
```

`scan` finds all non-overlapping matches. `match` finds only the first and returns a MatchData object with capture groups accessible via numeric indices.

---

## Scenario 22 — respond_to? and duck typing

```ruby
def process(obj)
  if obj.respond_to?(:each)
    obj.each { |item| puts item }
  elsif obj.respond_to?(:to_s)
    puts obj.to_s
  end
end

process([1, 2, 3])
process("hello")
process(42)
```

**Answer:**
```
1
2
3
hello
42
```

This is duck typing — we don't check `obj.is_a?(Array)`, we check if it can do what we need (`each`). Strings respond to `each_char` but not `each`, so they fall to `to_s`. Integers respond to `to_s`.
