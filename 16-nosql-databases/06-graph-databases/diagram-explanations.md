# Chapter 06: Graph Databases — Diagram Explanations

## Diagram 1: Neo4j Native Graph Storage — Physical Record Layout

```
NODE STORE (neostore.nodestore.db) — Fixed 15 bytes per record
═══════════════════════════════════════════════════════════════════════════

Node Record #5 (byte offset: 5 × 15 = 75):
┌─────────┬─────────────────┬─────────────────┬───────────────┬──────────┐
│ inUse   │ firstRelId      │ firstPropId     │ labelId       │ extra    │
│ (1 byte)│ (4 bytes)       │ (4 bytes)       │ (4 bytes)     │ (2 bytes)│
│   0x01  │ → Rel Record 12 │ → Prop Record 8 │ → Label "User"│          │
└─────────┴─────────────────┴─────────────────┴───────────────┴──────────┘
                  │                   │
                  ▼                   ▼
RELATIONSHIP STORE               PROPERTY STORE
(34 bytes per record)            (41 bytes per record)

Rel Record #12:                  Prop Record #8:
┌──────────┬──────────┬─────┐   ┌──────────┬──────────┬──────────────────┐
│ firstNode│ secondNode│type│   │ propType │ propKey  │ propValue        │
│ → Node 5 │ → Node 9  │ 3  │   │ STRING   │ "name"   │ "Alice"          │
│ prevRel1 │ prevRel2  │    │   │ nextProp │ → Prop 9 │                  │
│ nextRel1 │ nextRel2  │    │   └──────────┴──────────┴──────────────────┘
└──────────┴──────────┴─────┘

KEY INSIGHT: Node #5 is always at byte 75. No index lookup needed.
Following the firstRelId pointer is a direct file seek. O(1) per hop.
```

