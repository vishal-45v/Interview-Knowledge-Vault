# Chapter 04 — Collections & Enumerable: Diagram Explanations

---

## Diagram 1 — Enumerable Pipeline

```
  INPUT ARRAY:  [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

  STEP 1: .select { |n| n.even? }
  ┌──────────────────────────────────────────┐
  │  1 → block(1)=false → DISCARD           │
  │  2 → block(2)=true  → KEEP             │
  │  3 → block(3)=false → DISCARD           │
  │  ...                                    │
  └──────────────────────────────────────────┘
  Result: [2, 4, 6, 8, 10]

  STEP 2: .map { |n| n * n }
  ┌──────────────────────────────────────────┐
  │  2 → 4                                  │
  │  4 → 16                                 │
  │  6 → 36                                 │
  │  8 → 64                                 │
  │ 10 → 100                                │
  └──────────────────────────────────────────┘
  Result: [4, 16, 36, 64, 100]

  STEP 3: .sum
  4 + 16 + 36 + 64 + 100 = 220
```

---

## Diagram 2 — Lazy Evaluation Chain

```
  EAGER:
  (1..1_000_000)
       │
       ▼ .select { odd? }      ← builds array of 500,000 elements
       │
       ▼ .map { n*n }          ← builds array of 500,000 elements
       │
       ▼ .first(5)             ← picks 5 from 500,000 element array
       └→ [1, 9, 25, 49, 81]

  ═══════════════════════════════════════════════════

  LAZY:
  (1..1_000_000)
       │ .lazy
       │
    ┌──┴──────────────────────────────────────────┐
    │  ELEMENT-BY-ELEMENT PROCESSING              │
    │                                             │
    │  1 → odd? YES → 1*1=1   → collect → count=1│
    │  2 → odd? NO  → skip                        │
    │  3 → odd? YES → 3*3=9   → collect → count=2│
    │  5 → odd? YES → 5*5=25  → collect → count=3│
    │  7 → odd? YES → 7*7=49  → collect → count=4│
    │  9 → odd? YES → 9*9=81  → collect → count=5│
    │                          ← STOP (count=5)   │
    └─────────────────────────────────────────────┘
       └→ [1, 9, 25, 49, 81]
       Only 9 elements processed!
```

---

## Diagram 3 — inject (reduce) Step by Step

```
  [1, 2, 3, 4, 5].inject { |sum, n| sum + n }

  Iteration 0: memo = 1 (first element), n = 2
               sum = 1 + 2 = 3

  Iteration 1: memo = 3, n = 3
               sum = 3 + 3 = 6

  Iteration 2: memo = 6, n = 4
               sum = 6 + 4 = 10

  Iteration 3: memo = 10, n = 5
               sum = 10 + 5 = 15

  Result: 15

  ─────────────────────────────────────────────

  With initial value: [1,2,3,4,5].inject(100, :+)

  Iteration 0: memo = 100, n = 1  → 101
  Iteration 1: memo = 101, n = 2  → 103
  ...
  Result: 115
```

---

## Diagram 4 — Set vs Array: Membership Testing

```
  ARRAY lookup (O(n)):
  valid = ["red", "green", "blue", "yellow", "purple"]

  "blue" in valid?
  Check: "red"?    NO
  Check: "green"?  NO
  Check: "blue"?   YES! ← stop here (position 3)

  Worst case: scan ALL n elements
  Average:    scan n/2 elements

  ─────────────────────────────────────────

  SET lookup (O(1)):
  valid = Set.new(["red", "green", "blue", "yellow", "purple"])

  "blue" in valid?
  hash("blue") → bucket #47
  Look in bucket #47 → "blue" found!

  Always: ~1 operation regardless of set size

  ─────────────────────────────────────────

  Performance at scale:
  n = 1,000,000 elements
  Array:  avg 500,000 comparisons per lookup
  Set:    ~1 comparison per lookup
  Speedup: 500,000x
```

---

## Diagram 5 — group_by Structure

```
  orders = [{type: :credit, amount: 100},
            {type: :debit,  amount: 50 },
            {type: :credit, amount: 200}]

  orders.group_by { |o| o[:type] }

  ┌─────────────────────────────────────────────────────┐
  │  :credit ──→ [{type: :credit, amount: 100},         │
  │               {type: :credit, amount: 200}]         │
  │                                                     │
  │  :debit  ──→ [{type: :debit, amount: 50}]           │
  └─────────────────────────────────────────────────────┘

  Then transform:
  .transform_values { |orders| orders.sum { |o| o[:amount] } }

  ┌────────────────────────┐
  │  :credit ──→ 300       │
  │  :debit  ──→ 50        │
  └────────────────────────┘
```

---

## Diagram 6 — each_slice vs each_cons

```
  arr = [1, 2, 3, 4, 5, 6]

  each_slice(3) — non-overlapping chunks:
  [1, 2, 3]  [4, 5, 6]

  each_cons(3) — sliding window:
  [1, 2, 3]
     [2, 3, 4]
        [3, 4, 5]
           [4, 5, 6]

  ─────────────────────────────────────────────────
  Use case for each_slice: batch processing API calls
  user_ids.each_slice(100) { |batch| api.bulk_fetch(batch) }

  Use case for each_cons: rolling average
  prices.each_cons(3).map { |window| window.sum.to_f / 3 }
```

---

## Diagram 7 — chunk_while: Consecutive Groups

```
  data = [1, 2, 3, 5, 6, 10, 11, 12]

  chunk_while { |a, b| b == a + 1 }

  Walk the pairs:
  (1,2): 2 == 1+1? YES → same group
  (2,3): 3 == 2+1? YES → same group
  (3,5): 5 == 3+1? NO  → CUT here
  (5,6): 6 == 5+1? YES → same group
  (6,10): 10 == 6+1? NO → CUT here
  (10,11): 11 == 10+1? YES → same group
  (11,12): 12 == 11+1? YES → same group

  Result: [[1,2,3], [5,6], [10,11,12]]
```

---

## Diagram 8 — sort_by Schwartzian Transform

```
  SORT WITH BLOCK — key computed multiple times:
  words.sort { |a, b| a.length <=> b.length }

  For n=5 words, ~12 comparisons needed:
  Compare apple(5) vs banana(6) → compute length twice
  Compare banana(6) vs cherry(6) → compute length twice
  ...
  Total: ~2 * n * log(n) = ~24 length computations

  ─────────────────────────────────────────────────

  SORT_BY — key computed once per element:
  words.sort_by { |w| w.length }

  STEP 1: Compute keys (once per element):
  apple   → 5
  banana  → 6
  cherry  → 6
  date    → 4
  elderberry → 10

  STEP 2: Sort by pre-computed keys:
  [4, 5, 6, 6, 10]

  Total: n = 5 length computations (not 24)

  Speedup factor: ~2 * log(n) improvement
  For 1000 elements: ~20x fewer key computations
```
