# Chapter 01 — Ruby Basics: Structured Answers

These are long-form answers designed for technical interviews where the interviewer expects a thorough explanation, not just a one-liner.

---

## Answer 1 — "Explain Ruby's object model and why everything is an object"

**The core principle:**
In Ruby, there are no primitive types. Every value is an instance of a class and has methods you can call on it. This is unlike Java or C, where integers and booleans are value types, not objects.

**Why this matters:**
```ruby
# In Java: int x = 5; is a primitive, you can't call methods on it
# In Ruby: every value responds to methods

42.class          # => Integer
42.respond_to?(:+)  # => true
42.to_s           # => "42"
42.to_f           # => 42.0
42.between?(10, 100)  # => true
42.zero?          # => false

true.class        # => TrueClass
false.class       # => FalseClass
nil.class         # => NilClass

# Even classes are objects!
String.class      # => Class
String.is_a?(Object)  # => true
```

**Practical consequence:**
Because everything is an object, you can store any value in variables, pass any value to methods, put any value in arrays/hashes, and call `respond_to?` to check capabilities — this enables polymorphism and duck typing without needing interfaces or type declarations.

**Interview signal:** "This design enables Ruby's expressive, method-chaining style and makes metaprogramming natural — you can open any class and add methods."

---

## Answer 2 — "What are the variable types in Ruby and their scopes?"

Ruby has five variable types, each with a naming convention that signals its scope:

```ruby
# 1. Local variables — lowercase/underscore start
#    Scope: method or block they're defined in
def calculate
  result = 42      # local to this method
  result
end

# 2. Instance variables — @ prefix
#    Scope: per object instance
class User
  def initialize(name)
    @name = name   # belongs to this specific User instance
  end

  def greet
    "Hello, #{@name}"
  end
end

User.new("Alice").greet  # => "Hello, Alice"

# 3. Class variables — @@ prefix
#    Scope: shared across the class AND all subclasses (dangerous!)
class Counter
  @@count = 0

  def initialize
    @@count += 1
  end

  def self.count
    @@count
  end
end

Counter.new
Counter.new
Counter.count  # => 2

# 4. Global variables — $ prefix
#    Scope: everywhere in the program (avoid in production code)
$app_started_at = Time.now
# Ruby uses built-in globals like $stdout, $0, $$, $LOAD_PATH

# 5. Constants — ALL_CAPS or CamelCase
MAX_SIZE = 1000
DatabaseURL = "postgres://localhost/db"
# Ruby warns but doesn't prevent reassignment
```

**Key differences table:**

| Variable | Prefix | Scope | Shared? |
|----------|--------|-------|---------|
| local | none | method/block | No |
| instance | @ | object | Per instance |
| class | @@ | class hierarchy | All subclasses |
| global | $ | everywhere | Yes |
| constant | uppercase | class/module | Accessible via :: |

---

## Answer 3 — "Explain symbols vs strings. When would you use each?"

**What makes symbols special:**
```ruby
# Symbols are immutable and interned — same symbol = same object
:status.frozen?              # => true
:status.object_id == :status.object_id  # => true (ALWAYS)

# Strings create new objects each time
"status".object_id == "status".object_id  # => false (new object each time)
```

**Memory implication:**
```ruby
# In a loop creating 1 million hash lookups:
# Using strings (bad): creates 1 million new String objects
1_000_000.times { config["host"] }

# Using symbols (good): :host is always the same object
1_000_000.times { config[:host] }
```

**When to use each:**
```ruby
# Symbols: internal identifiers, hash keys, method names, status values
config = { host: "localhost", port: 5432 }
state = :active
attr_accessor :name   # :name is a symbol

# Strings: content, user data, file content, anything that may change
name = "Alice"
error_message = "Connection failed: #{host}"
file_path = "/var/log/app.log"
```

**The conversion:**
```ruby
"active".to_sym   # => :active
:active.to_s      # => "active"
```

