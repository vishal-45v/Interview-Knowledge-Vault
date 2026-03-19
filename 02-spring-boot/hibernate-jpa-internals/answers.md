# Hibernate & JPA Internals — Structured Answers

---

## Q1: Four Entity States in JPA

1. **Transient**: Newly created, not associated with any persistence context. Not in DB.
2. **Persistent (Managed)**: Associated with an open persistence context. Changes are automatically tracked.
3. **Detached**: Was persistent, but persistence context was closed or entity was evicted. Changes not tracked.
4. **Removed**: Marked for deletion. Will be deleted from DB on flush.

State transitions:
- new → persistent: persist()
- persistent → detached: evict(), close(), end of transaction
- detached → persistent: merge()
- persistent → removed: remove()

---

## Q2: How Does Dirty Checking Work?

When an entity is loaded into the persistence context, Hibernate stores a snapshot of its initial state. On flush (before commit or explicit flush call), Hibernate compares the current state of all managed entities to their snapshots. Any differences generate UPDATE SQL statements.

This is why you don't need to call save() after modifying a managed entity inside a transaction.

Performance note: Dirty checking compares ALL fields. For entities with many fields, use @DynamicUpdate to only include changed fields in the UPDATE.

---

## Q3: What Is the N+1 Problem?

When loading a collection relationship lazily, Hibernate executes 1 query for the parent entities and then N additional queries for each child collection — one per parent. For 100 orders, this means 101 queries.

Solutions:
- JOIN FETCH in JPQL: SELECT o FROM Order o JOIN FETCH o.items
- @EntityGraph: specify eager paths per query
- @BatchSize: load collections in batches instead of one-by-one

---

## Q4: First-Level vs Second-Level Cache

| Cache | Scope | Enabled | Storage |
|-------|-------|---------|---------|
| L1 (Persistence Context) | Single EntityManager/Transaction | Always | JVM heap |
| L2 | SessionFactory/Application | Must configure | JVM heap or distributed |
| Query Cache | Application | Must configure | JVM heap or distributed |

L1 cache: Calling findById() twice in the same transaction returns the same object with no additional SQL.

L2 cache: Entity is shared across transactions. Must configure @Cache on entity and a cache provider (EhCache, Caffeine, Redis).

---

## Q5: merge() Behavior with Transient vs Detached Entity

merge(entity) behavior:
- If entity has no ID: INSERT (treats as new entity)
- If entity has ID and exists in DB: Load from DB, copy state, UPDATE on flush
- If entity has ID and NOT in DB: INSERT (or exception, depending on configuration)

merge() always returns a new managed instance. The original object passed to merge() remains detached.

---

## Q6: @OneToMany Cascade Types

- PERSIST: Cascade persist to child entities
- MERGE: Cascade merge to child entities
- REMOVE: Cascade delete to child entities (dangerous with bidirectional relationships)
- REFRESH: Cascade refresh from DB
- DETACH: Cascade detach
- ALL: All of the above

Common pattern: CascadeType.ALL + orphanRemoval = true for parent-owns-children (composition relationship).

Avoid REMOVE on entities that have independent existence (use only for tight composition).
