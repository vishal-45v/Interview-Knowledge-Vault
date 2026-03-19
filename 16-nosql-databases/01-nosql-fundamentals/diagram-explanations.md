# Chapter 01 — NoSQL Fundamentals: Diagram Explanations

ASCII diagrams illustrating core distributed systems concepts. Sketch these on a whiteboard
during interviews to demonstrate architectural depth.

---

## Diagram 1: CAP Theorem — The Partition Decision Tree

```
                    NETWORK PARTITION OCCURS
                            │
                            ▼
              ┌─────────────────────────────┐
              │   Does the minority side    │
              │   still accept writes?      │
              └─────────────────────────────┘
                    │                 │
                   YES               NO
                    │                 │
                    ▼                 ▼
          ┌──────────────┐   ┌──────────────────┐
          │  AP SYSTEM   │   │   CP SYSTEM      │
          │              │   │                  │
          │ Availability │   │  Consistency     │
          │ preserved    │   │  preserved       │
          │              │   │                  │
          │ Risk: writes │   │  Risk: writes    │
          │ diverge on   │   │  fail / block    │
          │ both sides   │   │  on minority     │
          │              │   │  partition side  │
          └──────────────┘   └──────────────────┘
          Examples:          Examples:
          - Cassandra (ONE)  - Cassandra (QUORUM)
          - DynamoDB default - HBase
          - CouchDB          - etcd / ZooKeeper
          - Riak             - Google Spanner

  WITHOUT PARTITION (normal operation):
  ┌────────────────────────────────────────┐
  │  PACELC: Latency vs Consistency        │
  │                                        │
  │  Async replication → Low latency       │
  │  but stale reads possible              │
  │                                        │
  │  Sync replication  → Higher latency    │
  │  but always fresh reads                │
  └────────────────────────────────────────┘
```

**Explanation:** The decision tree shows that P (partition tolerance) is not a choice —
it is a reality. The only decision point is what to sacrifice *during* a partition.
The PACELC rectangle at the bottom shows the additional trade-off that exists even on
a healthy network: synchronous replication ensures consistency at the cost of latency.

---

## Diagram 2: Quorum Intersection Guarantee

```
  Replication Factor N = 5
  Write Quorum W = 3  (nodes A, B, C receive the write)
  Read Quorum  R = 3  (nodes B, C, D are queried)

  ┌───────────────────────────────────────────────────────┐
  │                                                       │
  │   WRITE SET         INTERSECTION     READ SET         │
  │                                                       │
  │  ┌───┐                               ┌───┐           │
  │  │ A │                               │ D │           │
  │  └───┘                               └───┘           │
  │                  ┌───┐ ┌───┐                         │
  │                  │ B │ │ C │   ← Overlap: B and C    │
  │                  └───┘ └───┘     have latest write   │
  │                                                       │
  │  W + R = 6 > N = 5  →  Overlap = 6 - 5 = 1 node     │
  │  (actually 2 nodes overlap here: B and C)            │
  │                                                       │
  └───────────────────────────────────────────────────────┘

  Coordinator picks the HIGHEST TIMESTAMP from the read set
  and returns it as the authoritative value.

  Read repair: B and C have latest value. D gets repair write.

  ┌─────────────────────────────────────────────────────────┐
  │  W=1, R=1, N=3  →  W+R=2, NOT > 3  →  No overlap      │
  │  guaranteed  →  Stale read possible                     │
  │                                                         │
  │  WRITE: ──► Node A (value=42, ts=100)                   │
  │  READ:  ──► Node C (value=41, ts=99)  ← STALE!         │
  └─────────────────────────────────────────────────────────┘
```

**Explanation:** The diagram shows why W + R > N is the key formula. The write set and read set
are guaranteed to overlap by at least one node when this condition holds. That overlapping node
will always have the latest value, and the coordinator's timestamp comparison will pick it.

---

