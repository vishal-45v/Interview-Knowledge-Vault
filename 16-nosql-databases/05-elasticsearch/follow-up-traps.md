# Chapter 05 — Elasticsearch: Follow-up Traps

The follow-up questions that reveal deep Elasticsearch understanding.

---

## Trap 1: "Elasticsearch is a database — you can use it as your primary data store."

**What most people say:**
"Yes, Elasticsearch stores documents and supports complex queries, so it can be a primary database."

**Correct answer:**
Elasticsearch is a search engine, not a primary database. Using it as a primary store has
critical risks:

1. **No ACID guarantees**: Elasticsearch does not support transactions. A bulk index operation
   can partially fail, leaving your index in an inconsistent state.

2. **Data loss risk**: The `refresh_interval` defaults to 1 second, meaning documents are not
   immediately searchable. More critically, the translog (`fsync_interval` default: 5 seconds)
   means up to 5 seconds of data can be lost on node failure. With `index.translog.durability:
   async`, even more data can be lost.

3. **Not designed for point reads**: `GetItem` equivalent (`GET /index/_doc/id`) works, but
   Elasticsearch is not optimised for millions of random point-reads per second. The per-document
   overhead is much higher than a key-value store.

4. **No referential integrity**: No foreign keys, no constraints, no enforce-uniqueness semantics
   (you can index the same document ID twice — the second overwrites the first).

Best practice: use Elasticsearch as a *derived* view of your data, populated from a primary
store (PostgreSQL, DynamoDB, MongoDB) via CDC or event streaming. If Elasticsearch data is
lost, it can be rebuilt from the source of truth.

---

## Trap 2: "Adding more replicas speeds up write throughput in Elasticsearch."

**What most people say:**
"More replicas means more nodes to write to — faster indexing."

**Correct answer:**
Adding replicas *slows down* write throughput because every write must be replicated to all
replica shards. A write to a primary shard triggers N replica writes (one per replica). More
replicas = more replica writes = higher write latency and lower write throughput.

Replicas improve:
- **Read throughput**: queries can be routed to any replica, distributing read load
- **Availability**: if a primary shard is lost, a replica is promoted

For bulk indexing migrations, a common optimisation is:
```json
PUT /my_index/_settings
{
    "number_of_replicas": 0,
    "refresh_interval": "-1"
}
```
This eliminates replica writes and disables the per-second refresh during the bulk load,
dramatically improving write throughput. After the bulk load completes, restore the settings:
```json
PUT /my_index/_settings
{
    "number_of_replicas": 1,
    "refresh_interval": "1s"
}
POST /my_index/_forcemerge?max_num_segments=5
```

---

## Trap 3: "The `filter` clause in a bool query is faster than the `must` clause."

**What most people say:**
"Yes, filter is faster because it doesn't calculate scores."

**Correct answer:**
The `filter` clause is faster for two reasons, but the explanation needs precision:

1. **No scoring**: `filter` clauses do not contribute to the relevance `_score`. This saves
   the BM25 calculation, which is significant for large result sets.

2. **Query cache**: `filter` clauses are eligible for the **Elasticsearch filter cache**
   (a per-shard bit set cache). If the same filter is applied repeatedly (e.g., `status = active`),
   the matching document set is cached as a bit set and subsequent queries with the same filter
   reuse the cached bit set without re-scanning documents.

`must` clauses are NOT cached and recalculate for every query, even if the condition is the same.

