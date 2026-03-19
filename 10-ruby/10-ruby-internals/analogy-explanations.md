# Chapter 10: Ruby Internals — Analogy Explanations

## 1. YARV — A Country's Legal Code Translated into Court Instructions

Raw Ruby source code is like a legal statute written in plain English — human-readable but too ambiguous for a judge to directly enforce. YARV is the court system's translation of that statute into a precise, numbered list of legal procedures that a courtroom (the VM) can execute mechanically, step by step.

The parser is the legal scholar who interprets the statute into an AST. The compiler is the drafting committee that turns the AST into the procedural rulebook (bytecode). The YARV VM is the court itself, blindly executing each numbered instruction without re-reading the original statute.

When YJIT kicks in, it's like a court that has processed the same case type hundreds of times and has the procedure completely memorized — no need to consult the rulebook at all. The judge just announces the verdict from muscle memory.

---

## 2. Tri-Color GC — A Hospital's Patient Classification System

Imagine a hospital emergency room that needs to assess all patients simultaneously during a mass casualty event.

**White patients** = unevaluated (potentially discharged, potentially critical). **Grey patients** = a doctor has seen them but hasn't seen everyone they came in contact with (their "references" — family members, coworkers who might also be affected). **Black patients** = fully evaluated, and everyone connected to them has been assessed as well.

The triage team (GC) works through grey patients one at a time: evaluates them fully (marks black), and sends their contacts to the waiting room (marks them grey for evaluation).

The **write barrier** is the rule that if a fully-cleared (black) patient suddenly reports a new contact they forgot to mention, they get moved back to the "needs re-evaluation" (grey) queue to ensure their new contact gets assessed too.

After all grey patients are processed: white patients are discharged (memory freed), black patients are admitted (kept).

---

## 3. Object Shapes — Postal Routes vs Random House Visits

Before Object Shapes, finding an instance variable in a Ruby object was like a mail carrier who has to re-read the address label on every package to figure out which house to go to, even if they've been to that house thousands of times before.

Object Shapes are like established postal routes. If an entire neighborhood of houses (objects) always has the mailbox at the front door, living room to the right, and kitchen at the back, the mail carrier (YJIT) learns the route once and can deliver to any house on that route without reading the label. `@name` is always the first mailbox slot, `@email` is always the second — no lookup needed.

When houses are built with rooms in random order (instance variables set in different orders per object), it's a non-standard neighborhood — the mail carrier can't use the memorized route and has to read labels each time.

---

## 4. YJIT — A Simultaneous Interpreter Who Learns Your Speaker

A simultaneous interpreter at an international conference normally translates everything in real-time, word by word. Each sentence is work.

But after a few days with the same speaker, the interpreter has learned that speaker's patterns, idioms, and vocabulary. For the speaker's most common phrases, the interpreter no longer needs to process each word — they can translate the entire phrase in one instant motion.

YJIT is this interpreter. The first time through a method (cold), it translates every Ruby bytecode instruction. After seeing the method many times with the same types, it learns the pattern and compiles a fast path — the whole method executes in one "motion" of native machine code.

When the speaker says something unexpected (type change, method redefinition), the interpreter falls back to careful word-by-word translation (deoptimization to interpreter) until the new pattern is learned.

---

## 5. Ractors — Separate Work Rooms in a Factory

Traditional Ruby threads are like workers in an open-plan office. They're all productive, but there's only ONE microphone (the GIL), so only one worker can speak (execute Ruby code) at a time. Multiple workers doing I/O-bound tasks (waiting on hold with clients) can all be "active" at once because they're not talking.

Ractors are like separate, fully equipped work rooms. Each room has its own microphone, its own tools, its own communication channel. Two rooms can have completely independent conversations simultaneously — true parallel execution.

The trade-off: to communicate between rooms, you must pass documents through the mail slot (inter-Ractor message passing). Documents must be sealed (frozen) before they can be passed through — you can't pass a live plant (mutable object) through the mail slot, only packaged goods (frozen objects or primitive values).

---

## 6. Fibers — Taking Numbers at the DMV

The DMV has one clerk (single thread). Many people are waiting and their tasks have different durations. Instead of making everyone wait while one person's complex transaction completes, the DMV uses a "take a number" system.

When your task needs to pause (waiting for a form to be filled out, analogous to waiting for I/O), you step aside and the clerk serves the next number. When your form is ready, you're called back to the counter.

Fibers are each customer with a number. The Fiber scheduler is the clerk management system. When a Fiber yields (steps aside), the scheduler picks the next ready Fiber. All customers get served, no one blocks the entire queue, and it happens on a single thread (one DMV clerk).

---

## 7. Pattern Matching — Document Sorting with Templates

Traditional Ruby conditions (`if`, `case/when`) are like a human reading every document and making individual decisions. Pattern matching is like having a set of document templates — if a document matches this template's exact structure, process it this way.

`case/in` is the template-based sorter. You describe the shape: "if this is a Hash with a `:status` key that's 200 and a `:body` key that's a String, extract the body and handle it as success." The sorter checks documents against templates top-to-bottom and routes them accordingly.

`deconstruct` and `deconstruct_keys` are the document standardization process — they tell the sorter how to "unfold" your custom document type into an array or hash format that the templates understand.

---

## 8. `$LOAD_PATH` — A Library's Catalog and Stacks

`$LOAD_PATH` is like a library's catalog of physical sections to search. When you call `require 'json'`, the librarian searches each section in order: "Is there a book called 'json' in this section?" The first section that has it wins.

Adding to `$LOAD_PATH` is like adding a new shelf to search. Bundler does this automatically — it prepends the right gem version's shelf so that shelf is searched first, even if the same gem exists in another section (different version).

`$LOADED_FEATURES` is the checkout record. Once "json" is checked out, the librarian notes it and won't bother searching the stacks again on subsequent `require` calls — they just say "already checked out, you have it."

`load` is like asking for a fresh copy every time — the librarian ignores the checkout record and brings you a new copy regardless.

---

## 9. GIL — A Server Farm's Single Admin Terminal

The Global Interpreter Lock (GIL) is like a server farm that has many servers (CPU cores) but only ONE admin terminal (the GIL). Any administrator (Ruby thread) who needs to manage the servers (execute Ruby code) must take turns at the terminal.

While an admin is waiting for a long-running deployment script to finish (I/O: network, disk), they step away from the terminal so another admin can use it. This is why Ruby threads work well for I/O-bound work (multiple admins manage different tasks, each releasing the terminal while waiting).

For CPU-bound work, all admins need the terminal constantly — only one can work at a time, so four threads on a 4-core machine is no faster than one thread.

Ractors are like getting FOUR separate admin terminals — each room has its own terminal, each Ractor can execute Ruby code truly simultaneously.

---

## 10. `Data` Class — A Safety Deposit Box

A `Struct` is like a regular filing cabinet drawer — you can put documents in, rearrange them, remove them. The drawer is mutable.

`Data.define` (Ruby 3.2+) is like a safety deposit box at a bank. You deposit the contents at creation (constructor), and after that, the box is permanently sealed. Nobody can add, remove, or modify the contents. The only way to get a "different" box is to get a new one — the `.with(key: new_value)` method is like asking the bank to create a new sealed box with modified contents while leaving the original box untouched.

This immutability is by design: for value objects, coordinates, money, request IDs, and other things that should never change once created, a sealed box is exactly the right abstraction. It makes bugs impossible (no accidental mutation) and makes the object safe to share across threads without locks.
