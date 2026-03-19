# Chapter 01 — Ruby Basics: Theory Questions

## Everything Is an Object

**Q1. What does "everything is an object" mean in Ruby?**

In Ruby, every value — including integers, strings, nil, true, and false — is an instance of a class and responds to methods. There are no primitive types.

```ruby
1.class          # => Integer
"hello".class    # => String
nil.class        # => NilClass
true.class       # => TrueClass
false.class      # => FalseClass
:sym.class       # => Symbol
[].class         # => Array
{}.class         # => Hash

# Even integers respond to methods
3.times { print "ho " }   # => ho ho ho
-5.abs                     # => 5
42.to_s                    # => "42"
```

**Q2. What is an object_id in Ruby and when is it useful?**

Every object has a unique `object_id`. For symbols and small integers, the object_id is fixed and deterministic. Two variables pointing to the same object share the same object_id.

```ruby
:hello.object_id == :hello.object_id   # => true  (symbols are interned)
"hello".object_id == "hello".object_id # => false  (new string each time)

a = "test"
b = a
a.object_id == b.object_id             # => true   (same object)

# Integers: object_id = (n * 2) + 1
1.object_id   # => 3
2.object_id   # => 5
```

---

## Variables

**Q3. What are the five types of variables in Ruby?**

```ruby
x = 10            # local variable — lowercase or underscore prefix
@name = "Alice"   # instance variable — belongs to object instance
@@count = 0       # class variable — shared across class and subclasses
$global = true    # global variable — accessible everywhere (avoid in practice)
MAX = 100         # constant — ALL_CAPS or CamelCase, warning if reassigned
```

**Q4. What is the default value of an uninitialized instance variable?**

```ruby
class Person
  def greet
    puts @name.inspect  # @name was never assigned
  end
end

Person.new.greet  # => nil  (no error — returns nil silently)
```

**Q5. What are constants in Ruby and can they be changed?**

Constants start with an uppercase letter. Ruby issues a warning if you reassign them but does not raise an error. You can force-freeze them.

```ruby
MAX_RETRIES = 3
MAX_RETRIES = 5  # warning: already initialized constant MAX_RETRIES

# Truly immutable constant
DATABASE_URL = "postgres://localhost/mydb".freeze
```

---

## Symbols vs Strings

**Q6. What is a Symbol in Ruby and how does it differ from a String?**

A symbol is an immutable, interned identifier. All references to the same symbol point to the same object in memory. Strings create new objects each time.

```ruby
:status.class         # => Symbol
"status".class        # => String

# Symbols are immutable
:hello.frozen?        # => true

# Same symbol = same object
:foo.object_id == :foo.object_id   # => true

# Strings are not interned by default
"foo".object_id == "foo".object_id # => false

# Conversion
"active".to_sym   # => :active
:active.to_s      # => "active"
```

**Q7. When should you use a Symbol vs a String as a Hash key?**

Use symbols as hash keys when the key is a known, fixed identifier (most common case). Use strings when the key comes from external data (user input, JSON parsing, etc.).

```ruby
# Symbols as keys (preferred for internal hashes)
config = { host: "localhost", port: 5432 }
config[:host]   # => "localhost"

# String keys (common from JSON.parse)
data = { "name" => "Alice", "age" => 30 }
data["name"]    # => "Alice"

# Common gotcha: symbol key != string key
config["host"]  # => nil  (not the same key!)
config[:host]   # => "localhost"
```

---

## nil vs false vs 0

**Q8. What values are falsy in Ruby?**

Only `nil` and `false` are falsy in Ruby. Everything else — including `0`, `""`, `[]`, `{}` — is truthy.

```ruby
# Falsy
!nil    # => true
!false  # => true

# Truthy — these PASS the if condition
if 0    then "truthy" end   # => "truthy"
if ""   then "truthy" end   # => "truthy"
if []   then "truthy" end   # => "truthy"
if 0.0  then "truthy" end   # => "truthy"

# nil? check
nil.nil?   # => true
0.nil?     # => false
false.nil? # => false
```