**Explanation:**
Fixed-size records are the foundation of index-free adjacency. Because every node record
is exactly 15 bytes, computing the disk offset of any node is arithmetic, not a search.
The relationship doubly-linked list structure (prevRel/nextRel for both source and target)
allows traversal in both directions along the relationship chain without additional lookups.
This contrasts with a B-tree index (SQL's JOIN mechanism), which requires O(log n) key
comparisons to locate a row.

---

## Diagram 2: Property Graph Model — Social Network Example

```
PROPERTY GRAPH: Social Network Fragment
═══════════════════════════════════════════════════════════════════════════

         Labels: [Person, Employee]               Labels: [Person]
         ┌──────────────────────────┐              ┌─────────────────┐
         │  :Person:Employee        │              │  :Person        │
         │  id: "P001"              │              │  id: "P002"     │
         │  name: "Alice Chen"      │              │  name: "Bob K." │
         │  age: 32                 │              │  age: 28        │
         │  department: "Eng"       │              │                 │
         └──────────────────────────┘              └─────────────────┘
                    │    ▲                                  │
     [:WORKS_AT]    │    │ [:MANAGES]          [:FRIENDS_WITH {since: 2019}]
     {since: 2020,  │    │ {reports_to: true}               │
      role: "SWE"}  │    │                                  ▼
                    ▼    │                       ┌──────────────────────┐
         ┌──────────────────────────┐            │  :Person             │
         │  :Company                │            │  id: "P003"          │
         │  id: "C001"              │            │  name: "Carol M."    │
         │  name: "TechCorp"        │            │  age: 35             │
         │  founded: 2010           │            └──────────────────────┘
         │  employees: 5000         │
         └──────────────────────────┘
                    │
         [:LOCATED_IN {hq: true}]
                    │
                    ▼
         ┌──────────────────────────┐
         │  :City                   │
         │  id: "CI001"             │
         │  name: "San Francisco"   │
         │  state: "CA"             │
         └──────────────────────────┘

FOUR PRIMITIVES:
┌───────────────┬─────────────────────────────────────────────────────────┐
│ Nodes         │ Entities with labels and properties                     │
│ Relationships │ Typed, directed connections with properties             │
│ Properties    │ Key-value data on nodes and relationships               │
│ Labels        │ Categories/types for nodes (node can have multiple)     │
└───────────────┴─────────────────────────────────────────────────────────┘
```

**Explanation:**
This diagram shows all four property graph primitives in action. Alice has two labels
(Person AND Employee) — multi-labeling allows a node to participate in multiple semantic
contexts. The FRIENDS_WITH relationship carries its own property (since: 2019),
distinguishing it from a simple pointer. The WORKS_AT relationship stores employment
metadata (role, start date) directly on the edge rather than requiring a separate
"Employment" join table as a relational schema would. This is the key advantage of
property graphs over RDF triples for application-centric data modeling.

---

## Diagram 3: Neo4j Causal Clustering — Write and Read Routing

```
CAUSAL CLUSTER TOPOLOGY
═══════════════════════════════════════════════════════════════════════════

Application                    Cluster Core                  Read Scale
    │                               │
    │  WRITE (any core endpoint)    │
    ├──────────────────────────────►│
    │                          ┌────┴────┐
    │                          │ LEADER  │◄─── Raft consensus
    │                          │ Core #1 │     (commit requires
    │                          └────┬────┘      majority ACK)
    │                               │
    │              ┌────────────────┤────────────────┐
    │              ▼                ▼                │
    │         ┌─────────┐     ┌─────────┐           │
    │         │Follower │     │Follower │           │
    │         │ Core #2 │     │ Core #3 │           │
    │         └────┬────┘     └────┬────┘           │
    │              │               │                 │
    │    ┌─────────┘               └──────────┐     │
    │    │  Async replication (catch-up)       │     │
    │    ▼                                    ▼     │
    │ ┌────────┐                         ┌────────┐ │
    │ │  Read  │                         │  Read  │ │
    │ │Replica │                         │Replica │ │
    │ │  RR-1  │                         │  RR-2  │ │
    │ └────────┘                         └────────┘ │
    │                                               │
    │  READ (load balanced across RRs + Followers)  │
    └───────────────────────────────────────────────┘

CAUSAL BOOKMARK FLOW:
┌─────────────┐    ① Write Tx      ┌─────────────┐
│    Client   │───────────────────►│   Leader    │
│             │                    │   Core #1   │
│             │◄───────────────────│             │
│             │  ② Bookmark        └──────┬──────┘
│             │  (Raft log index)         │ ③ Raft commit
│             │                          ▼
│             │                   ┌─────────────┐
│             │  ④ Read +         │  Read       │
│             │  bookmark ────────►  Replica    │
│             │                   │  (waits for │
│             │◄──────────────────│  log index) │
└─────────────┘  ⑤ Consistent     └─────────────┘
                    result
```

**Explanation:**
The causal bookmark is a Raft log transaction ID. When the client presents this bookmark
on a subsequent read, the Read Replica checks whether it has applied all transactions
up to that log index. If not, it waits (with a configurable timeout) before serving the
read. This provides causal consistency — "your reads reflect your own writes" — without
requiring every read to go through the Leader. The cluster can tolerate Leader failure
and still elect a new Leader from the remaining Core Members via Raft, as long as a
quorum is available.

---

## Diagram 4: Graph Algorithm Comparison — Centrality Measures

```
SAMPLE GRAPH: Information Flow Network
═══════════════════════════════════════════════════════════════════════════

         [A]──────►[B]──────►[C]
          │                   │
          ▼                   ▼
         [D]──────►[E]──────►[F]
          │         ▲         │
          │         │         │
          └────────►[G]◄──────┘
                    │
                    ▼
                   [H]

CENTRALITY MEASURES FOR EACH NODE:
┌──────┬─────────────────┬──────────────────┬───────────────────────────┐
│ Node │ Degree          │ Betweenness      │ PageRank                  │
│      │ (raw connections│ (bridge score)   │ (quality-weighted         │
│      │  — in+out)      │                  │  incoming influence)      │
├──────┼─────────────────┼──────────────────┼───────────────────────────┤
│  A   │ 2 out           │  Low — source    │  Low — nobody points to A │
│  B   │ 1 in, 1 out     │  Low — linear    │  Medium — A points to B   │
│  C   │ 1 in, 1 out     │  Low — linear    │  Medium                   │
│  D   │ 1 in, 2 out     │  Medium          │  Medium                   │
│  E   │ 2 in, 1 out     │  High — bridge   │  High — multiple sources  │
│  F   │ 1 in, 1 out     │  Low             │  Medium                   │
│  G   │ 3 in, 1 out     │  HIGHEST — all   │  HIGHEST — 3 paths lead   │
│      │                 │  paths go thru G │  through high-rank nodes  │
│  H   │ 1 in            │  Low — sink      │  High — G points to H     │
└──────┴─────────────────┴──────────────────┴───────────────────────────┘

USE CASE GUIDANCE:
┌───────────────────────┬────────────────────────────────────────────────┐
│ Degree Centrality     │ Find most active users (who posts most)        │
│ Betweenness Cent.     │ Find bottlenecks, gatekeepers, fraud bridges   │
│ PageRank              │ Find influential accounts (quality connections) │
│ Closeness Cent.       │ Find nodes that can spread info fastest        │
│ Eigenvector Cent.     │ Find nodes connected to other influential nodes │
└───────────────────────┴────────────────────────────────────────────────┘
```

**Explanation:**
Node G illustrates the divergence between centrality measures. G has high betweenness
because ALL paths from {A,B,C,D,E,F} to H pass through G — remove G and H is unreachable
(G is an articulation point). G also has high PageRank because three nodes point to it,
and those nodes themselves receive multiple incoming edges. But G's raw degree (3 in + 1 out)
is not remarkable. In fraud detection, betweenness centrality identifies money mules —
accounts that serve as bridges between fraudsters and clean accounts. In recommendation
systems, PageRank identifies influencers whose product endorsements carry more weight.

---

## Diagram 5: Fraud Detection Ring Pattern — Graph vs SQL

```
MONEY LAUNDERING RING: Graph Representation
═══════════════════════════════════════════════════════════════════════════

External        Clean         Ring Accounts             Exit
Source          Entry         (Obfuscation Layer)       Account
  │               │                                        │
  │ $100K         │ $99K    ┌─── ACC-003 ──┐              │
  ▼               ▼         ▼              ▼   $96K        ▼
[Cash]──►[ACC-001]──►[ACC-002]          [ACC-005]──►[ACC-006]
              ▲         │    ▲              │              │
              │         │    │ $97K         │              │
              │         ▼    │              ▼              │
              │      [ACC-004]──────►[ACC-007]             │
              │         │                  │               │
              │         │ $98K   $95K      │               │
              └─────────┘◄─────────────────┘               │
                 (Ring closes:              └──►[CLEAN_BANK]│
                  money cycles 5 times)

CYPHER DETECTS THIS IN ONE QUERY:
┌─────────────────────────────────────────────────────────────────────────┐
│  MATCH path = (start:Account)-[:TRANSFERRED_TO*3..7]->(start)           │
│  WHERE all(r IN relationships(path) WHERE r.amount > 50000)             │
│  RETURN start.id, length(path), nodes(path)                             │
└─────────────────────────────────────────────────────────────────────────┘

EQUIVALENT SQL REQUIRES RECURSIVE CTE (complex, slow):
┌─────────────────────────────────────────────────────────────────────────┐
│  WITH RECURSIVE ring AS (                                               │
│    SELECT from_acc, to_acc, 1 AS depth, ARRAY[from_acc] AS path,       │
│           amount                                                        │
│    FROM transfers WHERE amount > 50000                                  │
│    UNION ALL                                                            │
│    SELECT t.from_acc, t.to_acc, r.depth + 1, r.path || t.from_acc,     │
│           t.amount                                                      │
│    FROM transfers t JOIN ring r ON t.from_acc = r.to_acc               │
│    WHERE r.depth < 7 AND NOT t.from_acc = ANY(r.path)                  │
│  )                                                                      │
│  SELECT * FROM ring WHERE to_acc = path[1] AND amount > 50000          │
└─────────────────────────────────────────────────────────────────────────┘
                    ↑
         This recursive CTE scans the entire
         transfers table at each level. At depth 7
         with 1B rows, it times out. Neo4j traverses
         only the subgraph reachable from each account.
```

**Explanation:**
This diagram demonstrates the core value proposition of graph databases for fraud detection.
The ring pattern — where money cycles through intermediaries before exiting — is trivially
expressed as a variable-length Cypher path that starts and ends at the same node. The SQL
equivalent requires a recursive CTE that grows exponentially with depth and must implement
cycle prevention manually (the `NOT t.from_acc = ANY(r.path)` check). In a production
fraud detection system processing 500M daily transactions, the Neo4j approach completes in
under 200ms per starting account. The SQL approach on a traditional RDBMS typically times
out beyond depth 4.
