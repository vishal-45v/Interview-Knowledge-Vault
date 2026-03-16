# Strings & Regex — ASCII Diagram Explanations

---

## 1. String Pool vs Heap (Literal vs new String())

```
  Source code:
    String a = "hello";
    String b = "hello";
    String c = new String("hello");
    String d = "hell" + "o";    // compile-time constant folding
    String e = getPrefix() + "o"; // runtime concatenation

  ┌─────────────────────────────────────────────────────────────────┐
  │                           JVM HEAP                              │
  │                                                                 │
  │   ┌─────────────────────────────────────────────────────────┐   │
  │   │               STRING POOL (intern pool)                 │   │
  │   │                                                         │   │
  │   │   ┌────────────────┐                                    │   │
  │   │   │  String "hello"│  ◄── a points here                │   │
  │   │   │  value: [h,e,l,│  ◄── b points here (SAME object!) │   │
  │   │   │         l,o]   │  ◄── d points here (constant fold)│   │
  │   │   └────────────────┘                                    │   │
  │   └─────────────────────────────────────────────────────────┘   │
  │                                                                 │
  │   ┌────────────────┐                                            │
  │   │  String object │  ◄── c points here (different object!)    │
  │   │  value: [h,e,l,│                                            │
  │   │         l,o]   │                                            │
  │   └────────────────┘                                            │
  │                                                                 │
  │   ┌────────────────┐                                            │
  │   │  String object │  ◄── e points here (runtime concat)       │
  │   │  value: [h,e,l,│                                            │
  │   │         l,o]   │                                            │
  │   └────────────────┘                                            │
  └─────────────────────────────────────────────────────────────────┘

  Comparisons:
  a == b  → true   (same pool object)
  a == c  → false  (c is a different heap object)
  a == d  → true   (compile-time constant, placed in pool)
  a == e  → false  (runtime computation, not pooled)
  a.equals(c) → true (content comparison)

  c.intern() → returns pool reference (same as a, b, d)
  After: String f = c.intern();
         a == f → true
```

---

## 2. StringBuilder Internal char[] Resizing

```
  new StringBuilder()   ← default capacity = 16 chars
  ┌────────────────────────────────────────────────────┐
  │  char[] buf = new char[16]                         │
  │  [ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ] │
  │  len = 0                                           │
  └────────────────────────────────────────────────────┘

  After .append("Hello World!") (12 chars)
  ┌────────────────────────────────────────────────────┐
  │  [H][e][l][l][o][ ][W][o][r][l][d][!][ ][ ][ ][ ] │
  │  len = 12,  capacity = 16  (4 slots remaining)     │
  └────────────────────────────────────────────────────┘

  After .append(" More text") (10 more chars = 22 total > 16)
  ┌────────────────────────────────────────────────────────────────┐
  │  Growth formula: newCapacity = (oldCapacity * 2) + 2 = 34      │
  │                                                                │
  │  1. Allocate new char[34]                                      │
  │  2. System.arraycopy(old buf → new buf)                        │
  │  3. Append new chars                                           │
  │  4. Discard old buf (eligible for GC)                          │
  │                                                                │
  │  [H][e][l][l][o][ ][W][o][r][l][d][!][ ][M][o][r][e]         │
  │  [ ][t][e][x][t][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ][ ]...         │
  │  len = 22,  capacity = 34                                      │
  └────────────────────────────────────────────────────────────────┘

  .toString() at end: creates ONE final String from the buffer
  ┌────────────────────┐
  │ Immutable String   │
  │ "Hello World! More │
  │  text"             │
  └────────────────────┘

  Pre-sizing eliminates all resizing copies:
  new StringBuilder(estimatedSize)   ← supply known capacity upfront
```

---

## 3. String Immutability (New Object on Each Operation)

