# Chapter 03 — Blocks, Procs & Lambdas: Analogy Explanations

---

## Analogy 1 — Blocks: One-Time Instructions

**Concept:** A block is anonymous code passed to a method, used once, then discarded.

**Analogy:** A block is like whispered instructions you give to a courier as they're about to leave. You say: "When you get to the warehouse, do this specific thing with each package." You don't write those instructions down — you just say them. The courier follows them and you can't reuse those exact instructions anywhere else. The block exists only for that one delivery run.

---

## Analogy 2 — Proc: Reusable Saved Instructions

**Concept:** A Proc is a block saved as an object you can store, pass around, and reuse.

**Analogy:** A Proc is like a printed checklist you laminate and hand out to any courier. Instead of whispering the same instructions each time, you print "Step 1: check the label. Step 2: scan the barcode. Step 3: leave a note." Anyone can pick up a copy and follow the checklist. You can hand the same checklist to multiple people or store it in a drawer for later. Unlike a whispered instruction (block), the checklist (Proc) is a physical object you can pass around.

---

## Analogy 3 — Lambda: The Formal Contract Version

**Concept:** A Lambda is a strict Proc — it checks the number of arguments and returns locally.

**Analogy:** A Lambda is like a signed legal contract. A Proc is like a verbal agreement — flexible, forgiving, you can add extra clauses and no one complains, and if someone says "I'm done" (return) the whole deal is off. A Lambda is the notarized version: the number of items in the contract must match exactly (strict arity), and signing one clause (return inside a lambda) doesn't void the entire transaction — it just completes that clause and life continues.

---

## Analogy 4 — Closure: A Backpack of Context

**Concept:** A closure carries its surrounding variables when it's passed around.

**Analogy:** Imagine sending an astronaut to the moon. The astronaut (the block/lambda) carries a life-support backpack. That backpack contains the oxygen (variables) from Earth that they need to survive in space. Even when they're on the moon — far from Earth (outside the original method scope) — they still have the oxygen from home. The backpack travels with them and connects them to their origin. When the astronaut modifies what's in the backpack (mutates a captured variable), they're changing the actual oxygen supply back home too.

---

## Analogy 5 — yield: Handing Off to the Guest Speaker

**Concept:** yield transfers execution to the block, then resumes.

**Analogy:** Imagine hosting a conference. You're the event organizer (the method). You set up the chairs, test the microphone, and run the show. But you hired a guest speaker (the block). When you step up and say "I yield the floor to our speaker" (call `yield`), the speaker delivers their content. When they're done, control returns to you and you wrap up the event. If no speaker was booked (`block_given?` returns false) and you try to yield, chaos erupts (LocalJumpError).

---

## Analogy 6 — Proc#return vs Lambda#return

**Concept:** return inside a Proc exits the enclosing method; return inside a Lambda exits only the lambda.

**Analogy:** A Proc is like a fire alarm in a building. When someone triggers it (calls return inside the Proc), the entire building evacuates (the enclosing method exits). Everyone inside — including the person who triggered it and everyone else in the building — has to leave immediately.

A Lambda is like a fire drill in a single meeting room. The participants in that room (the lambda code) go through the motions and "exit the room" (return from the lambda), but the rest of the building (the enclosing method) continues normal operations. No one else is affected.

---

## Analogy 7 — &:symbol: The Shape-Cutter

**Concept:** &:symbol converts a symbol to a block via Symbol#to_proc.

**Analogy:** `&:upcase` is like a cookie cutter in the shape of "UPCASE." You press the cutter onto each piece of dough (each element in the array) and it stamps out the uppercase version. You don't write `{ |dough| dough.upcase }` each time — you just hold up the :upcase cutter and say "use this shape." Symbol#to_proc is the mechanism that turns the cutter's label into actual cutting behavior.

---

## Analogy 8 — Lazy Enumerator: On-Demand Manufacturing

**Concept:** Lazy evaluation only processes what's needed, rather than processing everything upfront.

**Analogy:** Eager evaluation is like a bakery that bakes 10,000 loaves at 5am, then you pick up the 3 you need. Most loaves go stale. Lazy evaluation is like a bakery that bakes loaves one at a time as customers order them. If only 3 people order, only 3 loaves are baked. For infinite sequences (what if you needed bread forever?), lazy evaluation is the only approach — no warehouse could store infinite bread.

---

## Analogy 9 — Currying: Build-to-Order Configurator

**Concept:** Currying lets you pre-fill some arguments of a function to create a specialized version.

**Analogy:** Imagine a sandwich shop's ordering tablet. The full order function is `make_sandwich(bread, protein, topping)`. Currying lets you "lock in" some choices early. If you're hosting a vegetarian party, you pre-configure: "This terminal only orders veggie protein." Now guests just pick their bread and toppings — the protein is already locked in. You've partially applied the function. Later you can create another terminal for the meat-eaters, pre-configured with chicken. One function, multiple specialized versions.

---

## Analogy 10 — Method Objects: First-Class Work Orders

**Concept:** method(:name) wraps a method in an object you can store and pass around.

**Analogy:** Normally, methods are like skills that live inside a person — Alice knows how to sort files, but you can't take her "sorting skill" and give it to someone else to use. `method(:sort_files)` is like writing Alice's sorting procedure on a work order card. Now that card can be passed to anyone, stored in a box, mailed to another department. When someone has the card, they can call Alice's exact procedure without Alice being present. The Method object is the work order that captures the procedure and makes it portable.