Rule of thumb: Put *all* exact-match conditions that don't affect ranking into `filter`. Put
conditions that affect how results are ranked into `must`. Text search (`match`) almost always
goes in `must` because you want the relevance score. Status, date range, and category filters
go in `filter`.

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"name": "running shoes"}}  // affects score — in must
      ],
      "filter": [
        {"term": {"status": "active"}},       // no score contribution — in filter + cached
        {"range": {"price": {"lte": 100}}},   // no score contribution — in filter + cached
        {"term": {"category": "footwear"}}    // no score contribution — in filter + cached
      ]
    }
  }
}
```

---

## Trap 4: "A `match_all` query with a `size: 0` and aggregations is efficient."

**What most people say:**
"Yes — `size: 0` means no documents are returned, so the query is just the aggregation."

**Correct answer:**
`size: 0` means zero documents are returned in the *hits* array, but the query still scans
ALL documents to compute the aggregation. On a 500 million document index, this can take
seconds to minutes. The `size: 0` optimisation only saves the overhead of serialising and
transferring documents — it does not reduce the work done on the data nodes.

For expensive aggregations, the optimisations are:
1. **Filter the dataset** before aggregating (use `query` to narrow the document set)
2. **Pre-aggregate** using data transforms or ingest pipelines to maintain summary indices
3. **Use `execution_hint: map`** for low-cardinality terms aggregations to avoid the default
   global ordinals computation
4. **Use sampling** (`sampler` aggregation) for approximate results on very large datasets
5. **Index for aggregations**: ensure fields used in aggregations have `doc_values: true`
   (the default for keyword and numeric fields)

```json
// SLOW: aggregating all 500M documents
GET /logs-*/_search
{
  "size": 0,
  "aggs": {
    "top_ips": {
      "terms": {"field": "client_ip", "size": 10}
    }
  }
}

// FASTER: filter first, then aggregate
GET /logs-2024-01-15/_search    // specific index, not wildcard across all
{
  "size": 0,
  "query": {
    "range": {"@timestamp": {"gte": "now-1h"}}  // only last hour
  },
  "aggs": {
    "top_ips": {
      "terms": {"field": "client_ip", "size": 10}
    }
  }
}
```

---

## Trap 5: "Elasticsearch `_id` must be unique across the entire cluster."

**What most people say:**
"Yes, document IDs must be globally unique."

**Correct answer:**
Document `_id` must be unique *within an index*, not across the cluster. The same `_id` value
can exist in multiple different indices without conflict. The unique identifier for a document
in Elasticsearch is the tuple: `(index, _id)`.

More importantly: within an index, `_id` uniqueness is *per shard*. The routing function
determines which shard a document goes to: `shard = hash(_id) % number_of_shards` (by default).
Two documents with different `_id` values can go to the same shard; two documents with the same
`_id` in *different shards* of the same index would be a data inconsistency (but this would
require custom routing to create and is normally prevented by the routing function).

```bash
# Same _id in different indices — perfectly valid
PUT /products/_doc/1
{"name": "MacBook Pro"}

PUT /orders/_doc/1           # _id=1 also exists in products — no conflict
{"customer": "alice"}

# Same _id within same index — second write overwrites first (upsert)
PUT /products/_doc/1
{"name": "MacBook Pro 2024"}  # Overwrites the previous doc with _id=1 in products

# Check a document's shard assignment
GET /products/_explain/1
# or
GET /_cat/shards/products?v
```

---

## Trap 6: "You can change a field's mapping type by updating the mapping."

**What most people say:**
"Yes, use the PUT mapping API to update field types."

**Correct answer:**
You **cannot change the mapping type of an existing field** in Elasticsearch. Once a field
is mapped as `text`, it cannot be changed to `keyword` (or any other type). The Lucene index
structure for that field is already built with the original type's encoding.

The only way to change a field's type is:
1. Create a new index with the correct mapping
2. Use the Reindex API to copy data from the old index to the new index
3. Perform an alias swap to point traffic to the new index
4. Delete the old index

This is a fundamental operational constraint that should influence your schema design:
use explicit mappings and index templates from day one. Never rely on dynamic mapping for
production data that you might need to change.

```json
// WRONG: this will fail for existing field type change
PUT /products/_mapping
{
  "properties": {
    "product_id": {
      "type": "keyword"     // ERROR if product_id already mapped as text
    }
  }
}
// Error: "mapper [product_id] of different type, current_type [text], merged_type [keyword]"

// CORRECT: create new index, reindex, swap alias
PUT /products_v2
{
  "mappings": {
    "properties": {
      "product_id": {"type": "keyword"},   // Correct type in new index
      "name":       {"type": "text"}
    }
  }
}

