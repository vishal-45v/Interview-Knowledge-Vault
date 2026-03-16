# Collections — Theory Questions

> 32 theory questions covering the Java Collections Framework internals.

---

## Q1: What is the Java Collections Framework hierarchy?

**Answer:**

```
java.util.Collection (interface)
├── List (interface)
│   ├── ArrayList
│   ├── LinkedList
│   ├── Vector (legacy)
│   │   └── Stack (legacy)
│   └── CopyOnWriteArrayList (concurrent)
├── Set (interface)
│   ├── HashSet
│   ├── LinkedHashSet
│   └── TreeSet (implements SortedSet)
└── Queue (interface)
    ├── LinkedList
    ├── PriorityQueue
    ├── ArrayDeque
    └── BlockingQueue (interface)
        ├── ArrayBlockingQueue
        ├── LinkedBlockingQueue
        └── PriorityBlockingQueue

java.util.Map (interface) — NOT a Collection
├── HashMap
├── LinkedHashMap
├── TreeMap (implements SortedMap)
├── Hashtable (legacy)
├── ConcurrentHashMap (concurrent)
└── WeakHashMap
```

---

## Q2: How does HashMap work internally?

**Answer:**

HashMap stores entries in an array of "buckets" (an array of `Node<K,V>[]`).

**put(key, value) process:**
1. `hashCode()` is called on the key
2. Hash is spread further: `hash = (h = key.hashCode()) ^ (h >>> 16)` (spreads high bits down to reduce collisions)
3. Bucket index: `index = hash & (capacity - 1)`
4. If bucket is empty: store node there
5. If bucket has entries: check each with `equals()` — update if key matches, append to chain if not

**Collision handling:**
- Java 7: Linked list per bucket (O(n) worst case)
- Java 8+: Convert linked list to Red-Black tree when bucket size reaches 8 (TREEIFY_THRESHOLD) → O(log n) worst case. Convert back to list when size falls to 6 (UNTREEIFY_THRESHOLD).

**Load factor and rehashing:**
- Default load factor: 0.75 (75% full triggers resize)
- Default initial capacity: 16
- When threshold exceeded: double capacity, rehash all entries
- Rehashing is O(n) — expensive! Initialize with expected capacity to avoid it.

```java
// Avoid rehashing for known large maps:
Map<String, Order> orders = new HashMap<>(1024, 0.75f);  // initial capacity, load factor

// Compute initial capacity to avoid any resizing:
int expectedEntries = 1000;
int initialCapacity = (int) (expectedEntries / 0.75) + 1;
Map<String, Order> map = new HashMap<>(initialCapacity);
```

---

## Q3: What is the difference between HashMap, LinkedHashMap, and TreeMap?

**Answer:**

| Feature | HashMap | LinkedHashMap | TreeMap |
|---------|---------|--------------|---------|
| Ordering | None | Insertion or access order | Natural/custom sorted order |
| Performance | O(1) average | O(1) average | O(log n) |
| Null key | Yes (1) | Yes (1) | No (NPE on compareTo) |
| Null values | Yes | Yes | Yes |
| Thread safety | No | No | No |
| Implements | Map | Map | SortedMap, NavigableMap |
| Use case | Fast lookup | LRU cache, ordered iteration | Range queries, sorted traversal |

```java
// LinkedHashMap as LRU cache:
Map<String, Object> lruCache = new LinkedHashMap<>(16, 0.75f, true) {  // accessOrder=true
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Object> eldest) {
        return size() > 100;  // evict when > 100 entries
    }
};

// TreeMap for range queries:
TreeMap<Integer, String> scores = new TreeMap<>();
scores.subMap(60, 100);       // entries with key 60 ≤ k < 100
scores.headMap(50);            // entries with key < 50
scores.tailMap(70);            // entries with key ≥ 70
scores.floorKey(85);           // largest key ≤ 85
scores.ceilingKey(85);         // smallest key ≥ 85
```

---

## Q4: What is the difference between ArrayList and LinkedList?

**Answer:**

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| Random access `get(i)` | O(1) | O(n) |
| Insert/delete at end | O(1) amortized | O(1) |
| Insert/delete at middle | O(n) (shift) | O(1) (once you have node ref) |
| Memory overhead | Low (array) | High (prev/next pointers per node) |
| Iteration | Fast (cache-friendly) | Slower (pointer chasing) |
| Implements | List | List, Deque |

