# Chapter 10: Database Selection — Scenario Questions

1. You are the lead architect for a new e-commerce platform expecting 10 million users
   at launch growing to 100 million within 2 years. The system needs: product catalog
   (10M SKUs with rich attributes), shopping cart (session-based, high read/write),
   order history (ACID guarantees required), inventory (real-time stock levels, prevent
   overselling), and product search (full-text + faceted filters + geo-proximity). Design
   the complete database architecture: which database for each concern, why, and how they
   interact. Include the consistency challenges and how you resolve them.

2. Your company built a successful SaaS product on a monolithic PostgreSQL database (2TB,
   300 tables, 500 stored procedures, 50 million users). The database is becoming a
   bottleneck: write latency is 200ms p99, cross-team schema deployments cause conflicts,
   and the on-call team is afraid to touch the stored procedures. Management wants to
   migrate to a microservices + polyglot persistence architecture within 18 months.
   Design the migration plan, sequence the steps, identify the highest-risk steps, and
   explain how you maintain business continuity throughout.

3. A fintech startup is building a digital wallet. Key requirements: real-time balance
   reads (p99 < 50ms), ACID transactions (no lost or duplicated transactions), 10K
   transactions per second peak, regulatory requirement to retain all transaction history
   for 7 years, and the ability to reconstruct any user's balance at any historical point
   in time. Design the database architecture with specific technology choices, schema
   design, and explain how you guarantee no double-spend.

4. Your team maintains a social graph for a professional networking platform (200 million
   users, average 500 connections each). Current tech stack: PostgreSQL for user profiles,
   Cassandra for activity feeds, Neo4j for the connection graph. A new product feature
   requires: "Show me people I might know" (2nd-degree connections) AND "Show their recent
   activity from the last 7 days." This requires joining data from Neo4j and Cassandra.
   How do you design a system that serves this query in under 500ms at scale?

5. You are a staff engineer at a startup that just closed Series B. Your architecture is
   a single PostgreSQL instance (500GB) handling everything: user data, analytics events,
   session storage, and full-text search. The system shows early signs of strain: analytics
   queries cause 100ms latency spikes on user-facing endpoints, search response time is
   2 seconds, and Redis was added for session storage but isn't well integrated. Design
   the polyglot persistence target architecture and create a 6-month migration roadmap.

6. A healthcare analytics company needs to process clinical trial data for 50 pharmaceutical
   clients. Each client's data must be strictly isolated (HIPAA compliance), queryable
   with complex statistical SQL, stored immutably (FDA audit requirements), and accessible
   via a standard SQL interface their data scientists already know. The data is 100GB per
   client per year. Evaluate: PostgreSQL schemas, PostgreSQL row-level security, ClickHouse,
   Snowflake, and a custom sharding approach. Make a recommendation.

7. Your team runs a ride-sharing platform. The driver location system (10K drivers, updates
   every 3 seconds, nearest-driver queries within 500 meters) currently runs on PostgreSQL
   with PostGIS. It works at 10K drivers but product wants to expand to 1 million drivers
   in 6 months. Evaluate: keep PostgreSQL+PostGIS (with partitioning), Redis Geo commands,
   Amazon DynamoDB with geohash, Apache Cassandra with geohash. Provide benchmarks you
   would run and a recommendation.

8. A global media company streams video to 50 million concurrent viewers. The CDN edge
   servers need to make sub-millisecond access control decisions: "Is user X allowed to
   watch content Y in region Z?" These decisions require checking subscription status,
   geographic licensing, device limits (max 3 concurrent streams per account), and parental
   controls. The system processes 500K authorization checks per second globally. Design the
   database architecture for this access control system.

9. You are evaluating whether to use DynamoDB or PostgreSQL for a new IoT device registry.
   Requirements: 10 million devices, each with 50 metadata attributes, 5K reads/second,
   500 writes/second, queries: by device_id (point lookup), by tenant_id (range query),
   by device_type AND location (multi-attribute filter), ACID for device registration.
   Walk through your evaluation, model the data in both DynamoDB and PostgreSQL, and
   make a final recommendation.

10. Your team recently adopted event sourcing for an order management system. Six months
    in, the engineering team reports these operational problems: (1) event replays take 4
    hours for a full state rebuild, (2) the event store is 2TB and growing 100GB/month,
    (3) ad-hoc queries require full event replay, (4) schema evolution is breaking older
    event versions. Diagnose each problem and describe the remediation, including
    snapshotting strategy, event archival, projection store design, and schema versioning
    approach.