**Q9. What is the difference between nil, false, and 0 in practical use?**

```ruby
result = some_method_that_might_return_nil

# Safe navigation operator
result&.upcase         # returns nil if result is nil, avoids NoMethodError

# Nil guard with ||
name = user_input || "default"

# Nil vs false distinction matters
user_logged_in = nil    # unknown/not set
user_logged_in = false  # known: explicitly not logged in
```

---

## Strings

**Q10. What are the different string literal syntaxes in Ruby?**

```ruby
single_quoted = 'No interpolation #{1+1}, no \n escape'
double_quoted = "Interpolation #{1+1} and \n newlines work"

# %q — like single quotes
%q(Hello #{name})   # => "Hello \#{name}"

# %Q or % — like double quotes
name = "World"
%Q(Hello #{name})   # => "Hello World"
%(Hello #{name})    # => "Hello World"

# Heredoc
text = <<~HEREDOC
  Line one
  Line two
HEREDOC

# Squiggly heredoc (<<~) strips leading whitespace based on least-indented line
```

**Q11. What is frozen_string_literal and why does it matter for performance?**

```ruby
# At the top of a file:
# frozen_string_literal: true

# This makes ALL string literals in the file frozen (immutable)
# They are deduplicated in memory — same literal = same object

str = "hello"
str << " world"   # => RuntimeError: can't modify frozen String

# Performance benefit: no new String object allocated for each literal
# Ruby 3.x encourages this — will be default in future versions

# Explicitly frozen string
name = "Alice".freeze
name.frozen?  # => true
```

**Q12. Explain the most important String methods with examples.**

```ruby
s = "  Hello, World!  "

s.strip         # => "Hello, World!"      — removes leading/trailing whitespace
s.chomp         # => "  Hello, World!  " — removes trailing newline only
s.chop          # => "  Hello, World!  " — removes last character always
s.upcase        # => "  HELLO, WORLD!  "
s.downcase      # => "  hello, world!  "
s.capitalize    # => "  hello, world!  " — capitalizes first char of string
s.reverse       # => "  !dlroW ,olleH  "
s.length        # => 18
s.size          # => 18  (alias for length)

# Split and join
"a,b,c".split(",")         # => ["a", "b", "c"]
["a", "b", "c"].join(", ") # => "a, b, c"

# Substitution
"hello world".gsub("l", "r")       # => "herro worrd"  — all occurrences
"hello world".sub("l", "r")        # => "herlo world"  — first occurrence only
"hello world".gsub(/[aeiou]/, "*") # => "h*ll* w*rld"  — with regex

# Check and scan
"hello".include?("ell")            # => true
"hello world".scan(/\w+/)          # => ["hello", "world"]
"hello".match?(/e/)                # => true
"hello".match(/e(l+)/)&.captures  # => ["ll"]

# String formatting
"%-10s %5d" % ["Alice", 42]        # => "Alice          42"
format("Hello, %s! You are %d.", "Bob", 25)  # => "Hello, Bob! You are 25."
```

---

## Integers

**Q13. What integer iteration methods does Ruby provide?**

```ruby
5.times { |i| print "#{i} " }        # => 0 1 2 3 4

1.upto(5) { |i| print "#{i} " }      # => 1 2 3 4 5

5.downto(1) { |i| print "#{i} " }    # => 5 4 3 2 1

1.step(10, 2) { |i| print "#{i} " }  # => 1 3 5 7 9

# Integer arithmetic
10 / 3          # => 3    (integer division — truncates!)
10.0 / 3        # => 3.3333...
10 % 3          # => 1    (modulo)
2 ** 10         # => 1024 (exponentiation)
10.divmod(3)    # => [3, 1]  (quotient and remainder)
-7.abs          # => 7
255.to_s(2)     # => "11111111"  (binary)
255.to_s(16)    # => "ff"        (hex)
```

---

## Ranges

**Q14. What is the difference between `..` and `...` in Ruby ranges?**