**When to use LinkedList:**
- Frequent insertions/deletions at the beginning
- Need Deque operations (push/pop from both ends)
- **Rarely** — ArrayList outperforms LinkedList for most use cases due to CPU cache locality

**ArrayList internals:**
- Backed by `Object[]` array
- Initial capacity: 10
- Growth: `oldCapacity + (oldCapacity >> 1)` = 1.5x growth
- `trimToSize()` reduces array to exact size

---

## Q5: What is ConcurrentHashMap and how is it different from HashMap and Hashtable?

**Answer:**

| Feature | HashMap | Hashtable | ConcurrentHashMap |
|---------|---------|-----------|-------------------|
| Thread safety | No | Yes (coarse-grained lock) | Yes (fine-grained) |
| Lock granularity | None | Entire map | Per bucket/segment |
| Null keys/values | Yes | No | No |
| Performance (concurrent) | Unsafe | Poor | High |
| Iterators | Fail-fast | Fail-safe | Weakly consistent |

**ConcurrentHashMap (Java 8+) internals:**
- No more segments (deprecated from Java 7 segments)
- Uses CAS operations for most operations
- Synchronized only on the bucket head node during structural modifications
- `size()` uses a parallel sum of cells (approximate)
- Full concurrent reads — no locking
- Atomic operations: `putIfAbsent()`, `compute()`, `merge()`, `computeIfAbsent()`

```java
ConcurrentHashMap<String, List<Order>> map = new ConcurrentHashMap<>();

// Atomic put-if-absent:
map.putIfAbsent("userId1", new ArrayList<>());

// Atomic compute:
map.compute("userId1", (k, v) -> {
    if (v == null) v = new ArrayList<>();
    v.add(newOrder);
    return v;
});

// merge — for counters:
map.merge("event", 1L, Long::sum);
```

---

## Q6: What is the difference between fail-fast and fail-safe iterators?

**Answer:**

**Fail-fast iterators:** Throw `ConcurrentModificationException` if the collection is structurally modified after the iterator is created. Detected via a `modCount` field.

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
Iterator<String> it = list.iterator();
list.add("d");  // modify after creating iterator
it.next();      // ConcurrentModificationException!
```

**Fail-safe iterators:** Do not throw exception. Either iterate on a copy (CopyOnWriteArrayList) or tolerate modifications (ConcurrentHashMap uses weakly consistent iterator).

```java
// CopyOnWriteArrayList — iterates over snapshot
CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>(Arrays.asList("a", "b", "c"));
Iterator<String> it = cowList.iterator();
cowList.add("d");  // modifies the live list, not the snapshot
it.next();         // "a" — iterates original snapshot, no exception

// ConcurrentHashMap — weakly consistent
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
// Iterator reflects some state of map at time of creation
// May or may not see subsequent updates
```

---

## Q7: What is PriorityQueue and how does it order elements?

**Answer:**

`PriorityQueue` is a min-heap (by default) — `peek()` and `poll()` return the smallest element. Ordering uses natural ordering (Comparable) or a custom Comparator.

```java
// Min-heap (default — natural ordering):
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.add(5); minHeap.add(1); minHeap.add(3);
System.out.println(minHeap.poll());  // 1 (smallest)

// Max-heap — reverse ordering:
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.add(5); maxHeap.add(1); maxHeap.add(3);
System.out.println(maxHeap.poll());  // 5 (largest)

// Custom object ordering:
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
              .thenComparingLong(Task::getCreatedAt)  // tie-breaking
);

// Common interview problem — top K elements:
PriorityQueue<Integer> topK = new PriorityQueue<>();  // min-heap of size K
for (int num : nums) {
    topK.offer(num);
    if (topK.size() > k) topK.poll();  // remove smallest
}
// topK now contains K largest elements
```

**PriorityQueue internals:** Binary heap stored in an array. Parent at `i`, children at `2i+1` and `2i+2`. Heapify on insert and remove: O(log n).

---

## Q8: What is EnumMap and when should you use it?

**Answer:**

`EnumMap` is a map implementation optimized for enum keys. Backed by a simple array indexed by enum ordinal — no hashing overhead.

```java
enum DayOfWeek { MON, TUE, WED, THU, FRI, SAT, SUN }

EnumMap<DayOfWeek, List<Order>> scheduleByDay = new EnumMap<>(DayOfWeek.class);
scheduleByDay.put(DayOfWeek.MON, mondayOrders);

