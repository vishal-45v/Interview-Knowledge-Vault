# Chapter 06: Graph Databases — Follow-Up Traps

## Trap 1: "Graph databases are always faster than relational for connected data"

**What most people say:**
"Yes, because graph databases have O(1) traversal per hop via index-free adjacency, while
SQL requires expensive JOINs that get exponentially slower with depth."

**Correct answer:**
This is context-dependent and the "always" is the trap. For deep traversals (4+ hops),
graph databases win decisively because the cost is proportional to the number of nodes
visited, not the total dataset size. For simple 1-2 hop lookups on small, well-indexed
datasets, PostgreSQL with a covering index is often comparable. The crossover point depends
on average node degree and query depth. Also, index-free adjacency is a Neo4j-specific
claim — graph databases that use adjacency list indexes (like some JanusGraph backends)
do NOT have O(1) per-hop traversal. The real comparison:

- SQL JOIN on 3 tables: O(n * log n) index lookups per JOIN level
- Neo4j 3-hop traversal: O(d^3) where d = average degree, but each step is pointer-chasing

For sparse graphs (social networks, org charts), Neo4j wins at depth > 2.
For dense graphs (full mesh), Neo4j can be slower due to supernode problems.

## Trap 2: "MERGE is just INSERT OR IGNORE in Cypher"

**What most people say:**
"MERGE creates the node if it doesn't exist, and does nothing if it does. Simple upsert."

**Correct answer:**
MERGE is "find or create the ENTIRE PATTERN." If any part of the pattern is missing,
the entire pattern is created. The dangerous trap: MERGE without a constraint-backed
index causes a full node scan on every call.

```cypher
-- DANGEROUS at scale: no index on email, scans ALL User nodes
MERGE (u:User {email: "alice@example.com"})

-- SAFE: create a uniqueness constraint first (auto-creates index)
CREATE CONSTRAINT user_email_unique IF NOT EXISTS
  FOR (u:User) REQUIRE u.email IS UNIQUE;

-- ON CREATE vs ON MATCH lets you handle both branches
MERGE (u:User {email: "alice@example.com"})
ON CREATE SET u.created_at = datetime(), u.status = 'new'
ON MATCH  SET u.last_seen  = datetime()
RETURN u
```

Second trap inside MERGE: it is NOT atomic under concurrent writes. Two concurrent
transactions can both "not find" the node, both attempt CREATE, and the second one
hits a ConstraintViolationException. You must handle this at the application layer with
retry logic, not assume database-level idempotency.

## Trap 3: "You can traverse any direction in Cypher — direction is just a hint"

**What most people say:**
"Cypher lets you omit the arrow direction, so direction doesn't really matter for queries."

**Correct answer:**
Omitting direction works for correctness but DESTROYS performance on large graphs.
Neo4j's node record stores separate linked-list chains for incoming and outgoing
relationships. A bidirectional query must traverse BOTH chains and union the results.

```cypher
-- Bidirectional: traverses both incoming AND outgoing chains, then deduplicates
MATCH (u:User)-[:FOLLOWS]-(other:User) WHERE u.id = $id RETURN other

-- Directional: traverses only the outgoing chain — half the work
MATCH (u:User)-[:FOLLOWS]->(other:User) WHERE u.id = $id RETURN other

-- For a user with 500K follows, the bidirectional version
-- processes 1M relationship records before deduplication
```

If you genuinely need bidirectional semantics (e.g., mutual friends), store two directed
relationships OR use UNION with direction-specific queries for better query plan control.

## Trap 4: "PageRank tells you who has the most connections"

**What most people say:**
"PageRank scores nodes by the number of incoming relationships — it's basically degree
centrality."

**Correct answer:**
PageRank scores nodes by the QUALITY of incoming connections, not just quantity. A node
pointed to by one highly-ranked node can score higher than a node pointed to by 1,000
low-ranked nodes. The algorithm propagates "importance" recursively. Degree centrality
is raw edge count. They produce very different rankings:

