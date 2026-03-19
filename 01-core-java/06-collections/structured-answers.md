# Collections — Structured Answers

---

## Q1: How does HashMap work internally in Java?

**Answer:**

HashMap uses an array of **buckets** (called `table`), where each bucket holds a linked list (or a red-black tree in Java 8+ for large buckets).

**Step-by-step put(key, value):**

1. Compute `key.hashCode()`
2. Apply hash spreading: `hash = (h = key.hashCode()) ^ (h >>> 16)` (reduces collision from poor hashCodes)
3. Compute bucket index: `index = hash & (capacity - 1)`
4. If bucket is empty → insert directly
5. If bucket has entries → check each entry's key using `equals()`
   - If key match found → update value
   - If no match → append new entry (linked list) or insert into tree (if bucket size ≥ 8)

**Default capacity:** 16
**Default load factor:** 0.75
**Rehash threshold:** capacity × load factor (e.g., 16 × 0.75 = 12 entries triggers resize)

**On resize:** Capacity doubles, all entries are rehashed into the new array.

**Java 8 optimization:** When a bucket has ≥ 8 entries AND total size ≥ 64, the linked list converts to a red-black tree (O(log n) instead of O(n) for worst-case lookups).

---

## Q2: What happens if hashCode() always returns the same value?

**Answer:**

All keys land in the same bucket. The HashMap degrades to a **linked list (pre-Java 8)** or **red-black tree (Java 8+)**.

- Pre-Java 8: O(n) for get/put — catastrophic performance
- Java 8+: O(log n) after treeification threshold — bad but survivable

This is also a **security vulnerability** — a malicious client can cause a DoS attack by crafting input that generates hash collisions, overwhelming the server. Java 8 addressed this by using tree bins.

**Rule:** Always implement `equals()` and `hashCode()` together. If two objects are `equal()`, they MUST have the same `hashCode()`.

---

## Q3: What is the difference between ArrayList and LinkedList?

**Answer:**

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| Random access (`get(i)`) | O(1) | O(n) |
| Add to end (`add(e)`) | O(1) amortized | O(1) |
| Add to middle (`add(i, e)`) | O(n) — shift elements | O(n) — traverse to position |
| Remove from middle | O(n) — shift elements | O(n) — traverse to position |
| Memory | Compact (array) | Higher (node + prev + next pointers) |
| Cache locality | Excellent | Poor (nodes scattered in heap) |

**When to use ArrayList:** Random access, iteration, most common use cases.
**When to use LinkedList:** Frequent insertions/deletions at the head (use as Deque), implementing queues.

**Practical advice:** In modern Java, `ArrayList` wins in almost all cases due to cache locality. `LinkedList` as a List is rarely the right choice.

---

## Q4: What is the difference between fail-fast and fail-safe iterators?

**Answer:**

**Fail-fast iterators** (ArrayList, HashMap, HashSet):
- Throw `ConcurrentModificationException` if the collection is modified while iterating
- Use an internal `modCount` counter; iterator checks if `modCount` changed since iteration started
- Does NOT guarantee to always throw — it's "best-effort"

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {
    list.remove(s); // ❌ throws ConcurrentModificationException
}
```

**Fail-safe iterators** (ConcurrentHashMap, CopyOnWriteArrayList):
- Operate on a **copy** of the collection (snapshot)
- Never throw `ConcurrentModificationException`
- May not reflect recent modifications

```java
List<String> list = new CopyOnWriteArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {
    list.remove(s); // ✅ No exception — iterates over original snapshot
}
```

---

## Q5: How is ConcurrentHashMap different from HashMap and Hashtable?

**Answer:**

| Feature | HashMap | Hashtable | ConcurrentHashMap |
|---------|---------|-----------|-------------------|
| Thread-safe | ❌ No | ✅ Yes (synchronized) | ✅ Yes (segment/bucket locking) |
| Null keys | ✅ 1 null key | ❌ No nulls | ❌ No nulls |
| Null values | ✅ Yes | ❌ No nulls | ❌ No nulls |
| Performance | Fast | Slow (whole map locked) | Very fast |
| Iterator | Fail-fast | Fail-fast | Weakly consistent |

**ConcurrentHashMap internals (Java 8+):**
- Uses lock striping — locks only the individual bucket being modified
- `get()` operations are lock-free (using volatile)
- `put()` uses CAS (Compare-And-Swap) for the first insertion, synchronized for chaining
- Provides atomic operations: `putIfAbsent()`, `computeIfAbsent()`, `merge()`

---

## Q6: When should you use TreeMap vs HashMap?

**Answer:**

**HashMap:** O(1) average for get/put — use when order doesn't matter and you need speed.

**TreeMap:** O(log n) for get/put — use when:
- You need **sorted keys** (natural order or custom Comparator)
- You need range queries: `subMap()`, `headMap()`, `tailMap()`
- You need `firstKey()`, `lastKey()`, `floorKey()`, `ceilingKey()`

```java
TreeMap<String, Integer> scores = new TreeMap<>();
scores.put("Charlie", 85);
scores.put("Alice", 92);
scores.put("Bob", 78);