```ruby
inclusive = (1..5)    # includes 5
exclusive = (1...5)   # excludes 5

inclusive.to_a   # => [1, 2, 3, 4, 5]
exclusive.to_a   # => [1, 2, 3, 4]

inclusive.include?(5)  # => true
exclusive.include?(5)  # => false

# Ranges work on any Comparable type
('a'..'e').to_a         # => ["a", "b", "c", "d", "e"]
(Date.today..Date.today + 7)  # a week range

# Ranges in case/when
age = 25
case age
when 0..12  then "child"
when 13..17 then "teen"
when 18..64 then "adult"
when 65..   then "senior"
end
# => "adult"

# Endless range (Ruby 2.6+)
(18..).include?(100)  # => true  (18 and above)
```

---

## Arrays

**Q15. What are the core Array manipulation methods?**

```ruby
arr = [3, 1, 4, 1, 5, 9, 2, 6]

# Adding and removing
arr.push(7)          # => [..., 7]  adds to end
arr.pop              # => 7         removes from end
arr.unshift(0)       # => [0, ...]  adds to front
arr.shift            # => 0         removes from front
arr << 99            # => [..., 99] shovel operator (alias push)

# Transformation
arr.sort             # => [1, 1, 2, 3, 4, 5, 6, 9]
arr.sort.reverse     # => [9, 6, 5, 4, 3, 2, 1, 1]
arr.uniq             # => [3, 1, 4, 5, 9, 2, 6]
arr.compact          # removes nil values: [1, nil, 2].compact => [1, 2]
arr.flatten          # => flattens nested arrays: [[1,[2]],3].flatten => [1,2,3]
arr.flatten(1)       # => one level deep: [[1,[2]],3].flatten(1) => [1,[2],3]

# Set operations
[1,2,3] | [2,3,4]   # => [1,2,3,4]  union
[1,2,3] & [2,3,4]   # => [2,3]      intersection
[1,2,3] - [2,3]     # => [1]        difference

# Zip and product
[1,2,3].zip([4,5,6])           # => [[1,4],[2,5],[3,6]]
[1,2].product([3,4])           # => [[1,3],[1,4],[2,3],[2,4]]

# Sample and shuffle
arr.sample          # => random element
arr.shuffle         # => new array with random order
arr.rotate(2)       # => rotates array by 2 positions
arr.take(3)         # => first 3 elements
arr.drop(3)         # => all but first 3

# Querying
arr.count           # => number of elements
arr.count(1)        # => number of 1s
arr.tally           # => {3=>1, 1=>2, 4=>1, ...}  Ruby 2.7+
arr.sum             # => sum of all elements
arr.min / arr.max   # => min/max values
arr.minmax          # => [min, max]
arr.first / arr.last
arr.first(2) / arr.last(2)
```

**Q16. What does Array#zip do and when is it useful?**

```ruby
names  = ["Alice", "Bob", "Carol"]
scores = [95, 87, 92]
grades = ["A", "B", "A"]

names.zip(scores, grades)
# => [["Alice", 95, "A"], ["Bob", 87, "B"], ["Carol", 92, "A"]]

# Build hash from two arrays
Hash[names.zip(scores)]
# => {"Alice"=>95, "Bob"=>87, "Carol"=>92}

# Or with to_h
names.zip(scores).to_h
# => {"Alice"=>95, "Bob"=>87, "Carol"=>92}
```

---

## Hashes

**Q17. What are the most important Hash methods?**

