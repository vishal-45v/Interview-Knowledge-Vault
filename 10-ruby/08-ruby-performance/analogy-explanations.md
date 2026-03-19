# Chapter 08: Ruby Performance — Analogy Explanations

## 1. Garbage Collector — City's Waste Management Service

The Ruby heap is like a city. Every Ruby object is a building constructed in the city. The garbage collector (GC) is the waste management service.

The city has two neighborhoods: the **new construction zone** (young generation / Eden heap) where fresh buildings go up every day, and the **established district** (old generation) where long-standing buildings have survived many inspections.

A **minor GC** is like the weekly trash pickup that only services the new construction zone — fast, handles the most common case (most new buildings are temporary). A **major GC** is like a full city-wide audit — much slower, but necessary periodically to clean up old structures that finally need to come down.

The challenge: while waste management is happening, the city is frozen (stop-the-world pause). Incremental GC is like deploying many small waste trucks throughout the day instead of one massive convoy that blocks all traffic at once.

```ruby
GC.stat[:minor_gc_count]  # weekly pickups done
GC.stat[:major_gc_count]  # full city audits done
GC.stat[:heap_live_slots] # currently occupied buildings
```

---

## 2. `||=` Memoization — A Post-it Note That Hates Blank Spaces

Think of `||=` as a sticky note system. When you check the note and it has something written on it, you read it. When it's blank (nil) or says "no" (false), you erase it and write a new value.

The problem: for people who legitimately wrote "no" or "0" or left it intentionally blank, the note system keeps erasing their answer and sending them back to get a new one. It can't distinguish between "nothing written yet" and "intentionally wrote false."

The fix is to use a different check: "Has anyone ever written on this note?" (`key?` check or `defined?`), not "Does this note have a truthy value?"

```ruby
@memoized ||= compute    # "is the note blank or saying no? → write new value"
# Vs:
defined?(@memoized) ? @memoized : (@memoized = compute)  # "was the note ever written?"
```

---

## 3. Object Allocation and GC Pressure — Opening and Closing Bank Accounts

Every Ruby object is like a bank account. Creating an object opens an account. The GC closes accounts that nobody references anymore.

`String#+` in a loop is like a bank where every transaction opens a brand new account, makes a transfer, then abandons the old account — leaving thousands of zombie accounts to be cleaned up later. The bank's cleanup crew (GC) has to occasionally pause operations to audit and close all those zombie accounts.

`String#<<` is like a single account you keep depositing into. One account, one cleanup record. The bank's audit crew barely notices it.

`#frozen_string_literal: true` is like a credit union that allows multiple people to share read-only accounts for identical amounts — "hello" doesn't need its own account for each person who has exactly $500.

---

## 4. Lazy Evaluation — A Buffet Line vs a Made-to-Order Kitchen

Eager evaluation (`map`, `select`) is like a buffet: all 1,000 dishes are prepared before anyone arrives. If only 3 people show up and each take one item, 997 dishes were cooked for nothing.

Lazy evaluation is a made-to-order kitchen. The menu is planned (the lazy chain is declared), but nothing is cooked until a specific order comes in. When someone says "give me 10 vegetarian options from this menu," the kitchen cooks items one by one until 10 vegetarian dishes are ready, then stops — even if there were 1,000 items on the menu.

```ruby
# Buffet: cooks all 1M items, serves 10
(1..1_000_000).select { |n| n.odd? }.first(10)

# Made-to-order: cooks 19 items, finds 10 odd ones, stops
(1..1_000_000).lazy.select { |n| n.odd? }.first(10)
```

---

## 5. `benchmark-ips` — A Speed Trap vs a Long-Distance Average

The standard `Benchmark.measure` is like timing a car with a speed gun from a single fixed point. You get one data point — was the car going 60mph at that moment? But was that representative? Was the driver braking? Accelerating? Was there a GC pause?

