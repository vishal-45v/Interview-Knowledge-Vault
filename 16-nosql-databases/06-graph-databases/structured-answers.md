# Chapter 06: Graph Databases — Structured Answers

## Q1: Explain index-free adjacency and why it makes graph traversal fundamentally different from SQL JOINs.

**Answer:**

Index-free adjacency means each node in Neo4j's native graph storage contains direct
memory pointers to its adjacent relationships. There is no lookup table or index to
consult during traversal — following a relationship is a single memory pointer dereference.

**Physical storage layout in Neo4j:**

```
Node Record (15 bytes, fixed-size):
┌──────────────┬──────────────────┬──────────────────┬────────────────┐
│ inUse (1B)   │ nextRelId (4B)   │ nextPropId (4B)  │ labelId (4B)  │
│              │ ← ptr to 1st rel │ ← ptr to 1st prop│                │
└──────────────┴──────────────────┴──────────────────┴────────────────┘

Relationship Record (34 bytes, fixed-size):
┌─────────┬──────────┬──────────┬────────┬──────────┬──────────┬──────────┐
│ inUse   │ firstNode│ secondNode│ type  │ firstPrevRel│firstNextRel│ ...  │
│ (1B)    │ (4B)     │ (4B)     │ (4B)  │ (4B)     │ (4B)     │          │
└─────────┴──────────┴──────────┴────────┴──────────┴──────────┴──────────┘
```

Because records are fixed-size, the position of node N's record in the node store file is
`N * 15 bytes` — an O(1) seek. Following a relationship is O(1). SQL JOIN requires:
1. Index lookup: O(log n) in a B-tree
2. For each match, another O(log n) lookup on the next table
3. For 5-hop traversal: O(5 * log n) vs Neo4j's O(5 * 1)

**Traversal comparison:**

```cypher
-- Neo4j: 4-hop traversal — performance is proportional to nodes VISITED, not dataset size
MATCH (fraud:Account)-[:SENT_TO*1..4]->(target:Account {flagged: true})
WHERE fraud.country <> target.country
RETURN fraud, target

-- Equivalent SQL: requires 4 self-joins, each scans the transfers table
SELECT DISTINCT t1.from_account, t4.to_account
FROM transfers t1
JOIN transfers t2 ON t1.to_account = t2.from_account
JOIN transfers t3 ON t2.to_account = t3.from_account
JOIN transfers t4 ON t3.to_account = t4.from_account
JOIN accounts a   ON t4.to_account = a.id AND a.flagged = true
JOIN accounts src ON t1.from_account = src.id AND src.country <> a.country
-- This query plan degrades as transfers table grows; Neo4j's does not
```

The key insight: SQL JOINs scale with the SIZE of the tables being joined. Neo4j
traversals scale with the SIZE of the subgraph being traversed. For sparsely connected
data relative to the full dataset, Neo4j has a decisive advantage.

---

## Q2: Design a fraud detection graph model and write a Cypher query to detect money laundering rings.

**Answer:**

Money laundering rings involve circular fund flows: money enters the ring, cycles through
intermediaries to obscure origin, and exits. The graph model captures this naturally.

**Graph schema:**

```cypher
// Constraints and indexes
CREATE CONSTRAINT account_id_unique IF NOT EXISTS
  FOR (a:Account) REQUIRE a.id IS UNIQUE;

CREATE CONSTRAINT transaction_id_unique IF NOT EXISTS
  FOR (t:Transaction) REQUIRE t.id IS UNIQUE;

// Node labels: Account, Transaction, Person, Bank
// Relationship types: OWNS (Person -> Account), SENT (Account -> Transaction),
//                    RECEIVED (Transaction -> Account), WORKS_AT (Person -> Bank)

// Sample data
CREATE (alice:Person {id: 'P001', name: 'Alice', risk_score: 0.2})
CREATE (bob:Person   {id: 'P002', name: 'Bob',   risk_score: 0.8})
CREATE (acc1:Account {id: 'ACC-001', balance: 50000, country: 'US'})
CREATE (acc2:Account {id: 'ACC-002', balance: 1000,  country: 'US'})
CREATE (acc3:Account {id: 'ACC-003', balance: 49000, country: 'BZ'})  // Belize
CREATE (alice)-[:OWNS]->(acc1)
CREATE (bob)-[:OWNS {since: date('2023-01-15')}]->(acc2)
CREATE (acc1)-[:TRANSFERRED_TO {amount: 49000, timestamp: datetime()}]->(acc2)
CREATE (acc2)-[:TRANSFERRED_TO {amount: 48500, timestamp: datetime()}]->(acc3)
CREATE (acc3)-[:TRANSFERRED_TO {amount: 48000, timestamp: datetime()}]->(acc1)
```

