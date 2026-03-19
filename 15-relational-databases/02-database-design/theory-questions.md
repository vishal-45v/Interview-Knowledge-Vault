# Chapter 02 — Database Design: Theory Questions

Senior-level questions covering ER modeling, normalization theory, constraint
design, schema patterns, and the trade-offs that architects face in production.

---

1. What is the difference between an entity, an attribute, and a relationship
   in ER modeling? Give an example where something can reasonably be modeled
   as either an entity or an attribute, and explain how you would decide.

2. Define the three types of cardinality (1:1, 1:N, M:N) and explain how each
   is physically represented in a relational schema. When does a 1:1 relationship
   warrant its own separate table rather than being merged into one table?

3. What is First Normal Form (1NF)? What is an "atomic" value in the context
   of normalization, and why is the definition more nuanced than most textbooks
   suggest?

4. Explain Second Normal Form (2NF). What is a partial dependency, and why does
   2NF only apply to tables with composite primary keys? Give an example of a
   table that violates 2NF and show how to decompose it.

5. Explain Third Normal Form (3NF). What is a transitive dependency? Show an
   example table that is in 2NF but not 3NF and demonstrate the decomposition.

6. What is Boyce-Codd Normal Form (BCNF)? How does it differ from 3NF? Give
   an example of a relation that is in 3NF but not BCNF.

7. What is Fourth Normal Form (4NF) and what are multi-valued dependencies?
   In what real-world scenario would a 3NF-compliant table still contain
   problematic redundancy that 4NF resolves?

8. What are the practical trade-offs of denormalization? Name at least four
   specific scenarios where denormalization is the correct engineering decision
   and explain the write-side cost for each.

9. Compare surrogate keys (e.g., auto-increment integer, UUID) vs natural keys
   (e.g., email, SSN). What are the performance implications, data integrity
   implications, and the main arguments for each in a high-scale system?

10. What is the difference between ON DELETE CASCADE, ON DELETE SET NULL,
    ON DELETE RESTRICT, and ON DELETE NO ACTION? When would each be the
    appropriate choice, and what is the subtle difference between RESTRICT
    and NO ACTION in PostgreSQL?

11. What is a CHECK constraint? What are its limitations compared to application-
    level validation? Can a CHECK constraint reference data in another row or
    another table?

12. What is the difference between UNIQUE and PRIMARY KEY constraints? Can a
    table have multiple UNIQUE constraints? Can a UNIQUE constraint include
    NULL values?

13. What is a star schema? How does it differ from a snowflake schema? In what
    circumstances would you choose one over the other for an analytics workload?

14. What is a soft delete pattern? What are its advantages and the significant
    engineering problems it introduces for queries, indexes, unique constraints,
    and foreign keys?

15. What are temporal tables (also called bi-temporal tables)? What two time
    dimensions do they track, and when would you choose a temporal design over
    soft deletes or an audit log table?

16. What is the purpose of audit columns (created_at, updated_at, created_by)?
    How do you enforce them reliably without relying on application code? What
    additional audit trail design is needed when you need to know what changed,
    not just when?

17. What is a composite key? What are the trade-offs of using a composite primary
    key (e.g., (tenant_id, user_id)) vs a surrogate key with a composite UNIQUE
    constraint? How does this affect foreign key design in child tables?

18. What is the functional dependency notation (X → Y), and how does it relate
    to normalization? Explain the concept of a candidate key using functional
    dependency theory.
