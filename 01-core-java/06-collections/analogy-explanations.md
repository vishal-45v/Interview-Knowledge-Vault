# Collections — Analogy Explanations

---

## HashMap — The Locker Room Analogy

Imagine a locker room with 16 numbered lockers (initial capacity). When you want to store your bag (value) under your name (key), the attendant runs a formula on your name to get a locker number (hashCode % capacity). If two people get the same locker number (hash collision), they share the locker with a chain system (linked list or tree). The attendant always finds your bag by running the same formula on your name — O(1) lookup.

## LinkedHashMap — The Locker Room with a Sign-In Sheet

Same locker room, but the attendant also maintains a sign-in sheet recording the order each item was stored. When you want items in order, consult the sheet. Still O(1) lookup, but adds ordering.

## TreeMap — The Filing Cabinet

A filing cabinet with alphabetically ordered folders. Finding a file requires flipping to the right section (O(log n) like binary search). But files are always in sorted order, and you can easily find all files from A to M (range queries).

## HashSet — The Stamp Collection Analogy

A collection where each stamp is unique. Before adding a new stamp, you check if you already have it (hashCode to find shelf, equals to confirm). If you already have it, you don't add another. It's just a HashMap where you only care about the key (the stamp) not the value.

## PriorityQueue — The Hospital Triage Analogy

Patients arrive in any order, but the most critical cases are always seen first. When you add a patient (offer), they're placed in the queue based on urgency (compareTo). When the doctor is free (poll), they always take the most urgent case — not necessarily the first to arrive.

## ArrayList vs LinkedList — The Bookshelf vs Chain of Post-Its

**ArrayList** is a bookshelf — all books in a row. Finding book #50 is instant (jump straight to position 50). Adding a book in the middle requires sliding all others over (expensive).

**LinkedList** is a chain of Post-Its where each note says "next note is on the left wall." Finding note #50 requires following the chain from the beginning (slow). But inserting a new note just means relinking two adjacent notes (fast if you're already at that position).

## ConcurrentHashMap — The Bank with Multiple Tellers

A regular HashMap is a bank with one teller and one long queue. ConcurrentHashMap is a bank with multiple tellers (segments/buckets). Customers waiting for different tellers don't block each other. Only customers at the same teller might briefly wait. Much higher throughput under concurrent access.

## BlockingQueue — The Vending Machine Analogy

A vending machine with N slots (bounded queue). The restocking person (producer) can only add items when there are empty slots — waits if full. Customers (consumers) can only take items when there's stock — wait if empty. Neither side needs to poll; they just block until their action is possible.