// Performance: O(1) all operations, extremely fast (array access)
// Memory: compact array, no hash overhead
// Ordering: natural enum declaration order
```

**Use whenever keys are enum values.** Much faster than HashMap for the same purpose.

---

## Q9: What is the Collections utility class and what are its key methods?

**Answer:**

`java.util.Collections` is a utility class with static methods for manipulating collections:

```java
// Sorting:
Collections.sort(list);                     // natural order
Collections.sort(list, comparator);         // custom order
Collections.reverse(list);                  // reverse in-place
Collections.shuffle(list);                  // randomize

// Finding:
Collections.min(collection);
Collections.max(collection);
Collections.binarySearch(sortedList, key);  // O(log n) — list must be sorted

// Modification:
Collections.fill(list, "default");          // fill with single value
Collections.copy(dest, src);               // copy (dest must be at least as large)
Collections.nCopies(3, "hello");           // [hello, hello, hello]
Collections.swap(list, i, j);

// Thread safety wrappers:
Collections.synchronizedList(new ArrayList<>());
Collections.synchronizedMap(new HashMap<>());
Collections.synchronizedSet(new HashSet<>());

// Immutable views:
Collections.unmodifiableList(list);         // throws on modification
Collections.unmodifiableMap(map);

// Singletons:
Collections.singletonList("one");           // immutable 1-element list
Collections.singleton("value");            // immutable 1-element set
Collections.singletonMap("key", "value");  // immutable 1-entry map

// Empty collections:
Collections.emptyList();   // immutable, shared singleton
Collections.emptyMap();
Collections.emptySet();

// Frequency and disjoint:
Collections.frequency(collection, "target");  // count occurrences
Collections.disjoint(coll1, coll2);           // true if no common elements
```

---

## Q10: How does HashSet work internally?

**Answer:**

`HashSet<E>` is backed by a `HashMap<E, Object>` where every element is stored as a key with a static dummy value (`PRESENT = new Object()`). All the uniqueness guarantee comes from HashMap's key uniqueness.

```java
// HashSet source (simplified):
public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // null return means new entry
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }
}
```

**Implication:** If you override `equals()` without `hashCode()`, HashSet will happily store duplicates because the hash lookup fails to find the existing element.

---

## Q11: What is the difference between `Iterator.remove()` and `Collection.remove()`?

**Answer:**

- `Iterator.remove()` — removes the last element returned by `next()` from the underlying collection. **Safe** during iteration.
- `Collection.remove(Object)` — removes by value. **Unsafe** during iteration (causes ConcurrentModificationException for fail-fast iterators).

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c", "b"));

// Correct — using iterator:
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if ("b".equals(it.next())) {
        it.remove();  // removes "b" safely
    }
}

// Also correct (Java 8+):
list.removeIf("b"::equals);

// Incorrect — ConcurrentModificationException:
for (String s : list) {
    if ("b".equals(s)) list.remove(s);  // throws CME
}
```

---

## Q12: What is the WeakHashMap?

**Answer:**

`WeakHashMap` stores keys using **weak references**. If no strong reference to a key exists (only the WeakHashMap holds it), the entry is eligible for garbage collection and will be removed automatically.

```java
WeakHashMap<Object, String> cache = new WeakHashMap<>();

Object key = new Object();
cache.put(key, "some large value");

System.out.println(cache.size());  // 1

key = null;  // remove the only strong reference to the key
System.gc();  // suggest GC (not guaranteed)

System.out.println(cache.size());  // may be 0 — key was GC'd, entry removed
```

**Use cases:**
- Object-keyed caches where entries should expire when the key object is no longer used
- Canonicalization mappings
- **Not for:** General purpose caches where you control lifetime (use Caffeine/Guava Cache)

---

## Q13: What is ArrayDeque and when should you use it over LinkedList?

**Answer:**

`ArrayDeque` is a resizable array-backed double-ended queue. More efficient than `LinkedList` as a stack or queue because:
1. No pointer overhead per element
2. Better CPU cache locality (contiguous memory)
3. No boxing for primitives

```java
// As a Stack (prefer over Stack class which is synchronized):
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);
stack.push(2);
int top = stack.pop();   // 2

// As a Queue:
Deque<String> queue = new ArrayDeque<>();
queue.offer("first");
queue.offer("second");
String head = queue.poll();  // "first"

// As a Deque (double-ended):
queue.offerFirst("zero");    // add to front
queue.offerLast("third");    // add to back
queue.peekFirst();
queue.peekLast();
```

