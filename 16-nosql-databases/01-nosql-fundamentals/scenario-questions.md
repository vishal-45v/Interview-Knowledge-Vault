# Chapter 01 — NoSQL Fundamentals: Scenario Questions

Real-world engineering scenarios at senior / staff level. Answer out loud before reading follow-up
material — the process of verbalising reveals gaps that silent reading hides.

---

1. **The shopping cart problem**
   Your team is building a shopping cart service for an e-commerce platform that processes
   150,000 orders per hour during peak sale events. The product manager insists that the cart
   must always be writable (customers should never see a "service unavailable" error), even
   during network partitions between data centres. The finance team insists that a customer
   should never be able to check out with the same item appearing twice due to a split-brain
   event. These two requirements are in direct tension. Walk through the CAP trade-off, name
   which NoSQL systems you would consider, and explain exactly how you would model the cart
   to satisfy the most critical constraint.

2. **The stale read complaint**
   Your analytics dashboard reads from a replica that is consistently 3-5 seconds behind the
   primary. Users are reporting that after they submit a form update, the dashboard still shows
   the old value. The replication lag is within your SLA, but the user experience is broken.
   Without changing the database, what consistency model would you implement at the application
   layer? What are at least two implementation strategies, and what are their trade-offs?

3. **The multi-region write conflict**
   You operate a user-profile service across three AWS regions (us-east-1, eu-west-1,
   ap-southeast-1). Each region accepts writes locally for low latency. A user travels from
   New York to London and updates their profile picture from two mobile devices almost
   simultaneously — one request hits us-east-1, one hits eu-west-1. The writes replicate
   asynchronously. Design the conflict resolution strategy. Which CRDT or LWW approach would
   you apply, and what are the consistency guarantees to the end user?

4. **The quorum misconfiguration incident**
   A Cassandra cluster with replication factor RF=3 is running with consistency level ONE for
   both reads and writes (W=1, R=1). An SRE notices that after a node failure, some reads are
   returning stale data from 24 hours ago. Explain the sequence of events that caused this,
   what quorum math was violated, and how you would reconfigure reads and writes without
   causing a performance regression that kills the 99th-percentile latency SLA.

5. **The Merkle tree repair storm**
   Your operations team runs `nodetool repair` across a 48-node Cassandra cluster and triggers
   a cascading CPU spike that takes the cluster to 95% CPU for 6 hours. Three nodes OOM-crash.
   What happened at the Merkle tree / anti-entropy level? How should repair be scheduled and
   throttled in production? What alternative repair strategies exist in newer Cassandra versions?

6. **The event sourcing + eventual consistency mismatch**
   Your team is building an event-sourced order management system on top of DynamoDB. Each
   order is a stream of events (OrderPlaced, PaymentProcessed, OrderShipped). A downstream
   consumer reads the event stream and builds a read model. During a burst, the consumer
   falls 10,000 events behind. Business logic that depends on the latest state (e.g. "has this
   order already shipped?") is making wrong decisions. Describe the consistency model in play,
   why it is failing here, and at least two architectural changes to fix it without abandoning
   eventual consistency.

7. **The clock skew LWW disaster**
   Your distributed key-value store uses last-write-wins with wall-clock timestamps from each
   node. One node's NTP daemon is misconfigured and its clock drifts 8 seconds ahead of the
   other nodes. Describe what will happen to writes during this window, which writes will be
   silently discarded, and how the introduction of Hybrid Logical Clocks (HLC) would prevent
   this class of data loss.

8. **The vector clock explosion**
   A Riak cluster is storing shopping basket objects using vector clocks for conflict
   resolution. After 18 months in production, the engineering team notices that some basket
   objects have vector clocks with 300+ entries, causing read amplification and large object
   sizes. What causes vector clock bloat? What is "vector clock pruning," and what consistency
   risk does pruning introduce?

9. **The Raft split-brain**
   You are running a 5-node Raft cluster (etcd) across 3 availability zones: 2 nodes in AZ-A,
   2 nodes in AZ-B, 1 node in AZ-C. A network partition isolates AZ-A from the rest. Describe
   exactly what happens to leader election and write availability. What is the minimum number
   of nodes that can form a quorum, and which partition will elect a new leader? What happens
   to writes that were in-flight to the isolated AZ-A nodes?

10. **The polyglot persistence migration**
    Your monolith uses a single PostgreSQL database for everything: user accounts, product
    catalogue, session state, activity feeds, and real-time inventory counts. You are tasked
    with breaking this apart into purpose-fit data stores. For each of the five data domains,
    name the NoSQL category and a specific database you would choose, justify the choice using
    the access pattern, and describe the consistency model each domain requires.