```
  String s = "Java";

  Heap: [Object-1: "Java"]
         ↑
         s

  s = s.toUpperCase();

  Heap: [Object-1: "Java"]   ← still exists until GC
        [Object-2: "JAVA"]
                              ↑
                              s (now points to new object)

  s = s.concat(" Rocks");

  Heap: [Object-1: "Java"]        ← GC eligible (no references)
        [Object-2: "JAVA"]        ← GC eligible
        [Object-3: "JAVA Rocks"]
                                   ↑
                                   s

  s = s.substring(0, 4);

  Heap: ... (Object-3 eligible)
        [Object-4: "JAVA"]
                            ↑
                            s

  Total objects created for 3 operations: 4
  (Original + 3 new ones)

  Why this matters for security:
  ┌──────────────────────────────────────────────────────────────┐
  │  void authenticate(String password) {                        │
  │      // password cannot be altered by this method            │
  │      // no method can change the string the caller passed    │
  │      if (!validate(password)) throw new AuthException();     │
  │  }                                                           │
  │                                                              │
  │  vs char[] password — mutable, can be zeroed after use:      │
  │  Arrays.fill(password, '\0'); // clear sensitive data        │
  │  // String cannot be zeroed — lives in memory until GC       │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4. Regex Engine Matching Flow (NFA Concepts)

```
  Pattern: "a(b+)c"
  Input:   "abbc"

  NFA State Machine (simplified):

  Start ──'a'──► State1 ──'b'──► State2 ──'b'──► State2 (loop)
                                                  │
                                             'c'──►State3 (ACCEPT)

  Matching trace for "abbc":
  ┌──────────────────────────────────────────────────────────────┐
  │  Position 0: 'a'                                             │
  │    State: Start → 'a' matches → move to State1               │
  │                                                              │
  │  Position 1: 'b'                                             │
  │    State: State1 → 'b' matches → move to State2 (group 1)   │
  │    Group 1 capture begins at pos 1                           │
  │                                                              │
  │  Position 2: 'b'                                             │
  │    State: State2 → 'b' matches → stay in State2 (+ quantifier│
  │    Group 1 capture extends to pos 2                          │
  │                                                              │
  │  Position 3: 'c'                                             │
  │    State: State2 → try 'b': fails → try 'c': succeeds        │
  │    Group 1 capture ends at pos 2 (exclusive)                 │
  │    Move to State3 (ACCEPT)                                   │
  │                                                              │
  │  Result: MATCH                                               │
  │    group(0) = "abbc"  (full match)                           │
  │    group(1) = "bb"    (captured group 1)                     │
  └──────────────────────────────────────────────────────────────┘

  Catastrophic Backtracking Example:
  Pattern: "(a+)+"
  Input:   "aaaaaaaaaaX"  (10 a's + X that won't match)

  ┌──────────────────────────────────────────────────────────────┐
  │  Outer + tries: (group matches aaaaaaaaaa), then 'X' fails    │
  │  Backtrack: outer tries (aaaaaaaaaa) split differently...    │
  │  Number of paths explored: 2^10 = 1024 attempts              │
  │  With 30 chars: 2^30 = 1 billion attempts → engine hangs     │
  └──────────────────────────────────────────────────────────────┘
```

---

## 5. String.intern() — Heap to Pool Reference

```
  Without intern():
  ─────────────────
  String s1 = new String("config"); // explicit heap object
  String s2 = new String("config"); // another heap object

  Pool: ["config"]  ← exists because of the literal "config" in source

  Heap: [String "config" @0x100]  ← s1
        [String "config" @0x200]  ← s2

  s1 == s2  → false (different objects)
  s1.equals(s2) → true

  After intern():
  ──────────────
  String s3 = s1.intern(); // or s2.intern()

  Process:
  1. intern() checks the pool for a string equal to s1 ("config")
  2. Found! Returns the existing pool reference
  3. s3 now points to the POOL object

  Pool: ["config" @0x050]  ← s3 (and all literals)
  Heap: [String "config" @0x100]  ← s1 (eligible for GC if s1 reassigned)
        [String "config" @0x200]  ← s2

  s3 == "config"  → true   (both point to pool object @0x050)
  s1 == s3        → false  (s1 is still heap @0x100)
  s1.intern() == s3 → true (both return pool reference)

  Memory profile when intern() helps:
  ┌──────────────────────────────────────────────────────────┐
  │  Without intern: 1M strings "US" → 1M objects in heap    │
  │  With intern:    1M strings "US" → 1 pool entry +        │
  │                                    1M references to it   │
  │  Memory saved: ~(1M - 1) * (object header + char array)  │
  │               ≈ 1M * 56 bytes = 56 MB saved              │
  └──────────────────────────────────────────────────────────┘

  Risk with intern() in modern Java:
  ┌──────────────────────────────────────────────────────────┐
  │  Java 6 and below: pool was in PermGen (fixed size)       │
  │  Interning many strings → OutOfMemoryError: PermGen space │
  │                                                          │
  │  Java 7+: pool moved to main heap (GC managed)           │
  │  Java 8+: PermGen replaced by Metaspace                  │
  │  Risk reduced but pool still holds strong references      │
  │  → interned strings are never GC'd                        │
  └──────────────────────────────────────────────────────────┘
```
