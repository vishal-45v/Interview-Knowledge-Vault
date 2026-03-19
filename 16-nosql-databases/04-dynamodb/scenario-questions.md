# Chapter 04 — DynamoDB: Scenario Questions

Production DynamoDB challenges. Think out loud before reviewing the material.

---

1. **The e-commerce single-table design**
   You are designing a DynamoDB table for an e-commerce platform. The access patterns are:
   (a) Get customer by ID, (b) Get all orders for a customer, (c) Get a specific order by ID,
   (d) Get all items in an order, (e) Get all orders with status "PENDING" (admin view),
   (f) Get all products in a category. Design a single-table schema using overloaded keys,
   specify the base table primary key and all GSIs needed, and show sample items for each
   entity type with their key values.

2. **The hot partition investigation**
   Your DynamoDB table stores user activity events with partition key `user_id` and sort key
   `timestamp`. One user (a bot/crawler) is generating 50,000 writes per second — all going
   to the same partition. Your provisioned WCU is 100,000, but DynamoDB is throttling that
   single user. The CloudWatch metric `ConsumedWriteCapacityUnits` shows 90,000/sec total, but
   you have 10,000+ `ThrottledRequests`. Explain DynamoDB's per-partition throughput limit,
   why adaptive capacity may not help here, and design a solution using write sharding.

3. **The GSI eventually consistent read bug**
   Your application writes a new order to DynamoDB and immediately queries a GSI to get all
   orders for that customer. In testing, 3% of the time the query returns an empty list even
   though the write succeeded. Explain the consistency model of GSI reads, why this race
   condition exists, and at least three strategies to mitigate it without switching to LSI.

4. **The capacity mode migration**
   You have a DynamoDB table in provisioned mode with auto-scaling configured at 50% target
   utilisation, min 100 WCU, max 10,000 WCU. Your workload is: 500 WCU steady-state with
   daily burst to 8,000 WCU for 30 minutes, followed by unpredictable weekly spikes to
   15,000 WCU for 5 minutes. Auto-scaling is reacting too slowly to spikes (it takes 60-90
   seconds to scale up), causing throttling. Design a capacity strategy that handles all
   these patterns, considering on-demand mode, provisioned mode with different auto-scaling
   settings, and burst capacity management.

5. **The DynamoDB Streams Lambda deduplication**
   You are processing DynamoDB Stream events with a Lambda function to propagate changes to
   Elasticsearch. Lambda can invoke your function at least once, meaning the same stream
   record can be delivered multiple times. You have seen duplicate documents in Elasticsearch
   and duplicate audit log entries. Design an idempotent Lambda handler. What attributes
   from the DynamoDB Stream record do you use as a deduplication key, and how do you implement
   idempotency without adding significant latency?

6. **The multi-region write conflict**
   You deploy DynamoDB Global Tables across us-east-1, eu-west-1, and ap-southeast-1.
   A user updates their profile in us-east-1 and simultaneously updates their profile in
   eu-west-1 (two different devices). Both writes succeed locally. When replication propagates,
   there is a write conflict. How does DynamoDB Global Tables resolve this conflict? What is
   "last writer wins" in this context, what determines which write wins, and what data is
   permanently lost? How would you design the application to minimise this risk?

7. **The 400KB item size limit**
   Your product catalogue service stores product items in DynamoDB. Some products have very
   large attribute sets — images (stored as S3 URLs, not inline), technical specifications
   (hundreds of key-value pairs), and a long description (markdown, up to 50KB). The total
   item size frequently exceeds DynamoDB's 400KB limit. Design a solution that keeps the
   most frequently accessed attributes in DynamoDB while handling the overflow, and explain
   the read/write access patterns for both the hot and cold attributes.

8. **The cost explosion after traffic spike**
   Your DynamoDB table is in on-demand mode. A marketing campaign goes viral — traffic spikes
   from 1,000 reads/sec to 500,000 reads/sec for 6 hours. At the end of the month, your
   DynamoDB bill is $45,000 instead of the expected $400. Explain how on-demand pricing works
   at this scale, at what point provisioned mode with reserved capacity becomes cheaper, and
   design a cost-optimised hybrid approach for unpredictable-but-bounded traffic.

9. **The soft-delete and filter expression design**
   Your application soft-deletes records by setting a `deleted_at` timestamp attribute.
   Queries must filter out soft-deleted items. A developer adds a filter expression
   `attribute_not_exists(deleted_at)` to all Query calls. In production, many queries
   return 0 items even though pages of results are available (they were all soft-deleted).
   Explain how DynamoDB applies filter expressions relative to the limit and capacity
   consumption, why this causes problems, and redesign the access pattern to handle
   soft-deletes efficiently.

10. **The migration from multi-table to single-table design**
    Your team has 15 DynamoDB tables, each storing one entity type (users, orders, products,
    inventory, etc.). A new CTO mandates migrating to a single-table design for all entities.
    The tables have different access patterns, RCU/WCU requirements, TTL settings, and Stream
    configurations. Describe the migration strategy: how do you identify access patterns, how
    do you handle entities with very different capacity requirements in a single table, what
    operational risks does consolidation introduce, and when would you push back on this mandate?
