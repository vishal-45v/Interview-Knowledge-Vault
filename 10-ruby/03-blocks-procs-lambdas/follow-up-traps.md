# Chapter 03 — Blocks, Procs & Lambdas: Follow-Up Traps

---

## Trap 1 — return inside a Proc returns from the enclosing METHOD

```ruby
def dangerous_proc
  p = Proc.new { return 42 }
  p.call              # this EXITS the method entirely!
  puts "never runs"   # never reached
end

def safe_proc
  p = Proc.new { next 42 }  # use `next` to return from proc only
  result = p.call
  puts "continues: #{result}"
end

def safe_lambda
  l = lambda { return 42 }
  result = l.call     # lambda return exits lambda only
  puts "continues: #{result}"
end

dangerous_proc  # => 42 (method returns)
safe_proc       # => "continues: 42"
safe_lambda     # => "continues: 42"

# The dangerous trap: proc stored and called outside the original method context
def make_proc
  Proc.new { return "oops" }
end

p = make_proc
p.call  # => LocalJumpError: unexpected return
# The method make_proc has already returned — the Proc's return has nowhere to go!
```

---

## Trap 2 — Lambda checks argument count; Proc silently ignores extras

```ruby
strict = lambda { |a, b| [a, b] }
loose  = proc { |a, b| [a, b] }

strict.call(1, 2)    # => [1, 2]
strict.call(1)       # => ArgumentError (too few)
strict.call(1, 2, 3) # => ArgumentError (too many)

loose.call(1, 2)     # => [1, 2]
loose.call(1)        # => [1, nil]  (b is nil, no error)
loose.call(1, 2, 3)  # => [1, 2]   (3 is silently dropped)
loose.call           # => [nil, nil]  (no error at all)

# This can cause SILENT bugs:
process = proc { |user, role| grant_access(user, role) }
process.call(current_user)     # role is nil! no error raised
# => grant_access(user, nil)   might silently fail or grant wrong access
```

---

## Trap 3 — { } binds tighter than do...end

```ruby
def foo(x)
  "foo(#{x.inspect})"
end

# { } binds to map — foo receives the result of map
puts foo [1,2,3].map { |n| n * 2 }
# => foo([2, 4, 6])

# do...end binds to foo — map receives no block, returns Enumerator
puts foo [1,2,3].map do |n| n * 2 end
# => foo(#<Enumerator:...>)

# Real-world gotcha: RSpec style
# This passes the block to describe, not to expect:
expect(something).to change { counter }.by(1)

# This might parse differently depending on nesting:
result = some_method arg1, arg2 do |x|
  x * 2
end
# The block goes to some_method, not arg2 (if arg2 is a method)

# Safe practice: use { } for inline, do...end for multi-line
# And use parentheses when in doubt:
foo([1,2,3].map do |n| n * 2 end)  # explicit parens force { } bind
```

---

## Trap 4 — &:symbol works because Symbol#to_proc

```ruby
# :upcase.to_proc is APPROXIMATELY:
# proc { |receiver, *args| receiver.send(:upcase, *args) }

# So this works:
["hello", "world"].map(&:upcase)  # => ["HELLO", "WORLD"]

# But this does NOT work if the method needs arguments:
[1.234, 2.567].map(&:round)       # => [1, 3]  (round() with no args = 0 decimal places)
[1.234, 2.567].map(&:round(2))    # => SyntaxError!

# For methods with arguments, use a lambda:
[1.234, 2.567].map { |n| n.round(2) }    # => [1.23, 2.57]

# Or curry:
round_2 = :round.to_proc.curry.call(2)   # No, this won't work (wrong receiver order)
# Use a lambda:
round_2 = ->(n) { n.round(2) }
[1.234, 2.567].map(&round_2)             # => [1.23, 2.57]

# Trap: &:method_name on Hash.each yields [key, value] pairs
{a: 1, b: 2}.map(&:inspect)  # => ["[:a, 1]", "[:b, 2]"]
# (each element is [key, value] array, inspect is called on that array)
```

---

## Trap 5 — Proc.new inside a method without a block raises ArgumentError

```ruby
def greedy_proc_new
  Proc.new  # captures caller's block if one exists
end

p1 = greedy_proc_new { |x| x * 2 }
p1.call(5)   # => 10

p2 = greedy_proc_new  # no block
# => ArgumentError: tried to create Proc object without a block

# Contrast with proc {} which also raises if no block:
def greedy_proc
  proc  # shorthand — same issue
end

# lambda/-> does NOT have this issue — always creates a new lambda:
def make_lambda
  lambda { |x| x * 2 }  # always works, doesn't consume caller's block
end
```

---

## Trap 6 — Closures capture binding (reference), not value

