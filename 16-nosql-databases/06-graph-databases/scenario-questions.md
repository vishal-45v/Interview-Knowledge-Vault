# Chapter 06: Graph Databases — Scenario Questions

1. Your team is building a real-time fraud detection system for a payment network processing
   5,000 transactions per second. You need to detect ring fraud — where money cycles through
   3-7 intermediary accounts before reaching the fraudster. Queries must complete in under 200ms.
   How do you model the graph, what Cypher pattern do you write, and how do you ensure
   latency stays under 200ms as the graph grows to 500 million nodes?

2. Your e-commerce recommendation engine uses a Neo4j graph where PURCHASED edges connect
   Users to Products, and SIMILAR_TO edges connect Products. A product node "iPhone 14" has
   47,000 incoming PURCHASED edges. Recommendation queries that traverse this node now take
   8+ seconds. What is the root cause, how do you diagnose it, and what are your mitigation
   strategies without re-modeling the entire graph?

3. Your team runs a Neo4j causal cluster (3 Core Members + 2 Read Replicas) for a social
   network. During a network partition, one Core Member is isolated. Users on Read Replicas
   connected to the isolated member report stale friend lists — data that is 15 minutes old.
   Explain what happened at the consensus layer, and what operational and application-level
   changes you would make.

4. You are migrating a 200-table relational schema for a supply chain management system into
   Neo4j. The existing schema has SUPPLIER, PART, WAREHOUSE, SHIPMENT, and ORDER tables with
   complex many-to-many join tables. Walk through your data modeling process: which entities
   become nodes, which become relationships, and which join-table attributes go on
   intermediate nodes. Show the final Cypher schema.

5. Your knowledge graph for a pharmaceutical company contains 10 million nodes (drugs,
   proteins, diseases, genes) and 80 million relationships. A researcher needs to find all
   drugs that target proteins associated with Alzheimer's disease within 3 hops, filtered by
   clinical trial phase. The query times out after 30 seconds. Diagnose the problem and
   redesign the query and data model for sub-second performance.

6. You are building a LinkedIn-style "degrees of separation" feature. Given any two users,
   show the shortest connection path (up to 4 hops). Your graph has 50 million users with
   an average of 500 connections each. The feature must respond in under 500ms. How do you
   write the Cypher query, and what indexes, caching, or pre-computation strategy do you use?

7. Your team needs to run PageRank on a graph of 100 million nodes nightly to score content
   creators by influence. The GDS library runs out of heap space on your current 32GB Neo4j
   instance. Walk through your options: vertical scaling, graph projection filtering, algorithm
   approximation, or moving to a different tool entirely.

8. You are designing a graph data model for versioned relationships. A WORKS_AT relationship
   between a Person and a Company must record start_date and end_date, and a person can work
   at multiple companies over time. Show two modeling approaches and explain the trade-offs
   in terms of query complexity and traversal performance.

9. Your team uses Amazon Neptune for a social graph. A new compliance requirement mandates
   that all user data for EU residents must be deleted within 72 hours of a deletion request
   (GDPR right to erasure). Neptune does not support ACID transactions across the entire graph.
   How do you design a deletion pipeline that guarantees complete erasure of all nodes and
   edges associated with a user, and how do you verify completeness?

10. You are evaluating whether to use Neo4j or PostgreSQL with recursive CTEs for a
    hierarchical organization chart (100,000 employees, up to 12 levels deep). Engineering
    leadership wants a cost-benefit analysis. Walk through: query complexity comparison,
    performance benchmarks you would run, operational overhead, and your final recommendation
    with justification.