## Diagram 3: Vector Clock Causality and Concurrency

```
  Three nodes: A, B, C.  Format: [A, B, C]

  Time ──────────────────────────────────────────────────►

  Node A:  [1,0,0]──────────────────►[2,1,0]──►[3,1,0]
              │                          │
              │ A sends msg to B         │ A sends msg to C
              ▼                          ▼
  Node B:  [0,0,0]──►[1,1,0]──►[1,2,0]  [2,1,1]──►[2,1,2]
                                  │
                                  │ B sends msg to C (concurrent)
                                  ▼
  Node C:  [0,0,0]──────────────►[1,2,1]

  CAUSALITY EXAMPLES:
  ┌──────────────────────────────────────────────────────┐
  │  Event [1,0,0] HAPPENED-BEFORE [2,1,0]              │
  │  because [1,0,0] ≤ [2,1,0] component-wise           │
  │  (1≤2, 0≤1, 0≤0) AND strictly less in some ✓        │
  │                                                      │
  │  Event [1,2,0] and [2,0,1] are CONCURRENT           │
  │  because [1,2,0] vs [2,0,1]:                        │
  │    1 < 2 (A component: first < second)              │
  │    2 > 0 (B component: first > second)              │
  │  Neither dominates → CONFLICT must be resolved      │
  └──────────────────────────────────────────────────────┘

  CONFLICT RESOLUTION OPTIONS:
  ┌──────────────────────────────────────────────────────┐
  │  1. LWW (Last Write Wins):  pick higher wall-clock  │
  │     Risk: clock skew can discard newer write        │
  │                                                      │
  │  2. Multi-value (Siblings): return both to client   │
  │     Client must resolve (e.g., Riak siblings)       │
  │                                                      │
  │  3. CRDT merge: apply deterministic merge function  │
  │     No loss, no client burden, but limited to       │
  │     CRDT-compatible data types                      │
  └──────────────────────────────────────────────────────┘
```

**Explanation:** The diagram traces vector clock evolution across three nodes. Reading down
any column shows a node's clock progressing. Reading diagonally (following message arrows)
shows how events at one node become causally visible at another. When no such path exists
between two events, they are concurrent.

---

## Diagram 4: Consistent Hashing Ring with Virtual Nodes

```
  Physical nodes: P1, P2, P3    Vnodes per node: 3

  Ring (0 ──────────────────────────────── 2^32-1, wraps)

         P2v1          P3v2           P1v2
          │              │              │
  ────────●──────────────●──────────────●────────────
  │                                               wrap │
  ●                                               ●    │
  P1v3                                         P3v1   │
  │                                               │    │
  ──────●──────────────────────────●──────────────    │
        P2v3                     P2v2                  │
                     ●                                 │
                    P3v3                               │
                     │                                 │
  ─────────────────────────────────────────────────────
                     │
                    P1v1

  DATA KEY "user:alice" hashes to position X on ring.
  Owner = first vnode CLOCKWISE from X.

  ┌────────────────────────────────────────────────────┐
  │  Replication (RF=2):                               │
  │  Primary owner   = vnode at X + ε                 │
  │  First replica   = next distinct PHYSICAL node     │
  │                    clockwise (skip same-node vnodes)│
  └────────────────────────────────────────────────────┘

  ADD P4 with 3 vnodes:
  ┌────────────────────────────────────────────────────┐
  │  P4v1 placed at position → steals range from P3   │
  │  P4v2 placed at position → steals range from P1   │
  │  P4v3 placed at position → steals range from P2   │
  │                                                    │
  │  ~25% of data moves. Data comes from ALL 3 nodes. │
  │  All 3 nodes participate in streaming to P4.      │
  └────────────────────────────────────────────────────┘
```

**Explanation:** Each physical node owns multiple non-contiguous arcs of the ring. This
distributes load evenly. When a new node joins, it inserts its vnodes throughout the ring and
claims small segments from many existing nodes, preventing a single node from bearing all the
migration load.