POST /_reindex
{
  "source": {"index": "products"},
  "dest":   {"index": "products_v2"}
}

POST /_aliases
{
  "actions": [
    {"remove": {"index": "products",    "alias": "products_current"}},
    {"add":    {"index": "products_v2", "alias": "products_current"}}
  ]
}
// Application always reads/writes from "products_current" alias — no code change needed
```

---

## Trap 7: "Elasticsearch is real-time — documents are searchable immediately after indexing."

**What most people say:**
"Yes, the document is searchable as soon as the index call returns."

**Correct answer:**
Elasticsearch is **near real-time (NRT)**, not real-time. When you index a document, it is
written to the in-memory buffer and the translog. It is NOT searchable until the next *refresh*
operation creates a new Lucene segment from the buffer and makes it visible to searches.

By default, the `refresh_interval` is `1s` — documents become searchable within about 1 second
of indexing. But if you index a document and immediately search for it (within that 1 second
window), you may not find it.

For the `GetItem` equivalent (`GET /index/_doc/id`), the document IS immediately available
(it reads from the translog). For searches, you must wait for the refresh cycle.

```python
import elasticsearch
from time import sleep

es = elasticsearch.Elasticsearch()

# Index a document
es.index(index="products", id="1", body={"name": "MacBook Pro"})

# Search immediately — may return 0 hits!
result = es.search(index="products", body={"query": {"match": {"name": "MacBook"}}})
print(result['hits']['total']['value'])  # Might be 0 — not yet refreshed

# Option 1: Force refresh (expensive — use only in tests)
es.index(index="products", id="1", body={"name": "MacBook Pro"}, refresh=True)
# refresh="wait_for" blocks until the next scheduled refresh

# Option 2: Wait for refresh interval
sleep(1.1)
result = es.search(index="products", body={"query": {"match": {"name": "MacBook"}}})
print(result['hits']['total']['value'])  # Now returns 1

# Option 3: Use GET /index/_doc/id for known-ID lookups (always real-time)
response = es.get(index="products", id="1")  # Immediate, reads translog
```

---

## Trap 8: "You should always shard an index as many times as possible for parallelism."

**What most people say:**
"More shards = more parallelism = faster queries."

**Correct answer:**
Over-sharding is one of the most common Elasticsearch performance mistakes. Each shard is a
Lucene index with its own overhead:

1. **Memory**: Each shard requires file handles, heap memory for segment metadata, and filter
   cache entries. 1,000 shards × ~30MB overhead = 30GB overhead just in metadata.

2. **Query overhead**: A query is executed on every shard in parallel. For a 1000-shard index,
   the coordinator merges 1,000 partial results. For a single query returning 10 documents,
   this coordination overhead dominates.

3. **Cluster state**: Every shard's metadata is stored in the cluster state, which must be
   held by every node in memory. Large shard counts bloat the cluster state and slow down
   master node operations.

**The rule of thumb**: aim for shards between **10GB and 50GB**. Calculate:
`number_of_shards = ceil(expected_total_data_GB / target_shard_size_GB)`

```python
def calculate_shards(
    total_data_gb: float,
    target_shard_size_gb: float = 30,
    num_nodes: int = 6,
    replicas: int = 1,
) -> dict:
    """Calculate optimal shard count."""
    import math
    primary_shards = max(1, math.ceil(total_data_gb / target_shard_size_gb))
    total_shards   = primary_shards * (1 + replicas)
    shards_per_node = total_shards / num_nodes

    return {
        "primary_shards":  primary_shards,
        "total_shards":    total_shards,
        "shards_per_node": round(shards_per_node, 1),
        "recommendation":  "OK" if 1 <= shards_per_node <= 20 else "Adjust"
    }