**Fraud ring detection query:**

```cypher
// Detect circular fund flow (rings of length 2-6)
MATCH path = (start:Account)-[:TRANSFERRED_TO*2..6]->(start)
WHERE all(rel IN relationships(path) WHERE rel.amount > 10000)
  AND length(path) > 1
WITH start, path,
     reduce(total = 0, r IN relationships(path) | total + r.amount) AS total_flow,
     [r IN relationships(path) | r.timestamp] AS timestamps
WHERE total_flow > 100000
  // Ensure transfers happen in time order (not all cycles are fraud)
  AND timestamps[0] < timestamps[size(timestamps)-1]
RETURN start.id AS ring_entry_account,
       length(path) AS ring_length,
       total_flow,
       [n IN nodes(path) | n.id] AS account_chain
ORDER BY total_flow DESC
LIMIT 100
```

**Layering and structuring (3-phase laundering detection):**

```cypher
// Phase 1: Placement — large cash deposits from unknown sources
MATCH (acc:Account)<-[:TRANSFERRED_TO]-(external)
WHERE external.source = 'CASH' AND external.amount > 9000  // Just under CTR threshold
WITH acc, count(external) AS suspicious_deposits
WHERE suspicious_deposits > 5

// Phase 2: Layering — rapid movement through intermediaries
MATCH (acc)-[:TRANSFERRED_TO*2..5]->(layered:Account)
WHERE layered.country <> acc.country

// Phase 3: Integration — funds return to clean account
MATCH (layered)-[:TRANSFERRED_TO]->(clean:Account)-[:OWNED_BY]->(person:Person)
WHERE person.occupation IN ['Real Estate', 'Luxury Goods']
RETURN acc.id, suspicious_deposits, collect(DISTINCT clean.id) AS exit_accounts
```

---

## Q3: Explain Neo4j causal clustering — how writes flow, how reads are routed, and what a causal bookmark guarantees.

**Answer:**

Neo4j Causal Clustering has three member types with distinct roles:

**Architecture:**
- **Leader (1):** Accepts all writes, participates in Raft consensus
- **Followers (2+):** Participate in Raft consensus, can serve reads
- **Read Replicas (0+):** Async replication, serve reads only, do NOT vote in Raft

**Write flow:**
1. Client sends write to any Core Member
2. Core Member routes write to the current Leader (via cluster topology discovery)
3. Leader appends transaction to its Raft log
4. Leader sends AppendEntries RPC to all Followers
5. Once majority (quorum) acknowledge, transaction is committed
6. Leader returns success with a TRANSACTION ID (the causal bookmark)
7. Read Replicas eventually receive the committed transaction asynchronously

**Causal bookmark mechanism:**

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(
    "neo4j+routing://neo4j-cluster:7687",
    auth=("neo4j", "password")
)

# Write session — routed to Leader
with driver.session(database="neo4j") as write_session:
    write_session.run("""
        MERGE (u:User {id: $user_id})
        SET u.email = $email, u.updated_at = datetime()
    """, user_id="usr-456", email="bob@example.com")

    # Bookmark = the Raft log index of this committed transaction
    bookmark = write_session.last_bookmarks()
    print(f"Write committed, bookmark: {bookmark}")

# Read session — can be served by any Read Replica
# Bookmark ensures Read Replica WAITS until it has applied up to that log index
with driver.session(database="neo4j", bookmarks=bookmark) as read_session:
    result = read_session.run(
        "MATCH (u:User {id: $user_id}) RETURN u.email", user_id="usr-456"
    )
    email = result.single()["u.email"]
    # Guaranteed to see the write above — causal consistency
