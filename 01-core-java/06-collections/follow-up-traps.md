# Collections — Follow-Up Trap Questions

## Trap 1: "Can you put null in a TreeSet?"
**Answer:** No. TreeSet uses `compareTo()` for ordering. `null` has no `compareTo()` method → NullPointerException. HashSet allows exactly one null. LinkedHashSet allows one null. TreeSet never allows null keys.

## Trap 2: "What is the difference between List.of() and Arrays.asList() when modifying?"
**Answer:** Both throw UnsupportedOperationException on add/remove. But `Arrays.asList()` ALLOWS `set()` (update in place) because it's backed by the original array. `List.of()` throws on any structural or non-structural modification including `set()`. Always use `List.of()` when you want truly immutable lists.

## Trap 3: "Why does Collections.sort() require Comparable but TreeMap doesn't?"
**Answer:** TreeMap requires either a Comparator provided at construction OR that keys implement Comparable. The error manifests at runtime when you insert a non-Comparable key without a Comparator — ClassCastException. Collections.sort() fails at compile time if the list type doesn't implement Comparable.

## Trap 4: "Is ConcurrentHashMap's `size()` accurate?"
**Answer:** Not guaranteed to be exact in the presence of concurrent modifications. `size()` may return an estimate. Use `mappingCount()` for better accuracy with large maps. For exact concurrent counting, use `LongAdder` or `AtomicLong` separately.

## Trap 5: "What happens if hashCode() always returns the same value?"
**Answer:** All entries go to the same bucket. Effectively turns the HashMap into a linked list (or red-black tree in Java 8+). Lookup degrades from O(1) to O(n) or O(log n). This is a hash flooding attack vector — malicious input crafted to make all keys collide.

## Trap 6: "What is the modCount in ArrayList used for?"
**Answer:** `modCount` (inherited from AbstractList) is incremented on every structural modification (add, remove, set). The iterator captures `modCount` at creation time. On each `next()`, it checks current `modCount` against the saved value — if different, throws ConcurrentModificationException. This is why you can't modify a list while iterating with a for-each loop.

## Trap 7: "Can you have a HashMap with Integer keys -128 to 127 and have == work correctly?"
**Answer:** Only because Integer.valueOf() caches these. But this is an implementation detail you should never rely on. Always use `equals()` for key comparison. The HashMap itself uses `equals()` internally — it would work regardless.

## Trap 8: "What is the difference between Queue.offer() and Queue.add()?"
**Answer:** Both add to a Queue. `add()` throws IllegalStateException if capacity is exceeded (bounded queues). `offer()` returns false instead of throwing. For production code, `offer()` is safer. Similarly: `poll()` returns null when empty, `remove()` throws NoSuchElementException.
