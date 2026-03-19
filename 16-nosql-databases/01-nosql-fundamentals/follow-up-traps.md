# Chapter 01 — NoSQL Fundamentals: Follow-up Traps

These are the exact follow-up questions that interviewers use to separate candidates who have
memorised definitions from those who truly understand the trade-offs.

---

## Trap 1: "You said CAP says pick 2 of 3 — so can I build a CA system?"

**What most people say:**
"Yes, a CA system is one that is consistent and available but not partition-tolerant, like a
single-node database."

**Correct answer:**
No distributed system can be CA in any meaningful sense. Network partitions are not optional —
they happen in every real network (cable failures, packet drops, ARP timeouts). "CA" in CAP means
"behaves as CA when there is no partition," which is trivially true of all databases. The real
choice CAP describes is: *when a partition occurs*, do you maintain consistency by refusing writes
(CP) or maintain availability by accepting writes that may become inconsistent (AP)?

A single-node database is simply not subject to the CAP theorem because the theorem applies to
distributed systems with network partitions between nodes. Eric Brewer himself has said the CA
category is largely theoretical.

---

## Trap 2: "PACELC says there's a latency/consistency trade-off even without partitions — doesn't that mean we should just always pick low latency?"

**What most people say:**
"Yes, you pick low latency for the fast path because partitions are rare."

**Correct answer:**
This conflates two separate decisions. The ELC part of PACELC says: even without a partition,
synchronous replication (for consistency) adds latency vs asynchronous replication (for lower
latency). Choosing low latency means you accept the risk of reading stale data and the risk of
losing committed writes if a replica that hasn't received the async update becomes primary.

The right answer depends on the business domain. A financial ledger must trade latency for
consistency. A social media like-count can trade consistency for latency. The point of PACELC is
that this is a *continuous* trade-off, not just a failure-mode trade-off.

---

## Trap 3: "You said W + R > N guarantees strong consistency — what if a node crashes immediately after a successful write quorum?"

**What most people say:**
"That's fine because the write was already acknowledged by W nodes."

**Correct answer:**
If a node that participated in the write quorum crashes *before* the read quorum is assembled,
and the remaining alive nodes that constitute the new read quorum did not receive the write,
the read can miss the latest value. This is the "quorum overlap" guarantee breaking under churn.

For example: N=3, W=2, R=2 (W+R=4 > 3). Nodes A, B, C. Write goes to A and B (quorum met).
B then crashes. Read quorum: A and C. A has the new value, C does not. The read still returns
the new value because A participates. But if A *also* crashes before the read, only C remains,
and we have lost quorum entirely (not a stale read problem — an availability problem). The real
risk is a write that succeeds on W nodes but those nodes all fail before repair propagates:
the write is durably lost. This is why `replication_factor` and repair matter as much as quorum.

---

## Trap 4: "Eventual consistency means the system will eventually converge — how long is eventually?"

**What most people say:**
"It's undefined — just eventually."

**Correct answer:**
"Eventually" is bounded in practice, but the bound depends on anti-entropy / repair mechanisms,
replication topology, and network conditions. In a well-configured Cassandra cluster, replica
divergence under normal operation is typically milliseconds to seconds. Under a node failure, it
can extend until `hinted_handoff_throttle_in_kb` delivers hints (up to 3 hours by default window)
or until manual repair runs.

The dangerous follow-up is: what if repair never runs? Some systems provide no automatic
background convergence beyond hinted handoff (which has an expiry). Without regular `nodetool
repair`, a Cassandra node that was down longer than `max_hint_window_in_ms` will permanently
diverge — "eventually" becomes "never" until you manually trigger repair. This is why "eventual
consistency" in production requires an operational contract, not just a theoretical guarantee.

---

## Trap 5: "You mentioned CRDTs — can you replace all of your distributed database's conflict resolution with CRDTs?"

**What most people say:**
"Yes, CRDTs are the ideal solution for all concurrent update conflicts."

**Correct answer:**
CRDTs are only suitable for data types where the merge operation is commutative, associative, and
idempotent — and where the semantics of the merge align with the business semantics. A G-Counter
(grow-only counter) is perfect for a like count. But a shopping cart that needs to support remove
is more complex (requires a 2P-Set or OR-Set). An account balance with a "cannot go negative"
invariant cannot be expressed as a CRDT without coordination, because the invariant requires
global knowledge.

CRDTs also have space costs: OR-Sets carry unique tags per element per add operation, causing
bloat. And CRDTs that merge diverged state (state-based) require shipping entire state per gossip
round, which is expensive for large structures. They are a powerful tool but not a universal one.

---

## Trap 6: "Vector clocks solve the causality tracking problem — why doesn't every database use them?"

**What most people say:**
"They should — vector clocks are the right solution."

**Correct answer:**
Vector clocks have two practical problems at scale:

1. **Size**: A vector clock has one entry per node (or per client in client-stamped vector clocks).
   In a cluster with hundreds of nodes, or a system with millions of clients, the clock becomes
   megabytes per object.

2. **Client complexity**: With server-side vector clocks, the client must pass the causal context
   token back on every write. Stateless clients or clients behind load balancers often fail to
   do this, causing false concurrency detection.

Dynamo-style systems (and Riak) use vector clocks but limit their size via pruning (which
introduces false-concurrency risk). Amazon's internal systems moved toward *causal tokens* /
dotted version vectors, which are more compact. DynamoDB uses *conditional writes* (optimistic
locking) rather than vector clocks, offloading conflict detection to the application layer.

---

