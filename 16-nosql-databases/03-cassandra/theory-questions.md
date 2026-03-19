# Chapter 03 — Cassandra: Theory Questions

Questions covering Cassandra architecture, data modelling, internals, and operations.
No answers provided — use as verbal practice prompts.

---

1. Describe Cassandra's ring topology. Why is it peer-to-peer instead of master-replica, and
   what are the operational benefits of having no single point of failure?

2. What is a virtual node (vnode) in Cassandra? How many vnodes does Cassandra assign per node
   by default, and why does using vnodes simplify adding/removing nodes from the cluster?

3. Explain Cassandra's consistent hashing. What is the difference between a token and a
   partition key hash, and how does Cassandra use Murmur3 hashing to distribute data?

4. What is the replication factor (RF), and what is the difference between SimpleStrategy
   and NetworkTopologyStrategy? When would you use each, and why is SimpleStrategy
   inappropriate for production multi-datacenter deployments?

5. List all Cassandra consistency levels. For RF=3 across 2 datacenters (3 nodes per DC),
   explain the quorum math for QUORUM, LOCAL_QUORUM, and EACH_QUORUM.

6. What does tunable consistency mean in Cassandra? Give three real examples of different
   R/W consistency level combinations and the trade-offs each makes.

7. Explain the Cassandra write path from client request to disk. Include: coordinator
   selection, commitlog, memtable, SSTable flush, and replication.

8. Explain the Cassandra read path from client request to data. Include: bloom filter,
   key cache, partition summary, partition index, and SSTable data file.

9. What is a memtable? When is it flushed to disk, and what happens to the commitlog
   after a flush?

10. What is an SSTable? Explain the components of an SSTable: data file, index file,
    summary file, bloom filter, compression info, and statistics file.

11. What is a Bloom filter in Cassandra? What does a Bloom filter tell you, and what does
    a Bloom filter NOT tell you? What is the false positive rate and how is it configured?

12. What is a tombstone in Cassandra? Why do tombstones cause problems in high-volume
    delete workloads? What is `gc_grace_seconds` and why must it be set carefully?

13. Explain Cassandra compaction. What are the three main compaction strategies — STCS
    (Size-Tiered Compaction Strategy), LCS (Leveled Compaction Strategy), and TWCS
    (Time Window Compaction Strategy)? For each, describe the ideal workload.

14. Explain the Cassandra data model: partition key, clustering columns, static columns,
    and regular columns. How does the primary key determine data distribution and storage?

15. What is the "query-first" design principle in Cassandra? Why does Cassandra require
    you to design tables around query patterns rather than around entities?

16. What is ALLOW FILTERING in CQL, and why is it dangerous in production? What does
    it actually do at the query execution level?

17. What is a secondary index in Cassandra, and why is it often an anti-pattern on
    high-cardinality columns? What is the difference between a secondary index and a
    materialized view?

18. What is a Cassandra Lightweight Transaction (LWT)? What protocol does it use under
    the hood, and what are the performance implications compared to a regular write?

19. What is the difference between USING TIMESTAMP and TTL in CQL? Can they both be
    set on the same write, and how do they interact with compaction?

20. What are Cassandra 4.x / 5.x major improvements? Include: audit logging, virtual
    tables, transient replication, zero-copy streaming, and the move away from Thrift.