**Rule:** Prefer `ArrayDeque` for stack/queue operations. Use `LinkedList` only when you need `List` operations alongside Deque operations.

---

## Q14: What are the immutable collection factory methods in Java 9+?

**Answer:**

```java
// Java 9+ factory methods — truly unmodifiable (no defensive copy needed):
List<String> list = List.of("a", "b", "c");         // immutable
Set<Integer> set = Set.of(1, 2, 3);                 // immutable, no duplicates allowed
Map<String, Integer> map = Map.of("a", 1, "b", 2);  // immutable

// For larger maps:
Map<String, Integer> bigMap = Map.ofEntries(
    Map.entry("key1", 1),
    Map.entry("key2", 2),
    Map.entry("key3", 3)
);

// Defensive copy to make existing collection immutable (Java 10+):
List<String> immutableCopy = List.copyOf(existingList);
Set<String> immutableSet = Set.copyOf(existingSet);
Map<String, Integer> immutableMap = Map.copyOf(existingMap);
```

**Characteristics of factory-created collections:**
- Null elements/keys/values not allowed
- Fixed size — `add()`, `remove()` throw `UnsupportedOperationException`
- Iteration order not guaranteed for Set and Map
- May return a different implementation based on size

---

## Q15: What is the difference between `Comparable` and `Comparator` for collections?

**Answer:**

```java
// Comparable — natural ordering defined in the class itself:
class Employee implements Comparable<Employee> {
    String name;
    double salary;

    @Override
    public int compareTo(Employee other) {
        return this.name.compareTo(other.name);  // sort by name
    }
}
List<Employee> employees = new ArrayList<>(/*...*/);
Collections.sort(employees);  // uses compareTo

// Comparator — external ordering strategy:
Comparator<Employee> bySalary = Comparator.comparingDouble(Employee::getSalary);
Comparator<Employee> bySalaryDesc = bySalary.reversed();
Comparator<Employee> byDeptThenSalary = Comparator
    .comparing(Employee::getDepartment)
    .thenComparingDouble(Employee::getSalary);

employees.sort(byDeptThenSalary);
TreeSet<Employee> sortedByDept = new TreeSet<>(byDeptThenSalary);
```

**Key rule:** Use `Comparable` when one canonical ordering makes sense for the class. Use `Comparator` when multiple orderings are needed or the class is not modifiable.

---

## Q16: How does the `equals()` and `hashCode()` contract affect collections?

**Answer:**

HashSet, HashMap, and their thread-safe counterparts rely entirely on correct `equals()` and `hashCode()`.

**If only `equals()` overridden (hashCode not):**
```java
// Objects that are equal may have different hashCodes
// → stored in different buckets
// → HashSet allows duplicates, HashMap loses entries
```

**If only `hashCode()` overridden (equals not):**
```java
// Objects with same hash go to same bucket
// But equals() from Object compares references
// → HashSet still allows duplicates unless they're literally the same object
```

**Correct contract:**
```java
@Override
public boolean equals(Object o) { ... }

@Override
public int hashCode() {
    // Must use SAME fields as equals!
    return Objects.hash(field1, field2);
}
```

**Mutation danger:** Never mutate fields used in `equals()`/`hashCode()` while the object is in a collection:
```java
Set<Point> set = new HashSet<>();
Point p = new Point(1, 2);
set.add(p);
p.setX(10);  // DANGER — now p is in the wrong bucket!
set.contains(p);  // returns false!
```

---

## Q17: What are the differences between `List.of()`, `Arrays.asList()`, and `Collections.unmodifiableList()`?

**Answer:**

| Feature | List.of() | Arrays.asList() | Collections.unmodifiableList() |
|---------|-----------|-----------------|-------------------------------|
| Null elements | Not allowed | Allowed | Depends on wrapped |
| add()/remove() | UnsupportedOp | UnsupportedOp | UnsupportedOp |
| set() (update) | UnsupportedOp | Allowed! | UnsupportedOp |
| Backed by array | No | Yes — fixed-size | No (view) |
| Serializable | Yes | Yes | Yes |
| Since | Java 9 | Java 1.2 | Java 1.2 |

