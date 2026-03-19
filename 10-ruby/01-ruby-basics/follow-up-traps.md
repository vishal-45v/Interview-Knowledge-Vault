# Chapter 01 — Ruby Basics: Follow-Up Traps

These are the gotchas and follow-up questions that interviewers use to separate candidates who understand Ruby from those who have just read a tutorial.

---

## Trap 1 — nil converts to safe defaults silently

```ruby
nil.to_s    # => ""     (empty string, not "nil")
nil.to_a    # => []     (empty array)
nil.to_i    # => 0
nil.to_f    # => 0.0
nil.to_h    # => {}     (empty hash)

# This is used defensively everywhere:
user = find_user(id)
name = user&.name.to_s   # "" if user is nil, rather than NoMethodError

# Gotcha: nil.to_s is "" but p nil prints "nil"
nil.to_s        # => ""
nil.inspect     # => "nil"
puts nil        # (blank line — prints "")
p nil           # nil    (prints inspect output)
```

**Interviewer follow-up:** "What does `[nil, 1, nil, 2].map(&:to_i)` return?"
**Answer:** `[0, 1, 0, 2]` — each nil becomes 0.

---

## Trap 2 — Symbol vs String as hash key (the silent nil trap)

```ruby
config = { host: "localhost", port: 5432 }

# These look the same but are different keys!
config[:host]    # => "localhost"
config["host"]   # => nil  (different key!)

# Coming from JSON.parse:
require 'json'
json_data = JSON.parse('{"host": "localhost"}')
json_data[:host]    # => nil    (JSON.parse returns string keys)
json_data["host"]   # => "localhost"

# Fix 1: symbolize keys manually
json_data.transform_keys(&:to_sym)[:host]  # => "localhost"

# Fix 2: JSON.parse with symbolize_names
JSON.parse('{"host": "localhost"}', symbolize_names: true)[:host]
# => "localhost"
```

**Interviewer follow-up:** "What happens when you use both string and symbol versions of the same key?"
```ruby
h = {}
h[:name] = "Alice"
h["name"] = "Bob"
h  # => {name: "Alice", "name" => "Bob"}  — two separate keys!
```

---

## Trap 3 — Three equality operators with different semantics

```ruby
# == (value equality, commonly overridden)
1 == 1.0        # => true   Integer and Float compare by value
"a" == "a"      # => true   same content

# eql? (strict value equality — same type AND value)
1.eql?(1.0)     # => false  different types
1.eql?(1)       # => true
"a".eql?("a")   # => true   String#eql? compares content

# equal? (identity — exact same object, like object_id check)
"a".equal?("a")    # => false  two different string objects
a = "hello"
b = a
a.equal?(b)        # => true   same object

# Hash uses eql? and hash to determine key equality
# This is why 1 and 1.0 are different hash keys:
{1 => "int", 1.0 => "float"}
# => {1=>"int", 1.0=>"float"}   two keys!
```

**Interviewer follow-up:** "Why does Ruby have three equality methods?"
- `==` is for human/business logic equality (overrideable)
- `eql?` is for hash key comparison (strict)
- `equal?` is for object identity (never override this)

---

## Trap 4 — Frozen string mutation raises RuntimeError/FrozenError

```ruby
str = "hello".freeze
str << " world"     # => FrozenError: can't modify frozen String

# Symbols are always frozen
:hello.frozen?      # => true
:hello << "x"       # => NoMethodError: undefined method '<<' for :hello

# Integer/Float are always frozen too
42.frozen?          # => true

# Freeze is SHALLOW for containers
arr = ["a", "b"].freeze
arr << "c"          # => FrozenError (can't add to frozen array)
arr[0] << "X"       # => Works! arr => ["aX", "b"]
                    # The array is frozen, not the strings inside it

# Deep freeze (not built-in, need recursive approach)
def deep_freeze(obj)
  obj.freeze
  case obj
  when Array then obj.each { |e| deep_freeze(e) }
  when Hash  then obj.each { |k, v| deep_freeze(k); deep_freeze(v) }
  end
  obj
end
```

---

## Trap 5 — Integer division truncates toward zero

```ruby
7 / 2       # => 3    (NOT 3.5)
-7 / 2      # => -4   (rounds toward negative infinity in Ruby!)
7 / -2      # => -4

# Compare with truncation vs floor division:
# C/Java: -7 / 2 = -3  (truncation toward zero)
# Ruby:   -7 / 2 = -4  (floor division)

7.0 / 2     # => 3.5   (float result)
7.fdiv(2)   # => 3.5   (explicit float division)
7.0.ceil    # => 7     (not useful here, showing ceil/floor)

# Modulo follows floor division:
-7 % 2      # => 1   (NOT -1)
# Because: -7 = (-4 * 2) + 1, floor quotient is -4

# remainder uses truncation:
-7.remainder(2)  # => -1  (truncation toward zero)
```

---

## Trap 6 — puts nil prints blank line; p nil prints "nil"

