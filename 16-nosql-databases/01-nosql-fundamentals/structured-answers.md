# Chapter 01 — NoSQL Fundamentals: Structured Answers

Complete, interview-ready answers with code examples, diagrams references, and precise language.

---

## Q1: Explain the CAP theorem and its practical implications for database design.

**Answer:**

The CAP theorem, proved by Eric Brewer and formalized by Gilbert and Lynch (2002), states that
a distributed data store can guarantee at most **two** of the following three properties
simultaneously when a network partition occurs:

- **Consistency (C)**: Every read receives the most recent write or an error. This is
  *linearisability* — not ACID consistency.
- **Availability (A)**: Every request receives a (non-error) response, though it may not
  contain the most recent write.
- **Partition Tolerance (P)**: The system continues to operate even when network partitions
  (message loss or delay between nodes) occur.

**The key insight**: P is not optional for any distributed system deployed on real hardware.
Network partitions will happen. So the real choice is **C vs A during a partition**:

```
Partition occurs
      │
      ├─► Choose Consistency (CP):
      │   - Refuse writes or return errors on minority partition
      │   - Example: HBase, Zookeeper, etcd, Cassandra with QUORUM
      │
      └─► Choose Availability (AP):
          - Accept writes on both sides, merge conflicts later
          - Example: Cassandra with ONE, Riak, CouchDB, DynamoDB (eventually consistent reads)
```

**PACELC Extension**: PACELC (Daniel Abadi, 2012) adds the dimension that even *without*
a partition, there is a trade-off:
- PA/EL: Partition → Availability, Else → Latency (DynamoDB default, Cassandra with ONE)
- PC/EC: Partition → Consistency, Else → Consistency (etcd, HBase, Spanner)

```python
# Illustrating read-your-writes in application layer when underlying store is AP
import redis
import hashlib

class SessionAwareCache:
    """
    Implements read-your-writes on top of an AP store by routing
    a user's reads to the same replica that received their write,
    using a session token that encodes the replica affinity.
    """
    def __init__(self, primary, replicas):
        self.primary = primary
        self.replicas = replicas

    def write(self, user_id: str, key: str, value: str) -> str:
        # Write to primary, return a session token
        self.primary.set(f"{user_id}:{key}", value)
        # Token encodes "read from primary until this logical time"
        token = self.primary.incr("global_write_counter")
        return str(token)

    def read(self, user_id: str, key: str, session_token: str = None):
        if session_token:
            # Read from primary to guarantee read-your-writes
            return self.primary.get(f"{user_id}:{key}")
        # No token: can read from any replica
        replica = self._pick_replica(user_id)
        return replica.get(f"{user_id}:{key}")

    def _pick_replica(self, user_id: str):
        idx = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % len(self.replicas)
        return self.replicas[idx]
```

---

## Q2: Explain vector clocks and how they track causality in distributed systems.

**Answer:**

A vector clock is a data structure that assigns a logical timestamp to each event in a
distributed system, enabling nodes to determine whether two events are causally related
(one happened-before the other) or concurrent (neither happened-before the other).

**Structure**: A vector clock for a system of N nodes is an array of N integers. Each node
maintains its own vector clock and increments its own position on every local event.

**Rules**:
1. On a local event: `VC[self] += 1`
2. On sending a message: send your current VC with the message
3. On receiving a message: `VC[i] = max(VC_local[i], VC_received[i])` for all i, then
   `VC[self] += 1`

**Comparison**:
- VC_A *dominates* VC_B (A happened-before B) if: all entries of A <= all entries of B
  AND at least one entry of A < the corresponding entry of B
- VC_A and VC_B are *concurrent* if: neither dominates the other

