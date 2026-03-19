# Lazy vs Eager Loading — Answers

---

## Q1: Default FetchType for Each Relationship

| Annotation | Default FetchType | Recommended |
|-----------|-------------------|-------------|
| @ManyToOne | EAGER | LAZY |
| @OneToOne | EAGER | LAZY |
| @OneToMany | LAZY | LAZY (keep) |
| @ManyToMany | LAZY | LAZY (keep) |

Change @ManyToOne and @OneToOne to LAZY:
```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```

---

## Q2: When to Use JOIN FETCH vs @EntityGraph

JOIN FETCH is specified in JPQL and gives you full control but requires custom query methods.
@EntityGraph is configuration-based and easier to apply to existing repository methods.

Use @EntityGraph when you want to control loading per-query without writing JPQL.
Use JOIN FETCH when you need complex queries with conditions on the joined entities.

---

## Q3: Why Is OSIV Problematic?

Open Session in View keeps a DB connection occupied for the entire HTTP request lifecycle, including view rendering and JSON serialization time. This reduces available connections for other requests and can cause connection pool exhaustion under load.

Additionally, OSIV hides lazy loading issues during development — code that works with OSIV breaks silently in environments where it's disabled.

The correct approach: Keep transactions as short as possible, convert to DTOs inside the transaction, and never rely on OSIV for lazy initialization.

---

## Q4: @BatchSize vs JOIN FETCH

JOIN FETCH loads all associations in a single query using SQL JOINs. Good for @ManyToOne or when you always need the association.

@BatchSize loads associations in batches — instead of N individual queries, it uses "SELECT WHERE id IN (batch)" queries. Good for @OneToMany when you can't always use JOIN FETCH (e.g., when paginating, JOIN FETCH causes count query issues).

Rule: Use @EntityGraph/JOIN FETCH for @ManyToOne, @BatchSize for @OneToMany with pagination.
