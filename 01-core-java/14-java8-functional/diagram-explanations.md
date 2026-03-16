# Java 8 Functional — Diagram Explanations

---

## Diagram 1: Stream Pipeline

```
  Source
  List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
       │
       ▼ Intermediate (lazy)
  .filter(n -> n % 2 == 0)   [2, 4, 6, 8, 10]
       │
       ▼ Intermediate (lazy)
  .map(n -> n * n)            [4, 16, 36, 64, 100]
       │
       ▼ Intermediate (lazy)
  .limit(3)                   [4, 16, 36]
       │
       ▼ Terminal (triggers execution)
  .collect(Collectors.toList())
       │
       ▼
  [4, 16, 36]

  Note: Execution is short-circuited by limit(3)
  Only processes: 1(filtered), 2(passes), 4(mapped), 3(filtered), 4(passes), 16(mapped), 5(filtered), 6(passes), 36(mapped)
  Then limit(3) satisfied → stop
```

---

## Diagram 2: map vs flatMap

```
  map: one-to-one transformation
  ["hello", "world"]
       │
       ▼ .map(s -> s.chars().boxed())
  [IntStream("hello"), IntStream("world")]  ← Stream<IntStream>

  flatMap: one-to-many, then flatten
  ["hello", "world"]
       │
       ▼ .flatMap(s -> s.chars().boxed())
  [104, 101, 108, 108, 111, 119, 111, 114, 108, 100]  ← Stream<Integer>
```

---

## Diagram 3: Optional Chain

```
  Optional<User> user = findUser(id)
       │
       │ (may be empty)
       ▼
  .map(User::getAddress)    → Optional<Address>
       │
       │ (still may be empty)
       ▼
  .map(Address::getCity)    → Optional<String>
       │
       ▼
  .orElse("Unknown")        → String (never null)

  At each step, if empty → stays empty and short-circuits
  No NullPointerException possible in this chain
```