```java
// Arrays.asList — can SET elements but not ADD/REMOVE:
List<String> asList = Arrays.asList("a", "b", "c");
asList.set(0, "x");  // OK — updates the array
asList.add("d");     // UnsupportedOperationException

// Sync with the underlying array:
String[] arr = {"a", "b", "c"};
List<String> list = Arrays.asList(arr);
arr[0] = "z";
System.out.println(list.get(0));  // "z" — same backing array!
```

---

## Q18: What is IdentityHashMap?

**Answer:**

`IdentityHashMap` uses reference equality (`==`) instead of `equals()` for key comparison. Two keys `k1` and `k2` are considered equal only if `k1 == k2` (same object reference).

```java
IdentityHashMap<String, Integer> map = new IdentityHashMap<>();
String s1 = new String("hello");
String s2 = new String("hello");

map.put(s1, 1);
map.put(s2, 2);
System.out.println(map.size());  // 2 — s1 != s2 (different objects)

// vs HashMap:
HashMap<String, Integer> hashMap = new HashMap<>();
hashMap.put(s1, 1);
hashMap.put(s2, 2);
System.out.println(hashMap.size());  // 1 — s1.equals(s2) is true
```

**Use cases:** Topology-preserving object graph traversals, circular reference detection in serializers, tracking object identity for debugging.

---

## Q19: What is the time complexity of common LinkedList operations?

**Answer:**

`LinkedList` is a doubly-linked list. Each node has `prev`, `next`, and `item` references.

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| `get(index)` | O(n) | Traverses from head or tail |
| `add(index, element)` | O(n) | Finding index is O(n), linking is O(1) |
| `add(element)` (end) | O(1) | Has tail reference |
| `addFirst(element)` | O(1) | Has head reference |
| `remove(index)` | O(n) | Must find node first |
| `removeFirst()/removeLast()` | O(1) | Direct head/tail access |
| `contains(element)` | O(n) | Linear scan |
| `iterator traversal` | O(n) | Each step is O(1) |

---

## Q20: What are the characteristics of CopyOnWriteArrayList?

**Answer:**

`CopyOnWriteArrayList` creates a fresh copy of the underlying array on every write operation. Reads use the existing array without any locking.

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
list.add("a");
list.add("b");

// Safe concurrent iteration — even if list is modified while iterating:
for (String item : list) {
    list.add("new");  // safe — adds to a new copy, not to the iteration snapshot
    System.out.println(item);
}
```

**Performance characteristics:**
- Read: O(1), no synchronization, no locking
- Write: O(n) — full array copy
- Iterator: snapshot semantics — never sees concurrent modifications

**When to use:**
- Read-heavy, write-rare scenarios (observer lists, static config lists)
- When concurrent modification would cause problems
- **Avoid when:** Writes are frequent — O(n) copy is expensive

---

## Q21: What is the difference between HashMap and Hashtable?

**Answer:**

| Feature | HashMap | Hashtable |
|---------|---------|-----------|
| Thread safety | No | Yes (synchronized every method) |
| Null key/value | Yes (1 null key) | No (NPE on null) |
| Performance | Better | Worse (sync overhead) |
| Iteration | Fail-fast | Fail-safe (Enumeration) |
| Inheritance | AbstractMap | Dictionary (legacy) |
| Since | Java 1.2 | Java 1.0 |

**Never use Hashtable in new code.** Use `ConcurrentHashMap` for thread-safe maps — it has much better concurrency characteristics.

---

## Q22: What is the NavigableMap interface?

**Answer:**

`NavigableMap<K,V>` extends `SortedMap` with navigation methods. `TreeMap` is its main implementation.

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "one"); map.put(3, "three"); map.put(5, "five"); map.put(7, "seven");

map.floorKey(4);      // 3 — largest key ≤ 4
map.ceilingKey(4);    // 5 — smallest key ≥ 4
map.lowerKey(5);      // 3 — largest key < 5
map.higherKey(5);     // 7 — smallest key > 5

map.firstKey();       // 1
map.lastKey();        // 7

map.subMap(3, true, 7, false);   // {3=three, 5=five} (inclusive lower, exclusive upper)
map.headMap(5);                   // {1=one, 3=three}
map.tailMap(5);                   // {5=five, 7=seven}

map.descendingMap();              // reversed view

// pollFirstEntry / pollLastEntry — remove and return:
Map.Entry<Integer, String> first = map.pollFirstEntry();  // removes and returns 1=one
```

---

## Q23: What is a Deque and what are its use cases?

