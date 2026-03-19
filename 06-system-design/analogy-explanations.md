# System Design — Analogy Explanations

---

## Load Balancer as a Restaurant Host

A load balancer is like a restaurant host who greets customers and assigns them to available tables (servers). The host ensures no single waiter is overwhelmed while others are idle. If a waiter calls in sick (server down), the host stops sending customers to that station.

---

## Cache as a Whiteboard

A cache is like a whiteboard vs a filing cabinet. The filing cabinet (database) has all the information but takes time to search. A whiteboard (cache) holds the most frequently needed information, visible instantly. When the whiteboard is full, you erase the oldest or least-used notes (LRU eviction).

---

## Message Queue as an Airport Check-In Counter

A message queue is like baggage handling at an airport. Passengers (producers) drop off their bags at check-in (enqueue). Baggage handlers (consumers) process bags at their own pace. If there's a surge of passengers, bags wait in queue rather than the conveyor belt crashing.

---

## CAP Theorem as a Business Decision

Imagine a distributed coffee chain. CAP theorem is like choosing between:
- Consistency: Every barista sees the same menu (including real-time price changes) — requires network communication before every order
- Availability: Every barista can take orders even if HQ is unreachable — may charge wrong prices briefly
- Partition Tolerance: Store can operate even if communication between locations is down

During a network outage (partition), you must choose: refuse orders (CP) or continue with potentially stale prices (AP).

---

## Microservices as Departments in a Company

A monolith is like a single team doing everything — fast to start, but eventually everyone's in each other's way. Microservices are like separate departments — HR, Finance, Engineering — each with their own responsibilities and processes. Coordination (API calls) is needed between departments, but each can grow independently.

---

## Consistent Hashing as a Ring-Shaped Seating Chart

Traditional hashing is like assigning people to seats 1 to N. Add a new seat → everyone's assignment changes. Consistent hashing is like a round table. Each person (key) sits at their nearest named chair (server) clockwise. Add a new chair → only the person between the old chairs moves. Everyone else stays put.
