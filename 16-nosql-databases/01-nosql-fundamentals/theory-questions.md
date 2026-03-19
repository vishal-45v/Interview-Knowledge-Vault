# Chapter 01 — NoSQL Fundamentals: Theory Questions

These questions test deep recall of distributed systems theory as it applies to NoSQL databases.
No answers are provided — use these as flashcard prompts or verbal practice starters.

---

1. State the CAP theorem precisely. What does "consistency" mean in CAP's definition, and how does
   it differ from the "C" in ACID?

2. Explain why "pick 2 of 3" in CAP is a misleading shorthand. What does a network partition
   actually force you to choose between, and what happens when there is no partition?

3. What is the PACELC theorem? How does it extend CAP, and what does the "ELC" part capture that
   CAP cannot?

4. Define BASE (Basically Available, Soft state, Eventually consistent). For each property, give a
   concrete example of what it looks like in a running distributed database.

5. List and explain the four ACID properties. Then explain why achieving all four across multiple
   nodes in a distributed system is expensive — what distributed protocols are required?

6. What is eventual consistency? What guarantees does it provide and what does it explicitly NOT
   guarantee? Give an example of a violation under concurrent updates with last-write-wins.

7. Explain read-your-writes consistency. What must a system do at the infrastructure level to
   provide this guarantee across multiple replicas?

8. What is monotonic reads consistency? How does session affinity (sticky sessions) relate to it,
   and what breaks when a client's session is redirected to a lagging replica?

9. What is causal consistency? How does it differ from sequential consistency and linearisability?
   Rank these models from weakest to strongest: eventual, causal, sequential, linearisable.

10. Describe how vector clocks work. What information does a vector clock carry, what does it mean
    when two clocks are concurrent vs when one dominates, and what does "dominate" mean precisely?

11. What are Lamport timestamps? How are they generated, what ordering guarantee do they provide,
    and what can they NOT tell you that vector clocks can?

12. What is a CRDT (Conflict-free Replicated Data Type)? Explain the difference between state-based
    (CvRDT) and operation-based (CmRDT) CRDTs. Give two concrete examples of CRDT data structures.

13. Explain quorum reads and quorum writes. Given a replication factor N, write quorum W, and read
    quorum R, what condition must hold for strong consistency? What happens if W + R <= N?

14. What is hinted handoff in Cassandra / Riak? What problem does it solve, what are its limits,
    and how does the system recover from a hint-delivery failure?

15. What is read repair? Distinguish between synchronous (blocking) read repair and asynchronous
    (background) read repair. What is the downside of aggressive synchronous read repair?

16. What is anti-entropy repair (Merkle tree reconciliation)? How does a Merkle tree allow two
    nodes to quickly identify which key ranges differ without transferring all data?

17. Describe the Paxos consensus algorithm at a high level: which roles exist, what are the two
    phases, and what safety guarantee does it provide?

18. How does the Raft consensus algorithm differ from Paxos in terms of understandability and
    operational properties? What is the role of the Raft leader, and what happens during
    a leader election?

19. What are the five main NoSQL database categories? For each, name a leading open-source or
    cloud-native representative and its primary access pattern.

20. When should an engineer choose a NoSQL database over a relational database? List at least five
    concrete technical reasons (not just "it's faster" or "it scales").

21. What is schema flexibility in document and wide-column stores, and what operational problems
    can it cause at scale? How do schema registries and schema validation help?

22. Explain the "hot partition" problem in hash-based distributed databases. How does write
    sharding (key suffix randomisation) mitigate it, and what new complexity does it introduce
    for read aggregation?

23. What is consistent hashing? How does adding a virtual node to a ring affect the load
    distribution across physical nodes, and why are virtual nodes (vnodes) preferred over
    single tokens per node?

24. What is the difference between a CP system and an AP system during a network partition?
    Give a real-world example of each choice made by a well-known database.

25. Explain "last write wins" (LWW) conflict resolution vs "multi-value" conflict resolution.
    What are the correctness risks of LWW under clock skew, and how do hybrid logical clocks
    (HLC) address them?
