# N+1 Problem — Structured Answers

---

## Q1: What Is the N+1 Problem?

The N+1 problem occurs when code executes 1 query to load N parent entities, then N additional queries to load child data for each parent. For 100 orders, that's 101 database round trips instead of 1.

Root cause: Lazy loading associations in a loop without upfront JOIN FETCH.

---

## Q2: How to Detect N+1

1. Enable SQL logging: `spring.jpa.show-sql=true`
2. Check Hibernate statistics: `hibernate.generate_statistics=true`
3. Use datasource-proxy library to count queries per request
4. Performance profiling: unusual number of short SQL queries in APM traces

---

## Q3: JOIN FETCH vs @EntityGraph vs @BatchSize

JOIN FETCH: JPQL-level, requires custom @Query. Best when you need conditional queries.

@EntityGraph: Annotation-based, works with derived queries. Best for ad-hoc loading control.

@BatchSize: Entity-level annotation. Loads in batches. Best for paginated queries where JOIN FETCH would break pagination.

---

## Q4: Why Does JOIN FETCH Break Pagination?

When you JOIN FETCH a collection, SQL returns multiple rows per parent (one per child). Hibernate cannot apply LIMIT to the correct number of parent entities at the SQL level — it would cut the result set in the middle of a parent's children.

Hibernate falls back to in-memory pagination (loads ALL data, then filters in Java), logging: "HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!"

Fix: Use two queries — paginate parents, then batch-load children with @BatchSize or a separate query.

---

## Q5: DTO Projections vs Entity Loading

DTO projections with JPQL constructor expressions (SELECT new DTO(o.id, ...)) never load entities and never trigger lazy loading. They're the most efficient approach when you need specific fields from multiple tables.

Trade-off: No entity lifecycle management — you get plain POJOs. Use for read-only reports and list views.