System.out.println(scores); // {Alice=92, Bob=78, Charlie=85} — sorted
System.out.println(scores.headMap("Bob")); // {Alice=92} — keys before "Bob"
```

---

## Q7: What is the internal structure of HashSet?

**Answer:**

`HashSet` is backed by a `HashMap`. When you add an element to a `HashSet`, it's stored as a **key** in the internal HashMap with a constant dummy `Object` (`PRESENT`) as the value.

```java
// Simplified internal view
private transient HashMap<E, Object> map;
private static final Object PRESENT = new Object();

public boolean add(E e) {
    return map.put(e, PRESENT) == null;
}
```

This means:
- All HashMap rules apply: O(1) average for add/contains/remove
- Elements must properly implement `hashCode()` and `equals()`
- No duplicates (same dedup logic as HashMap keys)

---

## Q8: How does PriorityQueue work? What are its time complexities?

**Answer:**

`PriorityQueue` is a **min-heap** by default (smallest element at the top).

**Internal structure:** Binary heap stored in an array.

**Time complexities:**
- `offer(e)` / `add(e)`: O(log n) — bubble up
- `poll()`: O(log n) — remove root, bubble down
- `peek()`: O(1) — just look at root
- `contains(e)`: O(n) — linear scan
- `size()`: O(1)

**Custom ordering:**
```java
// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// Custom object ordering
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
);
```

---

## Q9: What is the difference between Iterator and ListIterator?

**Answer:**

| Feature | Iterator | ListIterator |
|---------|----------|--------------|
| Applies to | All Collections | Only List |
| Direction | Forward only | Forward and backward |
| Methods | `hasNext()`, `next()`, `remove()` | + `hasPrevious()`, `previous()`, `add()`, `set()`, `nextIndex()`, `previousIndex()` |
| Can add elements | ❌ No | ✅ Yes |
| Can modify elements | ❌ No (remove only) | ✅ Yes (`set()`) |

---

## Q10: Explain the Collections utility class — key methods

**Answer:**

`java.util.Collections` provides static utility methods:

```java
// Sorting
Collections.sort(list);                    // natural order
Collections.sort(list, comparator);        // custom order

// Binary search (list must be sorted)
int idx = Collections.binarySearch(list, target); // O(log n)

// Shuffling
Collections.shuffle(list);
Collections.shuffle(list, new Random(seed));

// Min/Max
Collections.min(list);
Collections.max(list);

// Frequency
Collections.frequency(list, element);

// Reverse
Collections.reverse(list);

// Fill
Collections.fill(list, value);

// Copy
Collections.copy(destination, source);

// Immutable views (NOT truly immutable in older Java — use List.of() in Java 9+)
List<String> unmodifiable = Collections.unmodifiableList(list);

// Synchronized wrappers (use ConcurrentHashMap instead for performance)
Map<K, V> syncMap = Collections.synchronizedMap(new HashMap<>());

// Singleton collections
List<String> single = Collections.singletonList("only");
Set<String> emptySet = Collections.emptySet();
```

---

## Q11: What is the difference between `Arrays.asList()` and `List.of()`?

**Answer:**

| Feature | `Arrays.asList()` | `List.of()` (Java 9+) |
|---------|-------------------|-----------------------|
| Null elements | Allowed | ❌ Throws NullPointerException |
| Modifiable | Partially (set allowed, add/remove not) | Fully immutable |
| Backed by array | Yes — reflects array changes | No |
| Serializable | Yes | Yes |

```java
List<String> asList = Arrays.asList("a", "b", "c");
asList.set(0, "x");  // ✅ OK
asList.add("d");     // ❌ UnsupportedOperationException

List<String> listOf = List.of("a", "b", "c");
listOf.set(0, "x"); // ❌ UnsupportedOperationException
listOf.add("d");     // ❌ UnsupportedOperationException
```

---

## Q12: How would you choose between different Map implementations?

**Answer:**

```
Need ordering?
├── No  → HashMap (O(1), best general purpose)
│         LinkedHashMap (insertion/access order, O(1))
└── Yes → TreeMap (sorted by key, O(log n))
          EnumMap (enum keys, fastest for enums)

Thread safety needed?
├── No  → HashMap (fastest)
└── Yes → ConcurrentHashMap (high concurrency)
           Collections.synchronizedMap() (simple but slower)

Special cases:
- WeakHashMap: keys held weakly (cache-like, GC can remove entries)
- IdentityHashMap: uses == instead of equals() for key comparison
```