**Answer:**

`Deque` (double-ended queue) supports inserting and removing from both ends. Pronounced "deck."

```java
Deque<String> deque = new ArrayDeque<>();

// Stack operations (LIFO):
deque.push("first");   // addFirst
deque.push("second");  // addFirst
deque.pop();           // removeFirst — "second"

// Queue operations (FIFO):
deque.offer("a");      // addLast
deque.offer("b");      // addLast
deque.poll();          // removeFirst — "a"

// Palindrome check using Deque:
Deque<Character> chars = new ArrayDeque<>();
for (char c : "racecar".toCharArray()) chars.addLast(c);
boolean isPalindrome = true;
while (chars.size() > 1) {
    if (chars.pollFirst() != chars.pollLast()) { isPalindrome = false; break; }
}
```

---

## Q24: When should you use TreeSet vs HashSet?

**Answer:**

| Criteria | HashSet | TreeSet |
|----------|---------|---------|
| Ordering | Unordered | Sorted (natural or comparator) |
| Performance | O(1) add/remove/contains | O(log n) |
| null elements | Yes (one) | No |
| Range operations | No | Yes (subSet, headSet, tailSet) |
| Use case | Fast lookup, no ordering needed | Sorted iteration, range queries |

```java
// HashSet — use when ordering doesn't matter
Set<String> userIds = new HashSet<>();
userIds.contains("user123");  // O(1) — optimal for membership check

// TreeSet — use when sorted order or range operations needed
TreeSet<Integer> scores = new TreeSet<>();
scores.first();           // min score
scores.last();            // max score
scores.headSet(60);       // scores below 60 (failing)
scores.tailSet(90);       // scores 90+ (A grade)
scores.subSet(70, 90);    // B grade range
```

---

## Q25: What happens during HashMap rehashing?

**Answer:**

When the number of entries exceeds `capacity × loadFactor`, HashMap doubles its capacity and rehashes all existing entries.

**Rehashing steps:**
1. Create new array: `newCapacity = oldCapacity * 2`
2. For each existing bucket, for each node:
   - Recompute bucket index: `newIndex = hash & (newCapacity - 1)`
   - Move node to new array

```java
// Example:
// Initial: capacity=16, threshold=12 (16 × 0.75)
// After 13th put: resize to capacity=32, threshold=24
// After 25th put: resize to capacity=64, threshold=48

// Optimization — provide initial capacity to avoid rehashing:
// If you'll store 1000 entries:
// Minimum capacity = ceil(1000 / 0.75) = 1334 → HashMap rounds up to next power of 2 = 2048
int expectedSize = 1000;
Map<K,V> map = new HashMap<>((int)(expectedSize / 0.75) + 1);  // avoids any resize
```

**Thread safety note:** Rehashing in concurrent scenarios (pre Java 5) caused infinite loops via circular linked list chains. This is why Hashtable exists. In Java 8+, HashMap uses tail insertion (instead of head insertion) which eliminates the circular chain problem, but HashMap is still not thread-safe.

---

## Q26: What is the `entrySet()` iteration pattern and why is it preferred?

**Answer:**

When you need both key and value, use `entrySet()` instead of iterating `keySet()` and doing a `get()`:

```java
// Inefficient — two lookups per entry:
for (String key : map.keySet()) {
    String value = map.get(key);  // second hash lookup
    process(key, value);
}

// Efficient — one lookup via entrySet:
for (Map.Entry<String, String> entry : map.entrySet()) {
    String key = entry.getKey();
    String value = entry.getValue();
    process(key, value);
}

// Java 8+ — forEach with BiConsumer:
map.forEach((key, value) -> process(key, value));
```

---

## Q27: What are the characteristics of LinkedHashSet?

**Answer:**

`LinkedHashSet` maintains insertion order while providing O(1) add/remove/contains. Internally backed by a `LinkedHashMap`.

```java
LinkedHashSet<String> set = new LinkedHashSet<>();
set.add("banana");
set.add("apple");
set.add("cherry");
set.add("apple");  // duplicate — ignored

System.out.println(set);  // [banana, apple, cherry] — insertion order preserved
// vs HashSet: [banana, cherry, apple] or any order
```

**Use case:** Deduplicate while preserving order (e.g., deduplicating a list while keeping first occurrence).

---

## Q28: What is the contract of `Comparable.compareTo()`?

**Answer:**