```python
from typing import Dict, List

class VectorClock:
    def __init__(self, node_id: str, all_nodes: List[str]):
        self.node_id = node_id
        self.clock: Dict[str, int] = {n: 0 for n in all_nodes}

    def tick(self):
        """Increment on local event or before sending."""
        self.clock[self.node_id] += 1
        return dict(self.clock)

    def receive(self, remote_clock: Dict[str, int]):
        """Merge on receiving a message, then tick self."""
        for node, ts in remote_clock.items():
            self.clock[node] = max(self.clock.get(node, 0), ts)
        self.clock[self.node_id] += 1

    def dominates(self, other: Dict[str, int]) -> bool:
        """Return True if self happened-before other."""
        return (all(self.clock.get(n, 0) <= other.get(n, 0) for n in other) and
                any(self.clock.get(n, 0) < other.get(n, 0) for n in other))

    def concurrent_with(self, other: Dict[str, int]) -> bool:
        a_dom_b = self.dominates(other)
        b_dom_a = VectorClock._raw_dominates(other, self.clock)
        return not a_dom_b and not b_dom_a

    @staticmethod
    def _raw_dominates(a: Dict, b: Dict) -> bool:
        all_nodes = set(a) | set(b)
        return (all(a.get(n, 0) <= b.get(n, 0) for n in all_nodes) and
                any(a.get(n, 0) < b.get(n, 0) for n in all_nodes))


# Example: Two concurrent writes to the same key
vc_node_a = VectorClock("A", ["A", "B", "C"])
vc_node_b = VectorClock("B", ["A", "B", "C"])

# Node A writes version 1
clock_a_v1 = vc_node_a.tick()  # A:1, B:0, C:0

# Node B writes independently (has not seen A's write)
clock_b_v1 = vc_node_b.tick()  # A:0, B:1, C:0

# These are concurrent — neither dominates
print(vc_node_a.concurrent_with(clock_b_v1))  # True → conflict detected
```

**Key insight for interviews**: When a conflict is detected (two concurrent versions), the
system has three choices:
1. **Last Write Wins**: discard one based on timestamp (risks data loss under clock skew)
2. **Multi-value**: surface both versions to the client (Riak siblings)
3. **CRDT merge**: apply a deterministic merge function (e.g., union for a set)

---

## Q3: Explain the quorum consistency model with concrete math.

**Answer:**

Quorum-based consistency provides a middle ground between strong consistency (all replicas) and
weak consistency (single replica). The key constraint:

```
For strong consistency: W + R > N

Where:
  N = replication factor (total copies of each key)
  W = number of replicas that must acknowledge a write
  R = number of replicas that must respond to a read
```

**Why W + R > N guarantees overlap**: If W replicas hold the latest write and R replicas are
read, then at least one replica in the read set must be in the write set (pigeonhole principle).
That replica will return the latest value, and the coordinator takes the highest-timestamped
response.

**Common configurations**:

| N | W | R | Properties |
|---|---|---|------------|
| 3 | 2 | 2 | Strong consistency, tolerates 1 failure (most common) |
| 3 | 3 | 1 | Strong consistency, max write durability, low read latency |
| 3 | 1 | 3 | Strong consistency, fast writes, slow reads |
| 3 | 1 | 1 | Eventual consistency, maximum performance, lowest durability |
| 5 | 3 | 3 | Strong consistency, tolerates 2 failures |

```python
def quorum_analysis(N: int, W: int, R: int) -> dict:
    """
    Analyse a quorum configuration for consistency and fault tolerance.
    """
    strong_consistent = (W + R) > N
    write_fault_tolerance = N - W  # max nodes that can fail during write
    read_fault_tolerance  = N - R  # max nodes that can fail during read

    # Availability: what fraction of node-failure combinations still work?
    from math import comb
    total_combos = comb(N, max(N - W, N - R))

    return {
        "strong_consistent": strong_consistent,
        "write_fault_tolerance": write_fault_tolerance,
        "read_fault_tolerance": read_fault_tolerance,
        "overlap": W + R - N,  # number of nodes guaranteed to have latest write
    }

# Cassandra LOCAL_QUORUM with RF=3 across 2 DCs (DC1: 2 nodes, DC2: 1 node)
# LOCAL_QUORUM = ceil(RF/2)+1 = 2, R=2, W=2, N=3
print(quorum_analysis(N=3, W=2, R=2))
# {'strong_consistent': True, 'write_fault_tolerance': 1,
#  'read_fault_tolerance': 1, 'overlap': 1}
```