```ruby
h = { name: "Alice", age: 30, role: "admin" }

# Access
h[:name]              # => "Alice"
h.fetch(:name)        # => "Alice"
h.fetch(:missing, "default")   # => "default"
h.fetch(:missing) { |k| "#{k} not found" }  # => "missing not found"

h.keys               # => [:name, :age, :role]
h.values             # => ["Alice", 30, "admin"]
h.to_a               # => [[:name, "Alice"], [:age, 30], [:role, "admin"]]

# Filtering
h.select { |k, v| v.is_a?(String) }  # => {name: "Alice", role: "admin"}
h.reject { |k, v| v.is_a?(Integer) } # => {name: "Alice", role: "admin"}
h.filter_map { |k, v| [k, v.to_s] if v.is_a?(Integer) }.to_h
# => {age: "30"}

# Merging
defaults = { role: "user", active: true }
defaults.merge(h)    # => h values win for duplicate keys
h.merge(defaults)    # => defaults values win for duplicate keys
h.merge(defaults) { |key, old, new_val| old }  # block controls conflict resolution

# Transformation (Ruby 2.4+)
h.transform_values { |v| v.to_s }
# => {name: "Alice", age: "30", role: "admin"}

h.transform_keys { |k| k.to_s }
# => {"name"=>"Alice", "age"=>30, "role"=>"admin"}

# Aggregation
h.any? { |k, v| v == "admin" }   # => true
h.all? { |k, v| v.nil? }         # => false
h.count { |k, v| v.is_a?(String) } # => 2

# each_with_object to build new structure
h.each_with_object({}) { |(k, v), memo| memo[k] = v.to_s }
```

**Q18. How does Hash#each_with_object differ from Hash#inject?**

```ruby
data = { a: 1, b: 2, c: 3 }

# each_with_object: memo object is always the accumulator
result = data.each_with_object({}) do |(key, value), memo|
  memo[key] = value * 10
end
# => {a: 10, b: 20, c: 30}

# inject/reduce: memo is the return value of each iteration
result = data.inject({}) do |memo, (key, value)|
  memo[key] = value * 10
  memo  # must return memo explicitly!
end
# => {a: 10, b: 20, c: 30}
```

---

## Control Flow

**Q19. What are the conditional constructs in Ruby?**

```ruby
# Standard if/elsif/else
if x > 10
  "big"
elsif x > 5
  "medium"
else
  "small"
end

# unless (opposite of if)
unless user.nil?
  puts user.name
end

# One-line modifier syntax
puts "hello" if condition
puts "hello" unless condition

# Ternary
result = x > 0 ? "positive" : "non-positive"

# case/when — uses === for comparison
case status
when "active"         then "User is active"
when "suspended"      then "User is suspended"
when Integer          then "Status is a number: #{status}"
when /^error_/        then "Error status: #{status}"
when nil              then "No status set"
else                       "Unknown status"
end

# case without expression — acts like if/elsif chain
case
when x < 0   then "negative"
when x == 0  then "zero"
when x > 0   then "positive"
end
```

**Q20. What are the loop constructs in Ruby?**

```ruby
# while — runs while condition is true
i = 0
while i < 5
  puts i
  i += 1
end

# until — runs until condition becomes true
i = 0
until i >= 5
  puts i
  i += 1
end

# loop — infinite loop, use break to exit
i = 0
loop do
  break if i >= 5
  puts i
  i += 1
end

# for..in — rarely used (each is preferred)
for i in 1..5
  puts i
end

# each — idiomatic Ruby
(1..5).each { |i| puts i }

# next (like continue) and break
(1..10).each do |i|
  next if i.even?   # skip even numbers
  break if i > 7    # stop at 7
  puts i
end
# => 1, 3, 5, 7
```

---

## Output Methods

**Q21. What is the difference between puts, print, p, and pp?**

```ruby
# puts — adds newline, calls to_s, prints "nil" as blank line
puts "hello"      # hello\n
puts [1, 2, 3]    # 1\n2\n3\n  (each element on own line)
puts nil          # \n           (blank line)

# print — no newline, calls to_s
print "hello"     # hello (cursor stays on same line)
print nil         # ""     (prints nothing)

# p — calls inspect, adds newline, returns the value (useful for debugging)
p "hello"         # "hello"\n  (with quotes — shows it's a string)
p nil             # nil\n       (explicitly shows nil)
p [1, 2, 3]       # [1, 2, 3]\n

# pp — pretty-print, like p but formatted for complex objects
pp({ name: "Alice", scores: [1,2,3], metadata: { active: true } })
# {name: "Alice", scores: [1, 2, 3], metadata: {active: true}}
```