```ruby
puts nil      # prints a blank line (nil.to_s == "")
p nil         # prints nil  (nil.inspect == "nil")
print nil     # prints nothing

# Array with nil:
puts [1, nil, 3]   # prints: 1\n\n3  (nil becomes blank line)
p [1, nil, 3]      # prints: [1, nil, 3]  (shows structure)

# This matters when logging or debugging:
# puts user  => might show blank lines if user is nil
# p user     => shows nil explicitly
```

---

## Trap 7 — Array#flatten vs Array#flatten(1)

```ruby
nested = [1, [2, [3, [4]]]]

nested.flatten      # => [1, 2, 3, 4]      (full depth)
nested.flatten(1)   # => [1, 2, [3, [4]]]  (one level)
nested.flatten(2)   # => [1, 2, 3, [4]]    (two levels)

# flatten! returns nil if no change needed:
already_flat = [1, 2, 3]
result = already_flat.flatten!   # => nil  (nothing to flatten)
already_flat                     # => [1, 2, 3]  (unchanged)

# Safe pattern:
def normalize(arr)
  arr.flatten || arr   # if flatten! returns nil, use original
end
```

---

## Trap 8 — gsub vs gsub! — bang methods return nil if no match

```ruby
str = "hello world"

str.gsub("x", "y")   # => "hello world"  (new string, no match is fine)
str.gsub!("x", "y")  # => nil            (returns nil if nothing changed)
str                   # => "hello world"  (unchanged)

# Dangerous chaining:
"hello".gsub!("z", "q").upcase  # => NoMethodError: nil.upcase

# Safe pattern 1: use non-bang
result = str.gsub("x", "y")

# Safe pattern 2: use || for fallback
str.gsub!("x", "y") || str   # returns str if gsub! returns nil

# This applies to ALL bang methods: sub!, strip!, chomp!, etc.
"hello".strip!   # => nil (nothing to strip, returns nil)
"  hi  ".strip!  # => "hi" (stripped, returns the string)
```

---

## Trap 9 — case/when uses === not ==

```ruby
# The case equality operator is the "smart match"
x = "hello"
case x
when "hello"     then "exact match"
when /^h/        then "regex match (won't reach here)"
when String      then "class match (won't reach here)"
end

# Under the hood:
"hello" === "hello"  # => true  (String#=== uses ==)
/^h/    === "hello"  # => true  (Regexp#=== does =~ match)
String  === "hello"  # => true  (Module#=== does is_a?)
(1..10) === 5        # => true  (Range#=== does include?)
:sym    === :sym     # => true  (Symbol#=== uses ==)

# Trap: the order is important
x = 15
case x
when Integer then "integer"    # matches first
when 10..20  then "in range"   # never reached
end
# => "integer"

# If you want the range check first:
case x
when 10..20  then "in range"   # now matches first
when Integer then "integer"
end
# => "in range"
```

---

## Trap 10 — 0 is truthy (not falsy!)

```ruby
# Only nil and false are falsy — 0 is TRUTHY
if 0
  puts "zero is truthy!"  # this DOES print
end

# Common mistake from Python/JS background:
count = 0
if count
  puts "has count"   # prints even when count is 0!
end

# Correct check for zero:
if count.zero?    then "is zero"
elsif count > 0   then "positive"
end

# Or:
if count == 0 then ... end

# The ||= trap:
total = 0
total ||= 10   # total remains 0! (0 is truthy, ||= doesn't replace it)

# vs nil:
total = nil
total ||= 10   # total becomes 10 (nil is falsy, ||= replaces it)
```

---

## Trap 11 — String#* vs Array#* (multiplication operator)

```ruby
"ha" * 3      # => "hahaha"
[1, 2] * 3    # => [1, 2, 1, 2, 1, 2]  (repeats the array)
[1, 2] * ", " # => "1, 2"  (join!)

# Common confusion:
["a", "b", "c"] * " | "  # => "a | b | c"  (same as join)

# But this doesn't work on ranges:
(1..3) * 2    # => NoMethodError: undefined method '*' for Range
```

---

## Trap 12 — Block-local variables

```ruby
x = 10

[1, 2, 3].each do |n; x|   # x is block-local (semicolon syntax)
  x = n * 100               # does NOT affect outer x
  puts x
end

puts x   # => 10  (outer x unchanged)

# Without ;x — outer x IS affected:
[1, 2, 3].each do |n|
  x = n * 100   # overwrites outer x!
end
puts x   # => 300  (last value assigned)
```

---

## Trap 13 — require vs require_relative path resolution

```ruby
# require searches $LOAD_PATH (like Ruby's PATH for libraries)
require 'json'       # looks in gem paths
require 'my_gem'     # fails if not in $LOAD_PATH

# require_relative resolves from the current file's directory
# In /project/lib/foo.rb:
require_relative 'bar'           # loads /project/lib/bar.rb
require_relative '../config/db'  # loads /project/config/db.rb

# Gotcha: require_relative uses __FILE__, not Dir.pwd
# If your script is run from a different directory, require_relative still
# works because it's relative to the FILE, not the shell's working directory

# load always re-executes (even if already required)
load 'config.rb'          # runs every time
require 'config'          # runs only once

# Check what's loaded:
$LOADED_FEATURES.grep(/json/)  # find json in load path
```