## Trap 7: "You mentioned Paxos — is Raft just Paxos with a different name?"

**What most people say:**
"Yes, they're equivalent — Raft is just a more understandable version of Paxos."

**Correct answer:**
They are both consensus algorithms that solve the same safety problem, but they make different
algorithmic choices. Key differences:

- Paxos is leader-optional (Multi-Paxos uses a stable leader as an optimisation, but single-decree
  Paxos has no concept of a persistent leader). Raft mandates a strong leader for *all* decisions.
- Raft restricts log replication: only the leader can send log entries, and a candidate cannot win
  an election if its log is less up-to-date than a majority. This simplifies reasoning about
  log completeness.
- Raft handles cluster membership changes via joint consensus or single-server changes. Paxos
  leaves membership change as an open problem (Multi-Paxos papers vary on this).
- In practice, "Paxos" in production systems is usually a bespoke variant (Google Chubby, Zab in
  ZooKeeper) that differs significantly from the original paper.

Raft is closer to Multi-Paxos with explicit leader leases and a log-matching safety property.
They are not identical.

---

## Trap 8: "Read repair fixes consistency — so I don't need to run manual repair in Cassandra, right?"

**What most people say:**
"Correct — read repair keeps replicas in sync automatically."

**Correct answer:**
Read repair only fixes replicas for *read* keys. Data that is written and never read accumulates
replica divergence indefinitely. In a time-series or log system where data is written once and
queried rarely (or only during incidents), read repair provides essentially zero coverage.

Hinted handoff handles the short-term case (node temporarily down), but once the hint window
expires, divergence is permanent until repair. `nodetool repair` uses Merkle tree comparison to
find *all* divergent ranges, not just recently-read keys. Without periodic full-cluster repair,
a Cassandra cluster will slowly accumulate "entropy" — data present on some replicas but not
others — which becomes visible as inconsistency during node replacements, bootstraps, or when
a new node joins and receives an old snapshot.

---

## Trap 9: "BASE is the opposite of ACID — so BASE systems can't do transactions?"

**What most people say:**
"That's right — NoSQL BASE systems don't support transactions."

**Correct answer:**
This is a false dichotomy. Many NoSQL systems that operate as AP / BASE at large scale offer
*limited* or *scoped* transactions:
- Cassandra offers Lightweight Transactions (LWT) using Paxos for single-partition CAS operations
- MongoDB offers multi-document ACID transactions (since v4.0)
- DynamoDB offers `TransactWriteItems` across up to 25 items
- Google Spanner offers full serialisable distributed transactions (it is CP + transactions)

The spectrum is not ACID ↔ BASE binary. It is: "what scope of atomicity and isolation can the
system guarantee, and at what latency/availability cost?" A system can be BASE for its primary
workload (eventual consistency for reads) while offering ACID-like semantics for specific
critical paths (e.g., inventory decrement, seat reservation). The skill is knowing which
operations need which guarantee.

---

## Trap 10: "You said consistent hashing distributes load evenly — doesn't adding a node always cause exactly 1/N of the data to move?"

**What most people say:**
"Yes, consistent hashing only moves K/N keys when a new node is added."

**Correct answer:**
With a *single token per node* (original consistent hashing), adding a new node only takes over
the key range immediately before it on the ring, which is 1/N of the data *only if nodes are
evenly spaced*. In practice, nodes cluster unevenly on the ring, causing some nodes to own much
larger ranges than others — which is precisely the hot-spot problem.

Virtual nodes (vnodes) solve this by assigning each physical node many small, non-contiguous
token ranges across the ring. With 256 vnodes per physical node, data is distributed much more
evenly. However, when a node is added with vnodes, it now steals small ranges from *many* nodes
simultaneously, causing many nodes to transfer data concurrently. This can cause a brief I/O
spike across the entire cluster during bootstrapping. The trade-off is better balance at the cost
of more complex rebalancing I/O patterns.

---

## Trap 11: "Linearisability and serializability sound the same — are they?"

**What most people say:**
"Yes, they're basically the same consistency level."

**Correct answer:**
They are different and often confused:

- **Linearisability** is a *single-object* consistency model. It says that every read/write to a
  single key appears to happen atomically at a single point in real time between the operation's
  start and end. It is about the ordering of operations on *one* object.

- **Serializability** is a *multi-object* (transaction) isolation level. It says that concurrent
  transactions can be re-ordered such that the result is equivalent to some serial (one-at-a-time)
  execution. It says nothing about real-time ordering.

**Strict serializability** = serializability + linearisability: transactions are both isolated AND
happen in real-time order. This is the strongest guarantee and what systems like Google Spanner
provide. PostgreSQL's SERIALIZABLE isolation level provides serializability but NOT
linearisability (a transaction can read a snapshot from the past). Cassandra with `ALL`
consistency provides linearisability on single-partition reads/writes but not serializability
across partitions.

---

## Trap 12: "Hinted handoff means I can set consistency level ANY and still have durable writes?"

**What most people say:**
"Yes, ANY means at least one node receives the write, which is durable."

**Correct answer:**
`ANY` in Cassandra means the coordinator itself will store a hint if ALL replicas are down,
and the write is considered "successful." This is *not* the same as durable: a hint is stored
only on the coordinator node. If the coordinator crashes before delivering the hint, the write
is permanently lost. Hints are also not searchable — a read with `QUORUM` will not find the
data stored only as a hint on the coordinator. `ANY` is appropriate only for "fire and forget"
metrics or events where loss is acceptable and you need writes to succeed even during total
replica failure. Using `ANY` for user data in a production system is almost always a mistake.