---

## Object Introspection

**Q22. What are the key object introspection methods?**

```ruby
obj = "hello"

obj.class              # => String
obj.is_a?(String)      # => true
obj.is_a?(Object)      # => true  (String inherits from Object)
obj.kind_of?(String)   # => true  (alias for is_a?)
obj.instance_of?(String)  # => true  (exact class only, no inheritance)
obj.instance_of?(Object)  # => false

obj.respond_to?(:upcase)  # => true
obj.respond_to?(:fly)     # => false

obj.frozen?          # => false (unless frozen)
obj.freeze
obj.frozen?          # => true

obj.nil?             # => false
nil.nil?             # => true

# Methods listing
obj.methods.sort                  # all available methods
obj.public_methods(false)         # methods defined on String only
obj.respond_to?(:private_method, true)  # include private methods
```

**Q23. What is the difference between is_a?, kind_of?, and instance_of??**

```ruby
class Animal; end
class Dog < Animal; end

d = Dog.new

d.is_a?(Dog)      # => true   — checks class or any ancestor
d.is_a?(Animal)   # => true   — Dog inherits from Animal
d.is_a?(Object)   # => true   — everything inherits from Object

d.kind_of?(Dog)   # => true   — exact same as is_a?

d.instance_of?(Dog)     # => true   — exact class only
d.instance_of?(Animal)  # => false  — not the exact class
```

---

## Require / Load

**Q24. What is the difference between require, require_relative, and load?**

```ruby
# require — loads file once, searches $LOAD_PATH, no .rb extension needed
require 'json'
require 'my_library'

# require_relative — loads relative to current file's directory
require_relative 'models/user'       # loads ./models/user.rb
require_relative '../lib/helper'     # loads ../lib/helper.rb

# load — always re-executes the file (even if already loaded)
load 'config.rb'       # useful in scripts that need re-evaluation
load 'config.rb', true # wrap in anonymous module (prevents constant pollution)

# Checking if already required
$LOADED_FEATURES.include?("json.rb")  # check if loaded
```

---

## Special Variables and Keywords

**Q25. What are __method__, __LINE__, and __FILE__?**

```ruby
def greet
  puts __method__   # => greet    (current method name as symbol)
  puts __LINE__     # => 3        (current line number)
  puts __FILE__     # => app.rb   (current file name)
end

# Common use case: debugging
def dangerous_operation
  raise "Error in #{__method__} at #{__FILE__}:#{__LINE__}"
end

# __dir__ — directory of current file
path = File.join(__dir__, "config", "settings.yml")
```

**Q26. What is the difference between == and equal? and eql??**

```ruby
# == — value equality (most common, often overridden)
1 == 1.0        # => true   (Integer == Float by value)
"abc" == "abc"  # => true   (same value)

# eql? — strict value equality (same value AND same type)
1.eql?(1.0)     # => false  (different types)
1.eql?(1)       # => true
"abc".eql?("abc")  # => true

# equal? — identity equality (same object in memory)
a = "hello"
b = "hello"
c = a

a == b          # => true   (same value)
a.equal?(b)     # => false  (different objects)
a.equal?(c)     # => true   (same object)

:foo.equal?(:foo)  # => true  (symbols are always the same object)
```

**Q27. How does the safe navigation operator (&.) work?**

```ruby
user = nil
user.name           # => NoMethodError: undefined method 'name' for nil

# Safe navigation — returns nil if receiver is nil
user&.name          # => nil  (no error)
user&.name&.upcase  # => nil  (chains safely)

# Equivalent to:
user && user.name

# Common use case
account&.balance&.positive? || false
```

**Q28. What is the difference between frozen? and freeze?**

