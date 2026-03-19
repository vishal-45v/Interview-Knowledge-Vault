# Chapter 05 — Elasticsearch: Theory Questions

Questions covering Elasticsearch architecture, internals, query DSL, and operations.
No answers provided — use as verbal practice prompts.

---

1. Describe the Elasticsearch cluster architecture: what are the four node types (master,
   data, coordinating, ingest), what are their roles, and why should these roles be
   separated in a large production cluster?

2. What is an inverted index? Explain how it is constructed from a document corpus,
   what a posting list contains, and why it enables fast full-text search compared to
   a forward index.

3. What is TF-IDF scoring? Define term frequency, inverse document frequency, and explain
   how each contributes to relevance ranking. How does BM25 (the default in Elasticsearch 5+)
   improve on TF-IDF?

4. What is the difference between an Elasticsearch `index` and a `shard`? What is a
   primary shard vs a replica shard, and how are they used for reads vs writes?

5. What is a Lucene segment? How does segment merging work, and why can a large number
   of segments degrade search performance?

6. What is the `refresh_interval` in Elasticsearch? What happens during a refresh, what
   is the default value, and when would you change it?

7. What is the `flush` operation in Elasticsearch (distinct from refresh)? What does it
   do to the translog, and when does it occur?

8. Explain the difference between `text` and `keyword` field types in Elasticsearch.
   What are the implications for searching, sorting, and aggregating on each type?

9. What is dynamic mapping in Elasticsearch? What are the risks of unconstrained dynamic
   mapping in production, and what is "mapping explosion"?

10. Explain the Elasticsearch analyzer pipeline: character filters → tokenizer → token
    filters. Give an example of a custom analyzer and what it would do to the input
    "The Quick Brown Foxes are Running".

11. What is the difference between a `term` query and a `match` query? When would you
    use each, and what happens to the search string in each case?

12. Explain the `bool` query in Elasticsearch. What is the difference between `must`,
    `should`, `must_not`, and `filter` clauses? Which clauses contribute to the
    relevance score and which do not?

13. What are nested objects in Elasticsearch? Why are nested objects needed instead of
    regular object mappings, and what is the performance cost of nested queries?

14. What is a `terms` aggregation? What is the `doc_count_error_upper_bound` field in
    the aggregation response, and why does it appear for multi-shard aggregations?

15. What is a `date_histogram` aggregation? What parameters control its behaviour, and
    how does it differ from a `histogram` aggregation?

16. Explain Elasticsearch Index Lifecycle Management (ILM). What are the four ILM phases
    (hot, warm, cold, delete), and what typical operations are performed in each phase?

17. What is an Elasticsearch alias? What are the two primary use cases for aliases, and
    how do you perform a zero-downtime index reindex using an alias swap?

18. What is the `doc_values` data structure? How does it differ from `fielddata`, and
    why is `fielddata` disabled by default for `text` fields?

19. What is shard sizing in Elasticsearch? What is the recommended shard size range,
    and what are the consequences of having too many small shards or too few large shards?

20. What are the key differences between Elasticsearch and OpenSearch? When was the fork
    created, and what features have diverged in major versions?
