# Chapter 04 — DynamoDB: Theory Questions

Questions covering DynamoDB data model, capacity, indexes, and advanced patterns.
No answers provided — use as verbal practice prompts.

---

1. Describe the DynamoDB data model: what is a table, an item, an attribute? How does
   DynamoDB's schema-on-read model differ from a traditional relational schema?

2. What are the two types of primary keys in DynamoDB? When would you choose partition
   key only vs partition key + sort key, and what new query capabilities does the sort
   key enable?

3. Explain the "hot partition" problem in DynamoDB. Why does a uniformly random partition
   key NOT always prevent hot partitions, and what is "write sharding" — how does it work?

4. What is a Global Secondary Index (GSI) in DynamoDB? What are the limitations of a GSI
   compared to the base table in terms of consistency, storage, and capacity?

5. What is a Local Secondary Index (LSI) in DynamoDB? How does it differ from a GSI in
   terms of consistency guarantees, storage limits, and when it must be defined?

6. Explain DynamoDB's two capacity modes: provisioned with auto-scaling and on-demand.
   Under what workload patterns is each optimal, and what are the cost trade-offs?

7. What is a Read Capacity Unit (RCU)? What is a Write Capacity Unit (WCU)? Calculate
   the RCUs needed to read 10,000 items per second, each item being 6KB, using strongly
   consistent reads. Then calculate using eventually consistent reads.

8. What are DynamoDB Streams? What is their retention period, and what are the four stream
   view types (KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES)?

9. What is DynamoDB Accelerator (DAX)? What caching architecture does it use, and what
   is the difference between item cache and query cache in DAX?

10. Explain the single-table design pattern in DynamoDB. What is the "overloaded key"
    technique, and how do you model one-to-many and many-to-many relationships?

11. What is the DynamoDB `TransactWriteItems` API? What is the maximum number of items
    in a transaction, and what is the cost in WCUs compared to individual writes?

12. What are DynamoDB condition expressions? Give an example of using a condition expression
    to implement optimistic locking on a version attribute.

13. What is DynamoDB TTL? How does it work, how is expiry checked, and what is the
    typical latency between when the TTL expires and when the item is actually deleted?

14. What is PartiQL in DynamoDB? When would you use it over the DynamoDB native API
    (PutItem, GetItem, Query), and what are its limitations?

15. What are DynamoDB Global Tables? How do they handle write conflicts between regions,
    and what is the replication latency between regions?

16. Explain DynamoDB's pagination model: what is `LastEvaluatedKey`, and what happens
    when a Query or Scan returns `LastEvaluatedKey`? How do you implement keyset pagination?

17. What is the `begins_with` operator in DynamoDB and what does it require? What is the
    `between` operator and what data pattern does it enable for sort keys?

18. What is a GSI overload in single-table design? Give an example of using `GSI1PK`
    and `GSI1SK` as overloaded generic attribute names to support multiple entity types.

19. What is DynamoDB adaptive capacity? What problem does it solve automatically, and
    when is it NOT sufficient to solve hot partition issues?

20. What is the difference between a `Scan` and a `Query` in DynamoDB? When is `Scan`
    acceptable in production, and what safeguards should you apply when using it?
