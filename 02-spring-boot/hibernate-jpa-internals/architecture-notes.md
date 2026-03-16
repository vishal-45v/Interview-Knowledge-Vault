# Hibernate & JPA Internals — Architecture Notes

---

## JPA vs Hibernate Relationship

```
JPA (Jakarta Persistence API)
  ├── Specification (interfaces and annotations only)
  ├── No implementation code
  └── Implemented by:
      ├── Hibernate (most popular)
      ├── EclipseLink
      └── OpenJPA

Spring Data JPA
  ├── Built on top of JPA
  ├── Provides: JpaRepository, query derivation, projections
  └── Uses Hibernate as default JPA provider

javax.persistence (Java EE) → jakarta.persistence (Jakarta EE 9+)
```

---

## Persistence Context Lifecycle

```
Transaction begins
       │
       ▼
EntityManager created (bound to TX via TransactionSynchronizationManager)
       │
       ▼
┌──────────────────────────────────────────────────┐
│  Persistence Context (L1 Cache)                   │
│                                                   │
│  identityMap: {1: Product@abc, 2: Product@def}    │
│  snapshots:   {1: [name="Widget", price=9.99],    │
│                2: [name="Gadget", price=29.99]}    │
│                                                   │
│  Operations tracked:                              │
│  - entity.setPrice(10.99)  ← DIRTY               │
│  - entityManager.persist() ← NEW                  │
│  - entityManager.remove()  ← REMOVED              │
└──────────────────────────────────────────────────┘
       │
       ▼
flush() called (before commit or explicit)
  → Dirty check: compare state to snapshots
  → Generate SQL for dirty/new/removed entities
  → Execute SQL
       │
       ▼
Transaction commits
       │
       ▼
EntityManager.close()
  → Persistence context cleared
  → All entities become DETACHED
```

---

## Hibernate SQL Generation Strategy

```
@GeneratedValue(strategy = GenerationType.IDENTITY)
  → DB auto-increment
  → Hibernate inserts immediately (can't batch)
  → ID available after INSERT

@GeneratedValue(strategy = GenerationType.SEQUENCE)
  → DB sequence
  → Hibernate can pre-fetch IDs (hi-lo algorithm)
  → Enables batch inserts
  → Preferred for bulk operations

@GeneratedValue(strategy = GenerationType.TABLE)
  → Uses a table to simulate sequence
  → Portable but slow (lock on sequence table)
  → Avoid in production

@GeneratedValue(strategy = GenerationType.AUTO)
  → Hibernate picks based on DB dialect
  → Usually SEQUENCE for PostgreSQL, IDENTITY for MySQL
```

---

## Second-Level Cache Regions

```
Hibernate 2L cache has separate regions:

Entity cache:      entityClassName#id → entity state
Collection cache:  entityClassName.collectionName → list of IDs
Query cache:       queryString+params → list of IDs (needs entity cache to work)

Cache strategies:
  READ_ONLY:           Never modified after creation (e.g., countries, currencies)
  NONSTRICT_READ_WRITE: Rare updates, stale data briefly acceptable
  READ_WRITE:          Concurrent updates, uses soft locks
  TRANSACTIONAL:       Full transactions, JTA required
```
