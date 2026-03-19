# Chapter 06: Metaprogramming — Analogy Explanations

## 1. `method_missing` — The Receptionist Who Handles Unknown Requests

Imagine a large office building with a front-desk receptionist. When someone comes in asking for a specific person, the receptionist either:
- Connects them to that person (normal method dispatch), OR
- If no one by that name works there, the receptionist intercepts the request and either routes it creatively or says "No one by that name works here" (raises `NoMethodError`).

`method_missing` is that receptionist. When Ruby can't find a method anywhere in the lookup chain, it calls `method_missing` as a last chance to handle the call before raising an error.

And `respond_to_missing?` is the building directory. If someone calls ahead and asks "Does someone named `find_by_email` work there?", the directory should accurately say YES — otherwise people won't even bother showing up (they'll think the capability doesn't exist).

```ruby
# The receptionist (method_missing) handles unknown visitors
def method_missing(name, *args)
  # Route the request somehow, or call super to say "nobody home"
end

# The directory (respond_to_missing?) must be accurate
def respond_to_missing?(name, include_private = false)
  # Return true if we'd handle it in method_missing
end
```

---

## 2. `define_method` — A Factory That Stamps Out Customized Functions

Think of a cookie-cutter factory. You have one master template (the block you pass to `define_method`), but you stamp out many different cookies, each with a slightly different filling (the closed-over variable).

A regular `def` is like carving a cookie by hand — each one is independent and starts with a blank slate. `define_method` is the stamping machine — it creates cookies that all share the same shape but carry the filling that was in the hopper at the moment they were stamped.

```ruby
fillings = ['chocolate', 'vanilla', 'strawberry']
fillings.each do |filling|  # the "hopper" is loaded with each filling
  define_method(:"#{filling}_cookie") do
    "A cookie with #{filling} filling"  # the filling is baked in at stamp time
  end
end
```

This is why all `define_method` methods created in a loop capture the correct loop variable — each iteration loads a fresh filling into the hopper before stamping.

---

## 3. Open Classes (Monkey Patching) — Editing a Published Book

Imagine a book is already printed and distributed to thousands of readers. Monkey patching is like sneaking into every reader's copy and editing a sentence on page 47.

Everyone who reads that book from now on sees your edit. The original author never intended it. Other editors might also be editing the same sentence with contradictory changes. And readers who expected the original text are now confused.

```ruby
class String
  def upcase  # You just "edited page 47" of Ruby's String book
    "OVERRIDDEN"
  end
end

"hello".upcase  # => "OVERRIDDEN" — every string in every gem sees this
```

Refinements are like publishing an annotated edition — only readers who specifically pick up *your* annotated version see the changes. Everyone else reads the original.

---

## 4. `class_eval` vs `instance_eval` — Company Policy vs Personal Post-it Notes

`class_eval` is like announcing a new company-wide policy. Every employee (instance) in the company (class) follows this policy from now on.

`instance_eval` is like putting a post-it note on one specific employee's desk. Only that person follows the note; nobody else is affected.

```ruby
# class_eval: company-wide policy (instance method)
String.class_eval do
  def shout = upcase + "!!!"
end
"hello".shout   # Every string in the company follows the policy

# instance_eval: personal post-it note (singleton method)
one_string = "hello"
one_string.instance_eval do
  def shout = "just me shouting"
end
one_string.shout   # Only this string has the post-it
"world".shout      # NoMethodError — that employee doesn't have the note
```

---

## 5. `send` vs `public_send` — Master Key vs Employee Key Card

`send` is the master key that opens every door in the building — public offices, private server rooms, the CEO's office. It respects no access restrictions.

`public_send` is the standard employee key card that only opens doors marked "Public". Trying to open a door marked "Private" results in "Access Denied."

`__send__` is the unbreakable master key — even if someone replaces the normal master key (`send`) with a fake, this one always works.

```ruby
# send = master key (opens private doors)
object.send(:private_admin_method)  # works, bypasses access control

# public_send = employee key card (respects access control)
object.public_send(:private_admin_method)  # NoMethodError: access denied

# __send__ = unbreakable master key (can't be overridden)
object.__send__(:method_name)  # always delegates to Ruby's dispatch, cannot be redefined
```

---

## 6. Refinements — Noise-Canceling Headphones