```ruby
# Safe: each block creates new binding for block param
procs = (1..5).map { |i| -> { i } }
procs.map(&:call)   # => [1, 2, 3, 4, 5]  (each i is separate)

# Dangerous: while loop reuses same variable
procs = []
i = 1
while i <= 5
  procs << -> { i }  # all reference the SAME i
  i += 1
end
procs.map(&:call)   # => [6, 6, 6, 6, 6]  (i ended at 6)

# Fix: capture current value explicitly
procs = []
i = 1
while i <= 5
  current = i        # create a new binding each iteration
  procs << -> { current }
  i += 1
end
procs.map(&:call)   # => [1, 2, 3, 4, 5]

# Real-world version of this trap (setTimeout equivalent):
callbacks = []
["click", "hover", "focus"].each_with_index do |event, idx|
  callbacks << -> { "Handler #{idx} for #{event}" }
end
callbacks.map(&:call)
# => ["Handler 0 for click", "Handler 1 for hover", "Handler 2 for focus"]
# SAFE because each is used (not while)
```

---

## Trap 7 — yield with a splat doesn't work as expected

```ruby
def test
  yield 1, 2, 3
end

test { |*args| p args }        # => [1, 2, 3]  — splat collects all
test { |a, b| p [a, b] }      # => [1, 2]     — extra arg ignored (Proc rules)
test { |a, b, c, d| p d }     # => nil         — missing arg is nil

# Passing an array to yield:
def test2
  yield [1, 2, 3]
end

test2 { |a| p a }             # => [1, 2, 3]  — single array arg
test2 { |a, b, c| p [a,b,c] } # => [1, 2, 3]  — array is splatted automatically!
# When a single array is yielded to a multi-arg block, Ruby splats it
```

---

## Trap 8 — Lambda#lambda? returns true; Proc#lambda? returns false

```ruby
l = lambda { |x| x }
p = proc   { |x| x }
n = Proc.new { |x| x }

l.lambda?   # => true
p.lambda?   # => false
n.lambda?   # => false

# Methods converted via method() are lambda-like:
m = method(:puts)
m.to_proc.lambda?  # => true  (method-derived Proc is lambda-like)

# How to check at runtime:
def safe_call(callable, *args)
  if callable.respond_to?(:lambda?) && callable.lambda?
    callable.call(*args)     # strict arity check
  else
    callable.call(*args)     # will silently ignore extra args
  end
end
```

---

## Trap 9 — next vs return vs break inside blocks

```ruby
# next — skips to next iteration (like continue in other languages)
[1,2,3,4,5].each do |n|
  next if n.even?
  puts n
end
# => 1, 3, 5

# break — exits the loop, returns value to the calling expression
result = [1,2,3,4,5].each do |n|
  break n * 10 if n == 3
end
puts result  # => 30  (break returns the value)

# return — exits the ENCLOSING METHOD (dangerous in blocks!)
def find_first_even(arr)
  arr.each do |n|
    return n if n.even?  # returns from find_first_even
  end
  nil
end
find_first_even([1, 3, 4, 5])  # => 4

# In procs: next returns from the proc; return returns from enclosing method
my_proc = proc { |n| next n * 2 if n.odd?; n }
[1, 2, 3].map(&my_proc)   # => [2, 2, 6]
```

---

## Trap 10 — Reusing a block with & after storing it

```ruby
def make_adder(n)
  ->(x) { x + n }
end

add5 = make_adder(5)

# Works:
[1, 2, 3].map(&add5)     # => [6, 7, 8]

# Also works — & calls to_proc on lambdas:
add5.to_proc.call(10)    # => 15

# Trap: you can't use yield to re-yield a stored block
def outer(&blk)
  inner(blk)  # passes blk (a Proc) as a regular argument — NO
end

def inner(blk)
  yield  # => LocalJumpError — outer's block is not available here
end

# Correct way to pass blocks along:
def outer(&blk)
  inner(&blk)    # & converts the Proc back to a block
end

def inner(&blk)
  blk.call(42)   # or: yield 42
end

outer { |n| n * 2 }  # => 84
```

---

## Trap 11 — Proc composition order with >> and <<

```ruby
double    = ->(n) { n * 2 }
add_one   = ->(n) { n + 1 }

# >> is left-to-right (double THEN add_one)
f = double >> add_one
f.call(5)   # => 11  (5*2=10, then 10+1=11)

# << is right-to-left (add_one THEN double) — mathematical composition
g = double << add_one
g.call(5)   # => 12  (5+1=6, then 6*2=12)

# Think of << as f(g(x)) notation: double(add_one(x))
# Think of >> as left-to-right pipeline: x → add_one → double
```

---

## Trap 12 — tap is not the same as yielding a value

```ruby
# tap: yields self, returns self (for side effects in a chain)
result = [1, 2, 3]
  .tap { |arr| puts "before: #{arr.inspect}" }
  .map { |n| n * 2 }
  .tap { |arr| puts "after: #{arr.inspect}" }
  .select(&:odd?)

# then/yield_self: yields self, returns the BLOCK'S return value
result = 5
  .then { |n| n * 2 }   # => 10 (block return value, not 5)
  .then { |n| n.to_s }  # => "10"

# Trap: confusing tap with then
[1, 2, 3].tap  { |a| a.map { |n| n * 2 } }
# => [1, 2, 3]  (tap returns SELF, not the block result!)

[1, 2, 3].then { |a| a.map { |n| n * 2 } }
# => [2, 4, 6]  (then returns block result)
```