`benchmark-ips` is like monitoring a race track over the full session. It automatically determines how many laps to do to get statistically meaningful results, discards the warmup laps (YJIT compilation, cache warmup), and gives you iterations per second with a confidence interval (±%). The ± tells you how consistent the results are — low variance means reliable measurement.

---

## 6. YJIT — A Translator Who Gets Better the More You Talk

Ruby's interpreter normally works like a live translator at a conference — every time a speaker says a phrase, the translator renders it into the target language on the spot, every single time.

YJIT is a translator who takes notes. The first time a method is called, it's translated normally AND the native translation is written down. For very frequent phrases (hot methods), YJIT compiles the phrase directly into the target language's native words (machine code). Next time that exact phrase appears, there's no translation needed — the native text is read directly.

The catch: YJIT only helps with the translation step. If the conference speaker is waiting for a response from the audience (I/O wait — database queries, network calls), making the translator faster doesn't help — the bottleneck is the audience's response time.

---

## 7. Memory Profiler — A Blood Test Panel

When a doctor suspects something is wrong, they don't guess — they order a blood test panel that measures 30+ biomarkers simultaneously. The panel tells you: total cholesterol, HDL, LDL, blood sugar, liver enzymes, etc. High numbers in the right combination point to the specific problem.

`MemoryProfiler.report` is a blood test for your Ruby code:
- **Total allocated**: how much was created (your metabolic rate)
- **Total retained**: what was never released (what's still in your bloodstream)
- **Allocated by class**: which specific compound is accumulating (String, Hash, etc.)
- **Allocated by file/line**: which organ (code path) is producing too much

A high "retained" number with specific classes identified is like high liver enzymes — it points you to exactly where to investigate further.

---

## 8. `String#<<` vs `String#+` — Sculpting Clay vs Printing Forms

`String#<<` (mutating concatenation) is like sculpting clay. You start with one lump and keep adding to it, reshaping it. One lump, continuous work, minimal waste.

`String#+` (creating a new string each time) is like printing forms. Each time you want to add information, you print a new form with all the old information plus the new part, then throw away the old form. After 1000 additions, you've printed and discarded 999 forms. The shredder (GC) has to clean up all that paper.

```ruby
# Sculpting clay (one object, mutated):
buffer = +""
1000.times { |i| buffer << "entry #{i}\n" }

# Printing forms (1000 new strings):
result = ""
1000.times { |i| result = result + "entry #{i}\n" }  # 1000 new strings!
```

---

## 9. The GC Tuning Knobs — Adjusting a Car's Suspension

Ruby's GC tuning environment variables are like a car's suspension settings. The car (Ruby process) drives on different roads (workloads), and the right suspension depends on the terrain.

`RUBY_GC_HEAP_GROWTH_FACTOR` is the spring stiffness — a softer spring (higher factor) means the heap grows more aggressively when objects accumulate, reducing how often the GC needs to run (fewer bumps), but using more memory overall.

`RUBY_GC_HEAP_INIT_SLOTS` is like pre-inflating the tires before a race — starting with a larger heap means the GC doesn't fire too early during boot, reducing GC runs in the critical first seconds.

The right settings depend on your app's profile: a batch job that processes millions of objects benefits from a large, aggressive heap. A tiny API server with few concurrent objects benefits from conservative settings that keep memory low.

---

## 10. `stackprof` Sampling — A Detective Taking Random Photos

Imagine a detective investigating where employees spend their time. Instead of following every person every second (too invasive — like ruby-prof instrumentation), the detective takes a random snapshot of the office floor 1000 times throughout the day. Each snapshot shows exactly where everyone is standing.

After 1000 snapshots, the analysis is straightforward: if Alice is at the coffee machine in 400 out of 1000 snapshots, she spends ~40% of her day there. If the report generation function appears in the call stack in 300 snapshots, it's consuming ~30% of CPU time — and worth optimizing.

The detective's approach is non-invasive (low overhead), statistically accurate, and points directly to the hotspots. The occasional false negative (the detective missed a brief expensive call) is acceptable for the huge reduction in overhead.