When you put on noise-canceling headphones (`using SomeRefinement`), you hear a different version of the world — the music you choose, not the ambient noise. When you take them off (leave the `using` scope), the world sounds normal again.

Other people in the room don't hear your music. Your headphones are personal and scoped to you.

```ruby
module BetterMath
  refine Integer do
    def factorial
      (1..self).reduce(1, :*)
    end
  end
end

module Calculator
  using BetterMath   # put on headphones

  def self.compute(n)
    n.factorial   # I can hear factorial
  end
end

5.factorial          # NoMethodError — you don't have the headphones on
Calculator.compute(5)  # => 120 — Calculator "hears" factorial
```

---

## 7. `included` Hook with `extend(ClassMethods)` — A Package Deal

Think of this pattern like buying a piece of furniture that comes with both assembly instructions (class methods) AND the actual furniture parts (instance methods).

When you `include` the module, the `included` hook fires and automatically hands you both the instructions (`extend(ClassMethods)`) and the parts (`instance methods in the module`). You don't have to separately remember to call `extend` — the package deal handles it.

```ruby
module Searchable
  def self.included(base)           # When the package arrives...
    base.extend(ClassMethods)       # ...hand over the instructions (class methods)
  end                               # ...the module body itself is the furniture (instance methods)

  # Instance methods = furniture parts
  def matches?(query)
    to_s.include?(query)
  end

  # Class methods = assembly instructions
  module ClassMethods
    def search(query)
      all.select { |item| item.matches?(query) }
    end
  end
end

class Product
  include Searchable  # package arrives, both parts delivered automatically
end

Product.search("ruby")       # class method (instructions)
Product.new.matches?("ruby") # instance method (furniture)
```

---

## 8. `tap` vs `then` — Tasting vs Transforming

A chef is making a dish through multiple preparation steps.

`tap` is the chef tasting the dish at a step — they taste it, maybe make a note, but put it back unchanged and continue. The dish (object) keeps moving through the pipeline as-is.

`then` is when the chef hands the dish to a different station that transforms it — the sauce station returns a sauced dish, not the original unsauced one.

```ruby
# tap = taste and continue (dish unchanged)
sauce = "tomato"
  .tap { |s| puts "Checking ingredients: #{s}" }  # tasted
  .upcase  # continues as "tomato", then upcased to "TOMATO"

# then = transform at this station
result = 42
  .then { |n| n * 2 }   # station returns 84
  .then { |n| "Result: #{n}" }  # station returns "Result: 84"
# => "Result: 84"
```

---

## 9. `ObjectSpace` — The City Population Census

`ObjectSpace` is like the city's population census bureau. At any moment, it can tell you:
- How many people of each "type" (class) are alive
- Where each person was born (allocation source file and line)
- How much space each person takes up (memory size)

Most of the time you don't need the census — you just live your life. But when something goes wrong (memory leak = population explosion of one type), the census bureau helps you find where all these extra people came from.

```ruby
# Census bureau
ObjectSpace.each_object(String).count  # How many strings are alive?

require 'objspace'
ObjectSpace.trace_object_allocations_start
my_obj = SomeClass.new                  # Birth registered
ObjectSpace.allocation_sourcefile(my_obj)  # Where was this person born?
ObjectSpace.allocation_sourceline(my_obj)  # On which line?
```

---

## 10. Method Lookup Chain — Chain of Command in the Military

When a soldier (object) receives an order (method call), they check: "Can I handle this myself?" If yes, execute. If no, pass it up the chain of command.

The chain goes:
1. The soldier themselves (singleton class)
2. The soldier's unit's prepended protocols (prepended modules)
3. The soldier's unit (class)
4. The unit's included field manuals (included modules)
5. The parent command (superclass)
6. ... up to Supreme HQ (BasicObject)
7. If nobody can handle it → `method_missing` is called (filing a "cannot comply" report)

```ruby
# Chain of command for obj.greet:
# obj's singleton class → prepended modules → obj.class →
# included modules → superclass → ... → BasicObject
# → method_missing if nobody can handle it
# → NoMethodError if method_missing also can't handle it
```

The key insight: prepended modules are inserted *before* the unit in the chain, giving them first right of refusal. Included modules are inserted *after* the unit, acting as fallback field manuals.