```

**What the bookmark does NOT guarantee:**
- It does NOT guarantee you read from the Leader (still hits Read Replica)
- It does NOT provide linearizability across different sessions
- It ensures THIS session sees THIS session's writes — not other sessions' writes

**Cluster sizing for production:**
- Minimum: 3 Core Members (tolerates 1 failure)
- Recommended: 5 Core Members (tolerates 2 failures) + N Read Replicas for read scaling
- Read Replicas are the primary read-scaling mechanism; adding Core Members beyond 5
  INCREASES write latency (more Raft participants = more acknowledgments needed)

---

## Q4: Walk through the Graph Data Science (GDS) library workflow and explain why graph projection is necessary.

**Answer:**

GDS operates on in-memory graph projections rather than the on-disk Neo4j graph.
This separation is fundamental: GDS algorithms need fast random access to adjacency
data during iterative computation, which disk-based Neo4j storage cannot provide at
algorithm speeds.

**The GDS workflow:**

```cypher
// Step 1: Create a Named Graph Projection
// Projects a subgraph from Neo4j into GDS in-memory format
CALL gds.graph.project(
  'socialInfluence',          // Projection name
  {
    User: {                   // Node projection
      label: 'User',
      properties: ['age', 'country', 'join_date']
    }
  },
  {
    FOLLOWS: {                // Relationship projection
      type: 'FOLLOWS',
      orientation: 'NATURAL',  // NATURAL, REVERSE, or UNDIRECTED
      properties: ['weight', 'since']
    }
  }
)
YIELD graphName, nodeCount, relationshipCount, projectMillis
// Returns: socialInfluence, 50000000, 2500000000, 45230 (45 seconds to project)
```

**Why projection exists:**
1. GDS algorithms iterate over all nodes/edges many times (PageRank: 20+ iterations)
2. Disk I/O for each iteration would be prohibitively slow
3. In-memory adjacency arrays enable CPU cache-friendly traversal
4. Projection allows filtering: only project relevant node labels and rel types
5. Projection applies early filters, reducing memory footprint

**Running algorithms on the projection:**

```cypher
// Step 2: Run Algorithm (stream mode returns results, write mode persists to graph)
CALL gds.pageRank.write('socialInfluence', {
  maxIterations: 20,
  dampingFactor: 0.85,
  writeProperty: 'pagerank_score',  // Written back to Neo4j node properties
  scaler: 'MinMax'                   // Normalize scores to 0-1
})
YIELD nodePropertiesWritten, ranIterations, didConverge, centralityDistribution
RETURN nodePropertiesWritten, ranIterations, didConverge,
       centralityDistribution.mean AS mean_score,
       centralityDistribution.p99  AS p99_score

// Step 3: Use written scores in subsequent Cypher queries
MATCH (u:User) WHERE u.pagerank_score > 0.8
RETURN u.name, u.pagerank_score ORDER BY u.pagerank_score DESC LIMIT 20

// Step 4: ALWAYS drop the projection when done to free memory
CALL gds.graph.drop('socialInfluence')
```

**Memory estimation before projection (critical for production):**

```cypher
// Estimate memory requirements BEFORE actually creating the projection
CALL gds.graph.project.estimate(
  {User: {label: 'User'}},
  {FOLLOWS: {type: 'FOLLOWS'}}
)
YIELD requiredMemory, treeView
// Output: requiredMemory: "4 GiB", treeView shows breakdown
// If this exceeds available heap, projection will OOM
```

---

## Q5: Design a data model for a recommendation engine using Neo4j, including the Cypher query for collaborative filtering.

**Answer:**

Collaborative filtering in graph form: "Users who bought what you bought also bought X."

**Graph schema:**

```cypher
// Constraints
CREATE CONSTRAINT user_id   FOR (u:User)    REQUIRE u.id IS UNIQUE;
CREATE CONSTRAINT product_id FOR (p:Product) REQUIRE p.id IS UNIQUE;
CREATE CONSTRAINT category_id FOR (c:Category) REQUIRE c.id IS UNIQUE;

// Indexes for filter properties
CREATE INDEX user_country FOR (u:User) ON (u.country);
CREATE INDEX product_category FOR (p:Product) ON (p.category_id);

// Relationships
// (User)-[:PURCHASED {timestamp, quantity, price_paid}]->(Product)
// (User)-[:VIEWED    {timestamp, duration_seconds}]->(Product)
// (User)-[:RATED     {score: 1-5, timestamp}]->(Product)
// (Product)-[:BELONGS_TO]->(Category)
// (Product)-[:SIMILAR_TO {similarity_score}]->(Product)  // Pre-computed
```

**Collaborative filtering query:**

```cypher
// "Customers who bought this also bought" — pure graph traversal
MATCH (target:User {id: $user_id})-[:PURCHASED]->(bought:Product)
    <-[:PURCHASED]-(similar_user:User)-[:PURCHASED]->(recommended:Product)
WHERE NOT (target)-[:PURCHASED]->(recommended)    // Exclude already purchased
  AND NOT (target)-[:VIEWED {duration_seconds: {min:0}}]->(recommended)  // Exclude viewed
  AND recommended.in_stock = true
WITH recommended,
     count(DISTINCT similar_user) AS co_purchasers,
     collect(DISTINCT bought.name)[0..3] AS common_purchases
WHERE co_purchasers >= 5   // Minimum signal threshold
RETURN recommended.id     AS product_id,
       recommended.name   AS product_name,
       recommended.price  AS price,
       co_purchasers,
       common_purchases