```cypher
// PageRank with GDS — dampingFactor is critical
CALL gds.pageRank.stream('myGraph', {
  maxIterations: 20,
  dampingFactor: 0.85   // Probability of following a link vs random jump
})
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC LIMIT 10

// Degree Centrality for comparison
CALL gds.degree.stream('myGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC LIMIT 10
```

The dampingFactor trap: changing from 0.85 to 0.50 dramatically accelerates convergence
but changes final scores significantly. In fraud detection, betweenness centrality (which
nodes act as bridges between communities) is often more useful than PageRank.

## Trap 5: "Cypher WITH is just an alias operator like SQL AS"

**What most people say:**
"WITH lets you rename intermediate results, similar to AS in SQL."

**Correct answer:**
WITH is a PIPELINE BOUNDARY that ends the current scope. Any variable not explicitly
passed through WITH is gone — not just renamed, completely inaccessible. WITH also
enables mid-query aggregation, filtering on aggregated results, and LIMIT before
subsequent MATCH clauses (crucial for performance).

```cypher
MATCH (u:User)-[:PURCHASED]->(p:Product)
WITH u, count(p) AS purchase_count
-- 'p' is GONE here. Referencing it causes a SyntaxError, not NULL.
WHERE purchase_count > 10
-- Must re-traverse from u — no implicit join back to p
MATCH (u)-[:LIVES_IN]->(c:City)
RETURN u.name, purchase_count, c.name

-- WITH LIMIT prunes the working set BEFORE the next MATCH
-- This is a critical performance pattern
MATCH (u:User)-[:FOLLOWS]->(influencer:User)
WITH influencer, count(u) AS follower_count
ORDER BY follower_count DESC LIMIT 100   -- Only process top 100
MATCH (influencer)-[:POSTED]->(post:Post)
RETURN influencer.name, post.title
```

## Trap 6: "shortestPath() in Cypher uses Dijkstra's algorithm"

**What most people say:**
"Yes, shortestPath() finds the optimal shortest path using Dijkstra."

**Correct answer:**
Neo4j's built-in `shortestPath()` uses BFS and finds the path with the fewest HOPS.
It completely ignores relationship weight properties. For weighted shortest path
(actual Dijkstra or A*), you MUST use the GDS library.

```cypher
-- BFS hop-minimization — ignores 'distance' property
MATCH path = shortestPath(
  (a:City {name: 'New York'})-[*]-(b:City {name: 'Los Angeles'})
)
RETURN path, length(path)

-- True weighted Dijkstra via GDS
MATCH (source:City {name: 'New York'}), (target:City {name: 'Los Angeles'})
CALL gds.shortestPath.dijkstra.stream('roadGraph', {
  sourceNode: id(source),
  targetNode: id(target),
  relationshipWeightProperty: 'distance_km'
})
YIELD index, sourceNode, targetNode, totalCost, nodeIds, costs, path
RETURN totalCost, [nodeId IN nodeIds | gds.util.asNode(nodeId).name] AS route
```

Confusing hop-minimization with distance-minimization is one of the most common
graph database mistakes in interviews and in production routing systems.

## Trap 7: "Neo4j causal clustering provides strong consistency for reads"

**What most people say:**
"Yes, causal clustering ensures reads are consistent because all nodes see the same data."

**Correct answer:**
Read Replicas provide EVENTUAL consistency by default. Causal consistency is opt-in
per session using causal bookmarks. Without passing a bookmark, a read immediately
after a write can return stale data from a Read Replica that hasn't yet applied
that transaction — this is a common production bug:

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver(uri, auth=(user, password))

# Write to Core Member (Leader routes it)
with driver.session(database="neo4j") as session:
    session.run("CREATE (u:User {id: $id, name: $name})",
                id="user-123", name="Alice")
    bookmark = session.last_bookmarks()  # MUST capture this

