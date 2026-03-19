# Chapter 10: Database Selection — Theory Questions

1. Describe a principled framework for database selection. What questions do you ask
   in what order, and why is "access patterns first" the correct starting point rather
   than "scale first" or "technology familiarity first"?

2. What is polyglot persistence? Describe a realistic production system that uses
   5 different databases simultaneously, explain why each is the right tool for its
   specific job, and discuss the operational and consistency challenges this creates.

3. Compare the CAP/PACELC positioning of PostgreSQL, Cassandra, DynamoDB, MongoDB,
   Redis, and Elasticsearch. For each database, state whether it is CP or AP (with nuance),
   and explain the PACELC tradeoffs (latency vs consistency during normal operation).

4. Explain the Strangler Fig pattern for database migration. How does it differ from a
   "big bang" migration, and what are the specific implementation challenges when
   migrating from a monolithic MySQL database to a set of microservice-specific databases?

5. What is dual-write during a database migration? What are the consistency problems
   with naive dual-write, and how do you implement it correctly using an event-driven
   approach with a comparison/reconciliation layer?

6. Explain CQRS (Command Query Responsibility Segregation) with separate read and write
   databases. What consistency model does this create, how do you keep read and write
   models synchronized, and when does CQRS add more complexity than value?

7. What is event sourcing? How does an append-only event store differ from a traditional
   database, what is a "projection," and how do you rebuild a projection from an event log?

8. Explain the database-per-service pattern in microservices architecture. What isolation
   does it provide, what new challenges does it introduce, and how do you handle data
   that needs to be shared or joined across service boundaries?

9. What is the difference between embedding and referencing in a document store like
   MongoDB? Describe the decision criteria for each, and how does the choice affect
   read performance, write performance, and consistency?

10. Compare PostgreSQL vs MongoDB for a content management system storing articles,
    authors, tags, and comments. Where does MongoDB's document model provide a genuine
    advantage, and where does it create problems compared to PostgreSQL?

11. For a financial ledger system (immutable transaction records, exact balance queries,
    regulatory reporting), compare: PostgreSQL, DynamoDB, Cassandra, and an event store.
    Which do you choose and why?

12. What are the total cost of ownership (TCO) considerations when comparing a managed
    database service (e.g., AWS RDS Aurora, DynamoDB, Atlas MongoDB) to self-hosted
    open-source alternatives? Which hidden costs are most commonly underestimated?

13. What is vendor lock-in in the context of databases? Compare the lock-in risk of
    DynamoDB vs MongoDB vs PostgreSQL, and describe a multi-cloud database strategy
    that mitigates lock-in without sacrificing performance.

14. For a social media feed (write-heavy, read-heavy, eventually consistent acceptable,
    personalized per user), compare PostgreSQL, Cassandra, and a denormalized approach
    using Redis. Which would you choose for 100M users, and how does your answer change
    for 100K users?

15. How do you choose between a relational schema and a NoSQL schema for IoT sensor data
    (1M devices, 10-second resolution, 3-year retention, queries: per-device range,
    cross-device aggregation)? What factors tip the decision toward each option?

16. What is shadow reading in the context of database migration? How do you use it to
    validate a new database before cutting over, and what metrics do you compare between
    the old and new system?

17. When does using multiple databases in a microservices architecture require distributed
    transactions (2PC), and when can you avoid them with eventual consistency patterns
    like the Saga pattern or outbox pattern?

18. For a search-heavy application (full-text search, faceted navigation, geo-search,
    spell correction), compare PostgreSQL full-text search, Elasticsearch, and Meilisearch.
    What workload characteristics push you toward the dedicated search engine vs keeping
    everything in PostgreSQL?