**The repair dimension**: Quorum overlap only guarantees that *one* node in the read set has
the latest value. The coordinator must implement **read repair** to propagate the latest value
to the lagging replicas in the read set. Without repair, a replica can drift arbitrarily far
behind even with quorum reads, because quorum selects the *latest among the R responses*, not
the globally latest.

---

## Q4: Explain CRDTs with a concrete implementation example.

**Answer:**

A CRDT (Conflict-free Replicated Data Type) is a data structure designed so that concurrent
updates on different replicas can always be merged without coordination, without conflicts,
and without data loss. The merge function must be:
- **Commutative**: merge(A, B) = merge(B, A)
- **Associative**: merge(merge(A, B), C) = merge(A, merge(B, C))
- **Idempotent**: merge(A, A) = A (for state-based CRDTs)

**G-Counter (Grow-only Counter)** — simplest CRDT:

```python
from typing import Dict

class GCounter:
    """
    Grow-only counter CRDT.
    Each node tracks only its own increments.
    Global value = sum of all node counts.
    Merge = element-wise maximum.
    """
    def __init__(self, node_id: str, all_nodes: list):
        self.node_id = node_id
        self.counts: Dict[str, int] = {n: 0 for n in all_nodes}

    def increment(self):
        self.counts[self.node_id] += 1

    def value(self) -> int:
        return sum(self.counts.values())

    def merge(self, other: 'GCounter') -> 'GCounter':
        """Merge is element-wise max — always safe to apply."""
        merged = GCounter(self.node_id, list(self.counts.keys()))
        for node in self.counts:
            merged.counts[node] = max(
                self.counts.get(node, 0),
                other.counts.get(node, 0)
            )
        return merged


class PNCounter:
    """
    Positive-Negative Counter: supports both increment and decrement.
    Implemented as two G-Counters: P (positive) and N (negative).
    Value = P.value() - N.value()
    """
    def __init__(self, node_id: str, all_nodes: list):
        self.positive = GCounter(node_id, all_nodes)
        self.negative = GCounter(node_id, all_nodes)

    def increment(self):
        self.positive.increment()

    def decrement(self):
        self.negative.increment()

    def value(self) -> int:
        return self.positive.value() - self.negative.value()

    def merge(self, other: 'PNCounter') -> 'PNCounter':
        merged = PNCounter(self.positive.node_id,
                           list(self.positive.counts.keys()))
        merged.positive = self.positive.merge(other.positive)
        merged.negative = self.negative.merge(other.negative)
        return merged


# Simulation: two nodes independently update a like count
node_a = PNCounter("A", ["A", "B"])
node_b = PNCounter("B", ["A", "B"])

node_a.increment()  # A adds a like
node_a.increment()  # A adds another like
node_b.increment()  # B adds a like (concurrent, no coordination)
node_b.decrement()  # B removes a like (concurrent)

# Merge: deterministic, no conflict
merged = node_a.merge(node_b)
print(merged.value())  # 2: (2+1) - (0+1) = 2, regardless of merge order
```

**OR-Set (Observed-Remove Set)** — supports add and remove without ABA problems:
Each element gets a unique tag on add; remove only removes the specific tagged instance.
Concurrent add and remove of the same element results in the add winning (observed-remove
semantics).

**When CRDTs are NOT appropriate**:
- Invariants that require global knowledge: "account balance >= 0" requires coordination
- Ordered sequences with insertions (Logoot/LSEQ are complex and have tombstone bloat)
- Strong read-your-writes semantics (CRDT merge is background, not synchronous)

---

## Q5: Compare and contrast BASE vs ACID with production examples.

**Answer:**

