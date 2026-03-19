# Collections — Diagram Explanations

---

## HashMap Internal Structure (Java 8+)

```
HashMap<String, Integer>  capacity=16, size=7, loadFactor=0.75
threshold = 16 * 0.75 = 12

table array:
┌───┬──────────────────────────────────────────────────────────┐
│ 0 │ null                                                     │
│ 1 │ → Node["cat"→5] → Node["hat"→2]  (hash collision)      │
│ 2 │ null                                                     │
│ 3 │ → Node["dog"→1]                                         │
│ 4 │ null                                                     │
│...│                                                          │
│ 7 │ → Node["fox"→3]                                         │
│...│                                                          │
│11 │ → Node["pig"→4]  [TreeNode subtree if bucket.size ≥ 8]  │
│...│                                                          │
│15 │ → Node["ant"→6]                                         │
└───┴──────────────────────────────────────────────────────────┘

Node structure:
┌──────────────────────────────────────────┐
│ hash: 3456789                            │
│ key:  "cat"                              │
│ value: 5                                 │
│ next: → Node["hat"→2]                   │
└──────────────────────────────────────────┘
```

---

## Collection Hierarchy (Key Interfaces and Implementations)

```
                     Iterable<T>
                          │
                    Collection<T>
                   /      │      \
              List<T>   Set<T>  Queue<T>
                │         │         │
          ArrayList   HashSet   PriorityQueue
          LinkedList  LinkedHashSet  ArrayDeque
          Vector      TreeSet      LinkedList
                                BlockingQueue<T>
                                   │
                            ArrayBlockingQueue
                            LinkedBlockingQueue

         Map<K,V>  (NOT a Collection!)
            │
      ┌─────┴──────┬────────────┬────────────────┐
  HashMap  LinkedHashMap  TreeMap  ConcurrentHashMap
```

---

## PriorityQueue (Min-Heap) Structure

```
PriorityQueue [1, 3, 2, 7, 4, 5, 6] (min-heap)

         1
        / \
       3   2
      / \ / \
     7  4 5  6

Array representation: [1, 3, 2, 7, 4, 5, 6]
Parent of i: (i-1)/2
Left child of i: 2i+1
Right child of i: 2i+2

poll() returns 1 (root), then re-heapifies:
         2
        / \
       3   5
      / \ /
     7  4 6
```

---

## ConcurrentHashMap (Java 8+) - Bucket-Level Locking

```
ConcurrentHashMap - 16 default buckets

bucket[0]:  [no lock needed for reads]   key1→val1
bucket[1]:  [CAS for empty bucket]       key2→val2
bucket[2]:  [synchronized(bucket head)] key3→val3
            on write to same bucket       key4→val4
...
bucket[15]: ...

Multiple threads can write to DIFFERENT buckets simultaneously
Only threads writing to the SAME bucket contend with each other
Reads are NEVER blocked (volatile reads of bucket array)
```
