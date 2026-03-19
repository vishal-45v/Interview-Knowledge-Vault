# Chapter 06: Graph Databases — Theory Questions

1. Explain the property graph model. What are its four core primitives (nodes, relationships,
   properties, labels), and how does each map conceptually to constructs in the relational model?
   Where does the mapping break down?

2. What is index-free adjacency in Neo4j? How does it give graph traversals O(1) per hop instead
   of the O(log n) index lookup or O(n) table scan a relational database would require for a JOIN?

3. Neo4j stores data in a native graph format on disk. Describe the physical storage layout —
   the fixed-size record files for nodes, relationships, and properties. Why are fixed-size records
   critical to the O(1) adjacency claim?

4. What is the difference between a directed relationship and an undirected query in Cypher? If
   you store FOLLOWS as directed (Alice→Bob), can you query it bidirectionally? Show the Cypher.

5. Explain the difference between `MERGE` and `CREATE` in Cypher. When would using `CREATE`
   instead of `MERGE` cause data integrity issues at scale, especially under concurrent writes?

6. What is the `WITH` clause in Cypher used for? How does it behave like a pipeline stage? What
   happens to variables that are not carried forward in a `WITH` clause?

7. Describe Neo4j's causal clustering architecture. What is the difference between Core Members
   and Read Replicas? How does write routing differ from read routing, and what consistency
   guarantees does a "causal bookmark" provide?

8. What is the difference between PageRank centrality and Betweenness Centrality in graph
   algorithms? In a fraud detection graph, which would you choose and why?

9. Explain Dijkstra's algorithm vs A* (A-star) for shortest path in a weighted graph. Under what
   graph conditions does A* outperform Dijkstra? What additional data does A* require that
   Dijkstra does not?

10. What is the Louvain algorithm for community detection? How does it define "community" using
    the modularity score Q? What is the time complexity, and why is it preferred over spectral
    clustering for large graphs?

11. Compare the property graph model (Neo4j, Neptune/Gremlin) to the RDF/triple-store model
    (SPARQL). What are the fundamental philosophical and technical differences? When would you
    choose RDF over property graph?

12. What is APOC (Awesome Procedures On Cypher) in Neo4j? Describe three categories of
    operations where APOC is essential because pure Cypher cannot accomplish them.

13. Describe the Graph Data Science (GDS) library's execution model. What is the difference
    between a "named graph projection" and an "anonymous graph projection"? Why does GDS
    project graphs into in-memory representations before running algorithms?

14. When should you NOT use a graph database? Give at least four concrete scenarios where a
    relational database, document store, or key-value store is the correct choice.

15. What is the "intermediate node" pattern in graph data modeling? Give a real-world example
    where a direct relationship between two nodes is insufficient and an intermediate node with
    its own properties is required.

16. What is a "supernode" (or "dense node") in a graph database? How does it affect query
    performance and Cypher query planning? What data modeling techniques can mitigate the
    supernode problem?

17. Amazon Neptune supports both Gremlin (property graph) and SPARQL (RDF) over the same
    underlying storage. What are the operational and query-expressiveness trade-offs of using
    Neptune vs self-hosted Neo4j for a production recommendation engine?

18. In graph data modeling, should you store bidirectional relationships as two directed edges
    (A→B and B→A) or rely on Cypher's ability to traverse both directions on a single directed
    edge? What are the performance and storage implications of each choice?