# Read — without bookmark, might hit stale Read Replica
with driver.session(database="neo4j", bookmarks=bookmark) as session:
    result = session.run("MATCH (u:User {id: $id}) RETURN u", id="user-123")
    # Bookmark ensures Read Replica waits for the write to be replicated
    record = result.single()
```

Read Replicas do NOT participate in Raft consensus. They receive replicated data
asynchronously. In high-throughput systems, Read Replicas can lag by seconds.

## Trap 8: "Gremlin and Cypher are interchangeable graph query languages"

**What most people say:**
"They're both graph query languages — you can translate queries between them easily."

**Correct answer:**
They are FUNDAMENTALLY different paradigms. Cypher is declarative (describe the pattern,
let the optimizer choose traversal order). Gremlin is imperative/traversal-based
(describe the walk step by step). This affects optimizer behavior and bug surface:

```cypher
-- Cypher: declarative, optimizer decides join order and traversal direction
MATCH (a:Person {name: 'Alice'})-[:KNOWS*1..3]->(b:Person)
WHERE b.age > 30
RETURN DISTINCT b.name
```

```groovy
// Gremlin: imperative, YOU control every traversal step
g.V().has('Person', 'name', 'Alice')
 .repeat(out('KNOWS').simplePath())  // simplePath() prevents cycles
 .times(3)
 .has('age', gt(30))
 .dedup()
 .values('name')
```

The `simplePath()` trap in Gremlin: forget it, and your traversal enters infinite loops
on cyclic graphs. Cypher automatically prevents cycles in variable-length patterns.
Also, Amazon Neptune's Gremlin implementation does not support all TinkerPop features —
test your queries against the specific Neptune version.

## Trap 9: "The Louvain algorithm is deterministic"

**What most people say:**
"You run Louvain on the same graph and get the same communities every time."

**Correct answer:**
Louvain is NON-DETERMINISTIC by default. The greedy modularity optimization depends
on node processing order, which varies between runs. Two runs produce different but
equally valid community structures. For reproducible results (required for compliance,
A/B testing, or comparing runs over time), you must set a random seed:

```cypher
CALL gds.louvain.stream('socialGraph', {
  randomSeed: 42,          // Ensures reproducibility
  maxIterations: 10,
  maxLevels: 5,
  tolerance: 0.0001
})
YIELD nodeId, communityId, intermediateCommunityIds
RETURN gds.util.asNode(nodeId).name AS name,
       communityId,
       intermediateCommunityIds  // Shows community evolution across hierarchy levels
ORDER BY communityId, name
```

Also, Louvain has a resolution limit — it cannot detect communities smaller than
sqrt(2 * number_of_edges). For fine-grained community detection, consider Leiden
algorithm (an improvement over Louvain) also available in GDS.

## Trap 10: "Graph databases are great for all hierarchical data"

**What most people say:**
"Hierarchies are relationships between nodes, so graph databases are perfect for
organizational charts, file systems, and category trees."

**Correct answer:**
Graph databases handle GENERAL hierarchy well, but for STRICT TREE hierarchies
(no cycles, single parent), a relational database with Closure Table or Nested Set
pattern, or even PostgreSQL's LTREE extension, is often simpler to operate and query.
Graph adds value when the hierarchy has cross-links, multiple parentage paths,
or needs to be traversed alongside other relationship types.

```sql
-- PostgreSQL LTREE for strict hierarchy — often simpler than Neo4j
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT,
    path LTREE  -- e.g., 'electronics.phones.smartphones'
);

CREATE INDEX cat_path_gist ON categories USING GIST(path);

-- Find all descendants of 'electronics'
SELECT * FROM categories
WHERE path <@ 'electronics';

-- Find path between two categories
SELECT * FROM categories
WHERE path @> (SELECT path FROM categories WHERE name = 'iphone_15');
```

Use Neo4j when the hierarchy is part of a larger connected graph (e.g., a product
category connected to users, reviews, and inventory), not when hierarchy is the only
relationship type you need.