print(calculate_shards(total_data_gb=180, num_nodes=6))
# {'primary_shards': 6, 'total_shards': 12, 'shards_per_node': 2.0, 'recommendation': 'OK'}
```

---

## Trap 9: "A `nested` query searches nested objects — same as a regular object query."

**What most people say:**
"Nested and object arrays work the same way for queries."

**Correct answer:**
Elasticsearch flattens regular `object` arrays. Consider an order with two line items:
```json
{
  "order_id": "O1",
  "items": [
    {"product": "MacBook", "qty": 1},
    {"product": "Charger",  "qty": 2}
  ]
}
```
With a regular `object` mapping, this is stored internally as:
```
items.product: ["MacBook", "Charger"]
items.qty: [1, 2]
```

A query for `items.product = "MacBook" AND items.qty = 2` would match this document because
`MacBook` and `2` both exist in the flattened arrays — but they correspond to *different* items.
This is the "false cross-object match" problem with regular objects.

`nested` type stores each array element as a separate internal document, preserving the
association between fields within the same array element. A nested query on `items.product = "MacBook"
AND items.qty = 2` would NOT match the above document because MacBook has qty=1, not qty=2.

The cost: nested queries execute a join between the parent document and its nested children,
which is significantly more expensive than a flat query. Each nested document is also indexed
separately, increasing the total document count and memory usage.

---

## Trap 10: "Elasticsearch handles time zones automatically for date range queries."

**What most people say:**
"Yes, Elasticsearch converts dates to UTC internally."

**Correct answer:**
Elasticsearch stores dates as **milliseconds since epoch (UTC)** internally. Date field values
submitted as strings are parsed according to the field's `format` and the timezone specified in
the indexing request. If no timezone is specified, the date string is interpreted as UTC.

The subtle issue: if your application sends date strings without timezone info (e.g., "2024-01-15T10:00:00"),
and your servers are in different time zones, you get inconsistent data. A date of "10am" from
a US server (UTC-5) is stored as "3pm UTC," while the same "10am" from a UK server (UTC+0) is
stored as "10am UTC" — they represent different moments.

For search queries with date ranges:
```json
// Correct: specify timezone in the query
GET /events/_search
{
  "query": {
    "range": {
      "event_date": {
        "gte": "2024-01-15",
        "lt":  "2024-01-16",
        "time_zone": "America/New_York"   // query interprets dates in this tz
      }
    }
  }
}

// If event_date was stored without tz info:
// "2024-01-15T00:00:00" in Eastern Time = "2024-01-15T05:00:00Z" in UTC
// The range query accounts for this correctly with time_zone specified
```

Always store dates with UTC timestamps (`2024-01-15T05:00:00Z`) and specify `time_zone` in
queries where timezone-aware date ranges are needed.

---

## Trap 11: "You can use `wildcard` queries freely for flexible searching."

**What most people say:**
"Wildcard queries are flexible — use them when you need prefix or suffix matching."

**Correct answer:**
`wildcard` queries (especially leading wildcards: `*keyword`) are extremely expensive because
they must iterate over every term in the inverted index to find matches. A query like `*phone*`
on a field with 10 million unique terms must check every term.

Performance rules for search flexibility:
1. **Prefix matching**: Use `prefix` query or map the field with `search_as_you_type` for
   efficient prefix lookups (uses edge n-grams internally)
2. **Suffix/infix matching**: Requires an n-gram analyzer at index time — expensive storage
   but fast query time
3. **Wildcard queries**: Acceptable ONLY on `keyword` fields with low cardinality and only
   for trailing wildcards (`keyword*`). Leading wildcards are almost always wrong.
4. **Full-text search**: Use `match`, `multi_match`, or `match_phrase` — these go through the
   analyzer and inverted index, which is the efficient path

```json
// SLOW: leading wildcard on high-cardinality field
GET /products/_search
{
  "query": {"wildcard": {"product_name": {"value": "*phone*"}}}
}

// FAST: use match query with ngram analyzer for infix matching
GET /products/_search
{
  "query": {"match": {"product_name.ngram": "phone"}}
}

// Mapping for efficient infix search:
// "product_name": {
//   "type": "text",
//   "analyzer": "standard",
//   "fields": {
//     "ngram": {"type": "text", "analyzer": "ngram_analyzer"}
//   }
// }
```