| Property | ACID | BASE |
|----------|------|------|
| Atomicity | All-or-nothing transaction | Best effort; partial updates may be visible |
| Consistency | DB invariants always hold | "Soft state" — may temporarily violate invariants |
| Isolation | Concurrent txns don't interfere | No isolation guarantee by default |
| Durability | Committed data survives crash | Depends on config (RF, fsync, AOF policy) |
| Availability | Sacrificed during partition | Maintained during partition |
| Scale-out | Expensive (2PC coordination) | Cheap (independent node writes) |

**Production example — bank account (ACID required)**:

```sql
-- PostgreSQL: atomic transfer
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'alice' AND balance >= 100;
UPDATE accounts SET balance = balance + 100 WHERE id = 'bob';
COMMIT;
-- Either both updates happen or neither — atomicity + consistency guaranteed
```

**Production example — social media like count (BASE acceptable)**:

```python
# Redis INCR: atomic at single-node level, but in a cluster with eventual
# consistency, the global count may be temporarily inconsistent across shards.
# Business accepts this: if a post shows 10,001 likes vs 10,003 likes for
# a few seconds, no real harm is done.
import redis
r = redis.Redis()
r.incr("post:42:likes")         # atomic at local shard level
r.expire("post:42:likes", 86400)  # TTL: likes reset daily (soft state)
```

**The nuance**: Modern systems blur the line. DynamoDB Transactions, MongoDB multi-document
transactions, and Cassandra LWT bring ACID-like semantics to traditionally BASE stores. The
choice is not "ACID database vs BASE database" but "which operations in my system need which
guarantees, and what is the cost (latency, throughput, complexity) of providing them?"

---

## Q6: Explain consistent hashing and virtual nodes.

**Answer:**

Consistent hashing maps both data keys and nodes to positions on a logical ring (0 to 2^32-1).
A key is owned by the first node clockwise from the key's hash position.

**Without virtual nodes** — problems:
1. Nodes cluster unevenly on the ring → some nodes own much more data
2. Adding a node only takes data from its clockwise neighbour → rest of cluster unaffected
   (good for minimal data movement, bad for balance)

**With virtual nodes (vnodes)**:
- Each physical node claims V positions (tokens) on the ring (e.g., V=256 in Cassandra)
- Tokens are evenly distributed, so each node owns many small non-contiguous ranges
- Adding a node → steals a small range from many existing nodes simultaneously

```python
import hashlib
import bisect
from collections import defaultdict

class ConsistentHashRing:
    def __init__(self, nodes: list, vnodes: int = 150):
        self.vnodes = vnodes
        self.ring = {}        # hash_position -> node_id
        self.sorted_keys = [] # sorted list of hash positions

        for node in nodes:
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        for i in range(self.vnodes):
            vnode_key = f"{node}:vnode:{i}"
            h = self._hash(vnode_key)
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)

    def remove_node(self, node: str):
        for i in range(self.vnodes):
            vnode_key = f"{node}:vnode:{i}"
            h = self._hash(vnode_key)
            del self.ring[h]
            self.sorted_keys.remove(h)

    def get_node(self, key: str) -> str:
        if not self.ring:
            raise Exception("Empty ring")
        h = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, h)
        if idx == len(self.sorted_keys):
            idx = 0
        return self.ring[self.sorted_keys[idx]]

    def distribution(self, num_keys: int = 100_000) -> dict:
        """Show how evenly keys are distributed across nodes."""
        counts = defaultdict(int)
        for i in range(num_keys):
            node = self.get_node(f"key:{i}")
            counts[node] += 1
        return dict(counts)


ring = ConsistentHashRing(["node-1", "node-2", "node-3"], vnodes=150)
print(ring.distribution())
# Typical output: {'node-1': 33412, 'node-2': 33291, 'node-3': 33297}
# Very even distribution with 150 vnodes per node

print(ring.get_node("user:alice"))  # deterministic → 'node-2'
```

**Interview follow-up**: When you add node-4 with vnodes=150, approximately 25% of keys move
to the new node — but those keys come from *all three* existing nodes, not just one. This is
better for balance but means all three existing nodes participate in data transfer during
bootstrap, which requires throttling to prevent performance degradation.