ORDER BY co_purchasers DESC
LIMIT 20
```

**Advanced: Weighted hybrid recommendation (CF + content-based):**

```cypher
// Combine collaborative filtering score with content similarity
MATCH (target:User {id: $user_id})-[r:RATED]->(liked:Product)
WHERE r.score >= 4   // Only highly-rated products drive recommendations
WITH target, liked, r.score AS rating

// CF signal: other users who rated this highly
MATCH (liked)<-[r2:RATED]-(peer:User)-[r3:RATED]->(candidate:Product)
WHERE r2.score >= 4 AND r3.score >= 4
  AND NOT (target)-[:PURCHASED|RATED]->(candidate)
  AND id(peer) <> id(target)

// Content signal: category similarity
MATCH (liked)-[:BELONGS_TO]->(cat:Category)<-[:BELONGS_TO]-(candidate)

WITH candidate,
     count(DISTINCT peer) AS peer_signal,
     avg(r3.score) AS avg_peer_rating,
     rating AS source_rating,
     1.0 AS content_bonus   // Category match bonus

WITH candidate,
     (peer_signal * 0.6 + avg_peer_rating * 0.3 + content_bonus * 0.1) AS composite_score
WHERE composite_score > 2.0
RETURN candidate.id, candidate.name, round(composite_score, 2) AS score
ORDER BY score DESC LIMIT 10
```

**Handling the supernode problem for popular products:**

```cypher
// Product with 500K PURCHASED edges — naive traversal is extremely slow
// Solution: sample instead of full traversal
MATCH (target:User {id: $user_id})-[:PURCHASED]->(bought:Product)
WHERE bought.purchase_count < 100000  // Filter out supernodes (very popular items)

// OR: Use WITH LIMIT to cap intermediate results
MATCH (bought)<-[:PURCHASED]-(peer:User)
WITH peer, bought LIMIT 1000  // Sample at most 1000 co-purchasers per product
MATCH (peer)-[:PURCHASED]->(candidate:Product)
...
```

---

## Q6: Compare property graph vs RDF triple stores — when does RDF win?

**Answer:**

**Core data model difference:**

Property Graph (Neo4j, Neptune/Gremlin):
- Node has: ID, labels (set), properties (map)
- Relationship has: ID, type (single string), direction, properties (map)
- Data lives in the graph structure itself

RDF (Apache Jena, Stardog, Neptune/SPARQL):
- Everything is a triple: (Subject, Predicate, Object)
- URIs identify subjects and predicates globally
- No concept of "relationship properties" — requires reification
- Designed for global knowledge sharing (Linked Data)

```sparql
-- SPARQL: find drugs that treat diseases related to a protein
PREFIX bio: <http://example.org/bio/>
PREFIX schema: <http://schema.org/>

SELECT ?drug ?disease ?protein WHERE {
  ?drug     bio:treats      ?disease .
  ?disease  bio:associated_with  ?protein .
  ?drug     schema:clinicalPhase ?phase .
  FILTER(?phase = "Phase 3")

  -- Federated query: pull from external ontology database
  SERVICE <http://dbpedia.org/sparql> {
    ?protein rdfs:label ?proteinName .
    FILTER(lang(?proteinName) = "en")
  }
}
```

**When RDF wins:**

1. **Interoperability**: RDF uses global URIs — two datasets using the same URI for
   "Person" are automatically interoperable. Property graphs have no global namespace.

2. **Ontology reasoning**: RDF supports OWL (Web Ontology Language), enabling inference.
   If you state "owl:equivalentClass" between two concepts, the reasoner automatically
   infers equivalences. Property graphs have no built-in reasoning.

3. **Federated queries**: SPARQL's SERVICE keyword queries external SPARQL endpoints
   in one query. No equivalent in Cypher.

4. **Government/academic/life sciences data**: W3C standards compliance, LOD (Linked
   Open Data) integration. FHIR (healthcare), schema.org, WikiData all speak RDF.

5. **Knowledge graph requiring provenance**: Named graphs in RDF let you track WHERE
   each triple came from (named graph = context/provenance). Property graphs lack this.

**When property graph wins:**

1. Performance for deep traversal (RDF triplestores are slower for graph traversal)
2. Simpler data model for application developers
3. Relationship properties without reification complexity
4. Better tooling ecosystem (Neo4j Browser, Bloom, etc.)
5. When you don't need semantic reasoning or global interoperability

```python
# RDF reification (adding properties to a triple) is verbose
# "Alice KNOWS Bob since 2020" in RDF requires 4 triples:
# _:stmt rdf:subject   :Alice
# _:stmt rdf:predicate :knows
# _:stmt rdf:object    :Bob
# _:stmt :since        "2020"

# In property graph Cypher:
# (alice:Person)-[:KNOWS {since: 2020}]->(bob:Person)
# Much cleaner for application development
```