---

## Diagram 5: Paxos Two-Phase Consensus

```
  Roles: PROPOSER (P), ACCEPTORS (A1, A2, A3), LEARNER (L)
  Quorum = majority = 2 of 3 acceptors

  PHASE 1: PREPARE
  ─────────────────
  P ──── Prepare(n=5) ─────► A1  responds: Promise(n=5, accepted_value=null)
  P ──── Prepare(n=5) ─────► A2  responds: Promise(n=5, accepted_value=null)
  P ──── Prepare(n=5) ─────► A3  (no response / slower)

  P received promises from A1, A2 (quorum met)

  PHASE 2: ACCEPT
  ─────────────────
  P ──── Accept(n=5, value="commit") ──► A1  responds: Accepted(n=5)
  P ──── Accept(n=5, value="commit") ──► A2  responds: Accepted(n=5)
  P ──── Accept(n=5, value="commit") ──► A3  (no response)

  Quorum of accepts received → value is CHOSEN

  P ──── Notify(value="commit") ─────► L   (learner records decision)

  ┌──────────────────────────────────────────────────────┐
  │  SAFETY GUARANTEE: No two Proposers can choose       │
  │  different values for the same round number n.       │
  │  Any higher-numbered proposal must use the value     │
  │  already accepted by the highest prior proposal      │
  │  seen in Phase 1 responses.                          │
  │                                                      │
  │  LIVENESS: Not guaranteed — two proposers can        │
  │  indefinitely pre-empt each other (duelling          │
  │  proposers). Multi-Paxos uses a stable leader        │
  │  to avoid this.                                      │
  └──────────────────────────────────────────────────────┘
```

**Explanation:** Paxos is a two-round protocol. Phase 1 (Prepare/Promise) establishes that no
other proposer with a higher round number will override this proposer's value. Phase 2
(Accept/Accepted) commits the value. The key safety property is that once a value is chosen
(accepted by a quorum), any future proposal for any higher n will also choose the same value.

---

## Diagram 6: Eventual Consistency — Replica Convergence Timeline

```
  Three replicas: R1, R2, R3

  t=0ms:  Client writes key="X" value=42 to R1
  t=5ms:  R1 replicates to R2 (async)
  t=10ms: R1 replicates to R3 (async, slower link)

  ┌─────┬──────────────────────────────────────────────────┐
  │Time │ R1          R2          R3       Client Read     │
  ├─────┼──────────────────────────────────────────────────┤
  │ t=0 │ X=42        X=null      X=null                   │
  │ t=3 │ X=42        X=null      X=null   reads R2 → null │
  │ t=5 │ X=42        X=42        X=null   reads R3 → null │
  │ t=7 │ X=42        X=42        X=null   reads R1 → 42   │
  │t=10 │ X=42        X=42        X=42     reads R3 → 42   │
  └─────┴──────────────────────────────────────────────────┘

  CONSISTENCY MODELS LAYERED ON TOP:

  ┌────────────────────────────────────────────────────────┐
  │  Eventual:      No guarantee on when client sees 42   │
  │                                                        │
  │  Read-Your-Writes: After writing to R1, client MUST   │
  │  read from R1 (or wait for R2/R3) until t≥10ms        │
  │                                                        │
  │  Monotonic Reads: Once a client sees X=42, they       │
  │  must never see X=null again (no backward time travel) │
  │                                                        │
  │  Causal:        If client sees X=42 and then writes   │
  │  Y=100 (causally dependent), any reader who sees      │
  │  Y=100 must also see X=42                             │
  └────────────────────────────────────────────────────────┘
```

**Explanation:** The timeline shows the progression of replica synchronisation after a single
write. Eventual consistency guarantees convergence (all replicas reach X=42) but makes no
promise about *when*. The four consistency models layered on top are progressively stronger
guarantees that restrict which staleness windows are observable by clients.