`compareTo` returns:
- **Negative integer:** this < other
- **Zero:** this == other
- **Positive integer:** this > other

**Required properties:**
1. `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))` (antisymmetry)
2. Transitivity: `x > y && y > z` → `x > z`
3. Consistency: `x.compareTo(y) == 0` → `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`

**Highly recommended:** `(x.compareTo(y) == 0) == x.equals(y)` — inconsistency causes problems in sorted sets/maps.

```java
// Correct integer comparison (avoid subtraction — overflows!):
@Override
public int compareTo(IntWrapper other) {
    return Integer.compare(this.value, other.value);  // correct
    // return this.value - other.value;  // WRONG — can overflow!
}
```

---

## Q29: What is the difference between `Collections.sort()` and `List.sort()`?

**Answer:**

`Collections.sort(list)` internally delegates to `list.sort(null)` since Java 8. They are equivalent.

Both use **TimSort** — a hybrid merge sort + insertion sort. O(n log n) worst case, O(n) for nearly-sorted data (common in practice). Stable sort (equal elements maintain relative order).

```java
List<String> list = new ArrayList<>(Arrays.asList("c", "a", "b"));

// Equivalent:
Collections.sort(list);
list.sort(null);
list.sort(Comparator.naturalOrder());

// With comparator:
list.sort(Comparator.reverseOrder());
list.sort(String.CASE_INSENSITIVE_ORDER);
```

**Arrays.sort()** uses dual-pivot QuickSort for primitives (not stable, better cache performance) and TimSort for objects (stable).

---

## Q30: How do you choose the right collection?

**Answer:**

```
Need a collection of elements?
│
├─ Need key-value mapping? → Use MAP
│   ├─ Fast lookup, no ordering → HashMap
│   ├─ Insertion/access order → LinkedHashMap
│   ├─ Sorted order, range queries → TreeMap
│   ├─ Concurrent access → ConcurrentHashMap
│   └─ Keys are enums → EnumMap
│
├─ Need unique elements? → Use SET
│   ├─ Fast lookup, no ordering → HashSet
│   ├─ Insertion order → LinkedHashSet
│   ├─ Sorted order → TreeSet
│   └─ Concurrent access → ConcurrentHashMap.newKeySet()
│
└─ Allow duplicates, ordered? → Use LIST or QUEUE
    ├─ Random access, rare insert/delete → ArrayList
    ├─ Frequent front insert/delete, queue operations → ArrayDeque
    ├─ Priority-based access → PriorityQueue
    ├─ Thread-safe (write-rare) → CopyOnWriteArrayList
    └─ Concurrent queue → LinkedBlockingQueue / ArrayBlockingQueue
```

---

## Q31: What is the performance impact of autoboxing in collections?

**Answer:**

Standard Java collections use boxed types (`List<Integer>`, `Map<String, Long>`). For each element:
- Autoboxing on add: creates wrapper object on heap
- Unboxing on get: object dereference + primitive extract
- GC pressure: many short-lived or persistent wrapper objects

**For numeric-heavy workloads:**
```java
// Bad for performance — autoboxing:
List<Integer> numbers = new ArrayList<>(1000000);
for (int i = 0; i < 1000000; i++) numbers.add(i);  // 1M Integer objects

// Better — use primitive arrays:
int[] numbers = new int[1000000];

// Or Eclipse Collections primitive lists:
IntList primitiveList = IntLists.mutable.withAll(range);

// Or Stream with primitive operations:
int sum = numbers.stream().mapToInt(Integer::intValue).sum();  // avoids intermediate boxing
int sum = IntStream.range(0, 1000000).sum();  // no boxing at all
```

---

## Q32: What are the characteristics of `TreeMap` regarding `null` keys?

**Answer:**

`TreeMap` does NOT allow null keys because it calls `compareTo()` on keys for ordering, and `null.compareTo()` throws `NullPointerException`.

```java
TreeMap<String, Integer> map = new TreeMap<>();
map.put(null, 1);  // NullPointerException!

// Exception: custom Comparator that handles null (not recommended):
TreeMap<String, Integer> nullableMap = new TreeMap<>(Comparator.nullsFirst(Comparator.naturalOrder()));
nullableMap.put(null, 1);  // OK with this comparator
```

`TreeMap` allows null values — only keys must be non-null.

Contrast with `HashMap` which allows exactly one null key (stored in bucket 0 since `hashCode(null) = 0`).