```ruby
str = "mutable"
str.frozen?   # => false
str << " text"  # works fine

str.freeze
str.frozen?   # => true
str << " more"  # => FrozenError: can't modify frozen String

# Frozen strings share memory
a = "hello".freeze
b = "hello".freeze
# May share same object in memory with frozen_string_literal magic

# freeze is shallow for collections
arr = [1, [2, 3]].freeze
arr << 4        # => FrozenError
arr[1] << 4     # => works! [1, [2, 3, 4]] — nested array not frozen
```

**Q29. What does the splat operator (*) do in Ruby?**

```ruby
# In method definitions — collect remaining args
def greet(first, *rest)
  puts "Hi #{first}, also: #{rest.join(', ')}"
end
greet("Alice", "Bob", "Carol")
# => Hi Alice, also: Bob, Carol

# Double splat (**) for keyword arguments
def configure(**options)
  options.each { |k, v| puts "#{k}: #{v}" }
end
configure(host: "localhost", port: 5432)

# Splat in assignment
first, *middle, last = [1, 2, 3, 4, 5]
# first => 1, middle => [2, 3, 4], last => 5

# Array expansion
a = [1, 2, 3]
b = [0, *a, 4]   # => [0, 1, 2, 3, 4]

# Convert to array
def wrap(*args)
  args           # args is always an Array
end
wrap(1)          # => [1]
wrap(1, 2, 3)    # => [1, 2, 3]
```

**Q30. What is the difference between Integer() and .to_i?**

```ruby
# to_i — lenient, never raises, returns 0 on failure
"42abc".to_i     # => 42    (parses up to non-digit)
"abc".to_i       # => 0     (returns 0 if no leading digits)
nil.to_i         # => 0

# Integer() — strict, raises ArgumentError on invalid input
Integer("42")    # => 42
Integer("42abc") # => ArgumentError: invalid value for Integer()
Integer("0xFF")  # => 255   (understands hex)
Integer("0b101") # => 5     (understands binary)
Integer(nil)     # => TypeError

# Float() similarly
Float("3.14")    # => 3.14
Float("abc")     # => ArgumentError

# Best practice for user input validation
def safe_parse_int(str)
  Integer(str)
rescue ArgumentError, TypeError
  nil
end
```

**Q31. What is string interpolation and what types can be interpolated?**

```ruby
name = "Alice"
age  = 30
arr  = [1, 2, 3]

"Hello, #{name}!"          # => "Hello, Alice!"
"In 10 years: #{age + 10}" # => "In 10 years: 40"
"Array: #{arr}"            # => "Array: [1, 2, 3]"  (calls to_s)
"#{nil} is nothing"        # => " is nothing"        (nil.to_s == "")
"Result: #{1 + 2 * 3}"     # => "Result: 7"

# Any expression works inside #{}
"#{name.upcase} says hi"   # => "ALICE says hi"
"#{[1,2,3].map { |n| n * 2 }.join(', ')}"  # => "2, 4, 6"

# Single quotes: NO interpolation
'Hello #{name}'   # => "Hello \#{name}"  (literal)
```

**Q32. How do Ruby's boolean operators (&& vs and, || vs or) differ?**

```ruby
# && and || — high precedence (use in conditionals)
result = true && false    # => false
result = false || true    # => true

# and, or — very low precedence (use for control flow)
# WARNING: different precedence causes subtle bugs

x = true && false   # x = (true && false) = false
x = true and false  # (x = true) and false — x is true!

# Safe use of 'and'/'or': flow control, not assignment
user = find_user or raise "User not found"
# Equivalent to: raise "User not found" unless (user = find_user)

# Recommended: always use && and || in conditions
```

**Q33. What are the numeric literal formats in Ruby?**

```ruby
# Underscores for readability
1_000_000       # => 1000000
1_234.567_89    # => 1234.56789

# Different bases
0b1010          # => 10    (binary)
0o17            # => 15    (octal)
0xFF            # => 255   (hexadecimal)

# Rational and Complex
3/2r            # => (3/2)  Rational number
2+3i            # => (2+3i) Complex number

# Scientific notation
1.5e3           # => 1500.0
2.5e-2          # => 0.025
```