**Important caveat:** Symbol objects do stay in memory for the lifetime of the Ruby process (they're never garbage collected in older Ruby). In Ruby 2.2+, symbols created from strings via `to_sym` are garbage-collected if the string was dynamic.

---

## Answer 4 — "Walk me through Ruby's truthiness rules with examples"

**The rule:** Only `nil` and `false` are falsy. Everything else is truthy.

```ruby
# Falsy values (only these two):
if nil   then "truthy" else "falsy" end  # => "falsy"
if false then "truthy" else "falsy" end  # => "falsy"

# Everything else is truthy:
if 0     then "truthy" else "falsy" end  # => "truthy"  (surprise!)
if ""    then "truthy" else "falsy" end  # => "truthy"  (surprise!)
if []    then "truthy" else "falsy" end  # => "truthy"  (surprise!)
if {}    then "truthy" else "falsy" end  # => "truthy"  (surprise!)
if 0.0   then "truthy" else "falsy" end  # => "truthy"
```

**Why this matters in practice:**
```ruby
# Common bug: checking for empty string
user_input = ""
if user_input              # truthy! This runs even with empty input
  process(user_input)
end

# Correct:
if user_input && !user_input.empty?
  process(user_input)
end

# Or Rails way:
if user_input.present?  # handles nil, "", "  "
  process(user_input)
end

# Common use of nil-ness for optional values:
def find_user(id)
  # returns User object or nil
end

user = find_user(params[:id])
if user  # only nil is falsy, a User object is always truthy
  redirect_to user_path(user)
else
  render :not_found
end
```

---

## Answer 5 — "Explain the difference between puts, print, p, and pp"

```ruby
data = { name: "Alice", scores: [95, 87, 92] }

# puts — calls to_s, adds newline, special handling for arrays
puts data   # => {:name=>"Alice", :scores=>[95, 87, 92]}
            # uses Hash#to_s (same as inspect for Hash)

puts [1, 2, 3]
# 1
# 2
# 3   (each element on its own line)

puts nil    # (blank line — nil.to_s is "")

# print — calls to_s, no newline
print "hello "
print "world"
print "\n"   # need explicit newline

# p — calls inspect, adds newline, RETURNS the value
result = p data  # prints AND returns data
# {:name=>"Alice", :scores=>[95, 87, 92]}

# p distinguishes types:
p "hello"   # "hello"  (with quotes — string)
p 42        # 42       (no quotes — integer)
p nil       # nil      (explicit nil, not blank line)
p [1,nil,3] # [1, nil, 3]  (shows nil in array)

# pp — pretty print, handles complex objects with indentation
require 'pp'  # (automatic in Ruby 2.5+)
pp({ name: "Alice", address: { street: "123 Main St", city: "NYC" } })
# {:name=>"Alice",
#  :address=>{:street=>"123 Main St", :city=>"NYC"}}
```

**Interview signal:** "I use `p` for debugging because it returns the value, so I can insert it mid-chain without breaking the code: `arr.map { |x| x * 2 }.tap { |r| p r }.select(&:odd?)`"

---

## Answer 6 — "What is the safe navigation operator and when should you use it?"

```ruby
# Problem: chaining methods on potentially nil objects
user = find_user(id)
name = user.profile.display_name   # NoMethodError if user or profile is nil

# Old Ruby solution (verbose):
name = user && user.profile && user.profile.display_name

# Ruby 2.3+ safe navigation operator (&.)
name = user&.profile&.display_name   # returns nil at first nil in chain

# More examples:
order&.total&.round(2)       # nil if order or total is nil
config&.fetch(:timeout, 30)  # nil if config is nil
users&.first&.email&.downcase

# Safe navigation with methods that take blocks:
users&.map { |u| u.name }    # nil if users is nil, otherwise maps

# Combining with || for default values:
display_name = user&.name || "Anonymous"
timeout = config&.fetch(:timeout) || 30

# When NOT to use it:
# Overusing &. can hide nil propagation bugs
# Better to validate at the boundary (raise early, rescue at boundary)
```

---

## Answer 7 — "Explain freeze and when you'd use it"

```ruby
# freeze makes an object immutable
str = "hello"
str.freeze
str << " world"  # => FrozenError: can't modify frozen String

# Why freeze?
# 1. Safety: prevent accidental mutation of shared data
# 2. Performance: frozen strings can be deduplicated in memory
# 3. Thread safety: frozen objects are safe to share between threads

# Common patterns:
VALID_STATES = ["active", "inactive", "suspended"].freeze
MAX_RETRIES = 5  # constants don't need freeze for integers (already frozen)

# frozen_string_literal: true (magic comment)
# Freezes ALL string literals in the file automatically
# Encouraged for performance in Ruby 3.x

# freeze is shallow:
config = { host: "localhost" }.freeze
config[:host] = "other"  # => FrozenError (can't add/change keys)
config[:host] << "X"     # => Works! modifies the string VALUE

# For deep immutability, use a gem like ice_nine:
require 'ice_nine'
config = IceNine.deep_freeze({ host: "localhost", ports: [80, 443] })
config[:host] << "X"     # => FrozenError now
```

---

## Answer 8 — "How does Ruby handle string interpolation and what are heredocs?"

```ruby
# String interpolation: #{expression} evaluates any Ruby expression
name = "Alice"
puts "Hello, #{name}!"              # Hello, Alice!
puts "2 + 2 = #{2 + 2}"            # 2 + 2 = 4
puts "#{name.upcase} is #{30} now" # ALICE is 30 now

# Interpolation calls to_s on the result
arr = [1, 2, 3]
puts "Array: #{arr}"  # Array: [1, 2, 3]

# Single quotes: no interpolation
puts 'Hello, #{name}!'  # Hello, #{name}!  (literal)

# Heredoc: multi-line strings
sql = <<~SQL
  SELECT *
  FROM users
  WHERE active = true
  ORDER BY created_at DESC
SQL
puts sql.class   # => String

# <<~ strips leading whitespace (squiggly heredoc, Ruby 2.3+)
# <<- just allows closing delimiter to be indented

# Heredoc with interpolation (double-quote heredoc, default):
user_name = "Alice"
message = <<~MSG
  Dear #{user_name},
  Your account was created on #{Date.today}.
MSG

# Heredoc without interpolation (single-quote heredoc):
template = <<~'SHELL'
  echo "Hello, ${NAME}"
  export PATH=$PATH:/usr/local/bin
SHELL
# ${NAME} is NOT interpolated — useful for shell scripts
```

---

## Answer 9 — "Explain the splat operator and double splat operator"

```ruby
# Single splat (*) — collects remaining positional arguments
def log(level, *messages)
  messages.each { |msg| puts "[#{level}] #{msg}" }
end

log(:info, "Server started", "Listening on port 3000")
# [info] Server started
# [info] Listening on port 3000

# Splat in assignment (array decomposition)
first, *rest = [1, 2, 3, 4, 5]
first  # => 1
rest   # => [2, 3, 4, 5]

head, *body, tail = [1, 2, 3, 4, 5]
head  # => 1
body  # => [2, 3, 4]
tail  # => 5

# Array expansion/splatting
a = [2, 3, 4]
b = [1, *a, 5]  # => [1, 2, 3, 4, 5]

# Double splat (**) — keyword arguments
def connect(**options)
  host = options.fetch(:host, "localhost")
  port = options.fetch(:port, 5432)
  "#{host}:#{port}"
end
connect(host: "db.prod.com", port: 5433)  # => "db.prod.com:5433"
connect                                    # => "localhost:5432"

# Merging hashes with **
defaults = { timeout: 30, retries: 3 }
overrides = { timeout: 60 }
config = { **defaults, **overrides }
# => { timeout: 60, retries: 3 }  (overrides win)
```

---

## Answer 10 — "What are the common pitfalls with mutable default arguments?"

```ruby
# This is a Ruby version of the Python mutable default argument problem

# WRONG — don't use mutable objects as default arguments in methods:
def add_item(list = [])
  list << "item"
  list
end

# In Ruby, each call creates a fresh default array (unlike Python!)
add_item  # => ["item"]
add_item  # => ["item"]  (fresh default each time — Ruby is safe here)

# However, class-level mutation IS a problem:
class Cache
  DEFAULT_OPTIONS = { ttl: 300, max_size: 100 }

  def configure(opts = DEFAULT_OPTIONS)
    opts[:ttl] = 60   # MUTATES the class-level constant!
    opts
  end
end

# Fix: dup the default
def configure(opts = DEFAULT_OPTIONS.dup)
  opts[:ttl] = 60   # only mutates the copy
  opts
end

# Or freeze the constant:
DEFAULT_OPTIONS = { ttl: 300, max_size: 100 }.freeze
# Now any mutation attempt raises FrozenError
```
