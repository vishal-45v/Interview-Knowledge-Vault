# Chapter 05 — Elasticsearch: Structured Answers

Complete, interview-ready answers with Query DSL JSON and Python elasticsearch-py code.

---

## Q1: Explain the inverted index and BM25 relevance scoring.

**Answer:**

An inverted index maps each unique term in a corpus to the list of documents containing it —
the inverse of a forward index (document → terms). This structure enables O(1) lookup of
"which documents contain this term" regardless of corpus size.

**Construction example:**

```
Forward index (documents → words):
Doc 1: "Elasticsearch is fast and scalable"
Doc 2: "Fast search with inverted index"
Doc 3: "Elasticsearch and Lucene are powerful"

Inverted index (words → document positions):
"elasticsearch" → [Doc1:pos1, Doc3:pos1]
"fast"          → [Doc1:pos3, Doc2:pos1]
"scalable"      → [Doc1:pos5]
"search"        → [Doc2:pos2]
"inverted"      → [Doc2:pos4]
"lucene"        → [Doc3:pos3]
"powerful"      → [Doc3:pos5]

Posting list for "elasticsearch":
┌────────────────────────────────────────────────────┐
│  Term: "elasticsearch"                             │
│  Document frequency (df): 2                        │
│  Postings:                                         │
│  [Doc1: tf=1, positions=[0], norm=1.0]             │
│  [Doc3: tf=1, positions=[0], norm=0.9]             │
└────────────────────────────────────────────────────┘
```

**BM25 Scoring** (default since Elasticsearch 5.0):

```
BM25(q, d) = Σ IDF(qi) × (tf(qi,d) × (k1+1)) / (tf(qi,d) + k1 × (1 - b + b × |d|/avgdl))

Where:
  IDF(qi) = log((N - df + 0.5) / (df + 0.5) + 1)
    N     = total documents in corpus
    df    = number of documents containing term qi

  tf(qi,d) = frequency of term qi in document d
  k1      = saturation factor (default 1.2) — controls TF saturation
  b       = length normalisation factor (default 0.75) — penalises long docs
  |d|     = number of terms in document d
  avgdl   = average document length in corpus
```

**Why BM25 improves on TF-IDF:**
- TF-IDF has linear TF scaling: a term appearing 100 times scores 10x more than one appearing 10 times
- BM25 saturates TF: a term appearing 100 times vs 10 times gets only a modest score difference
  (controlled by `k1`). This prevents long documents with many term repetitions from dominating.
- BM25's length normalisation is configurable (parameter `b`). Setting `b=0` disables length
  normalisation entirely; `b=1` fully normalises by document length.

```python
from elasticsearch import Elasticsearch

es = Elasticsearch()

# Explain why a document scored the way it did
response = es.explain(
    index="products",
    id="1",
    body={
        "query": {"match": {"description": "elasticsearch fast"}}
    }
)
print(response['explanation']['description'])
# Output shows TF, IDF, and BM25 formula components for each term

# Customise BM25 parameters per index
es.indices.create(
    index="custom_search",
    body={
        "settings": {
            "similarity": {
                "custom_bm25": {
                    "type": "BM25",
                    "k1":  1.5,   # Higher = more TF saturation sensitivity
                    "b":   0.5,   # Lower = less length normalisation
                }
            }
        },
        "mappings": {
            "properties": {
                "title": {
                    "type":       "text",
                    "similarity": "custom_bm25"  # Use custom BM25 for this field
                }
            }
        }
    }
)
```

---

## Q2: Explain the Elasticsearch analyzer pipeline with a custom analyzer example.

**Answer:**

```json
// Custom analyzer definition
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "html_strip": {
          "type": "html_strip"
        },
        "ampersand_normalize": {
          "type":        "mapping",
          "mappings":    ["& => and"]
        }
      },
      "tokenizer": {
        "ngram_tokenizer": {
          "type":      "ngram",
          "min_gram":  3,
          "max_gram":  4,
          "token_chars": ["letter", "digit"]
        }
      },
      "token_filter": {
        "english_stop": {
          "type":       "stop",
          "stopwords":  "_english_"
        },
        "english_stemmer": {
          "type":       "stemmer",
          "language":   "english"
        },
        "synonym_filter": {
          "type":       "synonym",
          "synonyms":   ["laptop, notebook, computer", "phone, mobile, smartphone"]
        }
      },
      "analyzer": {
        "product_search_analyzer": {
          "type":         "custom",
          "char_filter":  ["html_strip", "ampersand_normalize"],
          "tokenizer":    "standard",
          "filter":       [
            "lowercase",
            "english_stop",
            "synonym_filter",
            "english_stemmer",
            "unique"
          ]
        },
        "product_index_analyzer": {
          "type":         "custom",
          "char_filter":  ["html_strip"],
          "tokenizer":    "standard",
          "filter":       [
            "lowercase",
            "english_stop",
            "english_stemmer"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type":             "text",
        "analyzer":         "product_index_analyzer",
        "search_analyzer":  "product_search_analyzer"
      },
      "name_keyword": {
        "type":  "keyword"
      }
    }
  }
}
```

```python
from elasticsearch import Elasticsearch

es = Elasticsearch()

# Test what an analyzer produces
response = es.indices.analyze(
    index="my_index",
    body={
        "analyzer": "product_search_analyzer",
        "text":     "The Quick Brown Foxes are Running & jumping"
    }
)
for token in response['tokens']:
    print(f"Token: {token['token']}, position: {token['position']}")
# Output (approximate):
# Token: quick, position: 1
# Token: brown, position: 2
# Token: fox,   position: 3   ← stemmed "Foxes" → "fox"
# Token: run,   position: 5   ← stemmed "Running" → "run"
# Token: and,   position: 7   ← "& => and" via char_filter
# Token: jump,  position: 8   ← stemmed

# "The" removed by stop filter
# "are" removed by stop filter
# All lowercase
```

---

## Q3: Design a search query with relevance tuning using bool and function_score.

**Answer:**

```python
from elasticsearch import Elasticsearch
from datetime import datetime, timedelta

es = Elasticsearch()

def search_products(
    query: str,
    category: str = None,
    max_price: float = None,
    page: int = 0,
    page_size: int = 20,
) -> dict:
    """
    Multi-signal product search with relevance tuning.
    """
    must_clauses = [
        {
            "multi_match": {
                "query":  query,
                "fields": [
                    "name^3",           # Title match is 3x more important
                    "description^1",
                    "brand^2",
                    "tags^1.5",
                ],
                "type":      "best_fields",  # Best matching field wins
                "fuzziness": "AUTO",         # Handle typos
                "operator":  "and",          # All terms must match
            }
        }
    ]

    filter_clauses = [
        {"term": {"status": "active"}},
        {"term": {"in_stock": True}},
    ]

    if category:
        filter_clauses.append({"term": {"category": category}})

    if max_price:
        filter_clauses.append({"range": {"price": {"lte": max_price}}})

    # function_score: boost by multiple signals without overriding relevance
    search_body = {
        "from":  page * page_size,
        "size":  page_size,
        "query": {
            "function_score": {
                "query": {
                    "bool": {
                        "must":   must_clauses,
                        "filter": filter_clauses,
                    }
                },
                "functions": [
                    # Boost recently added products
                    {
                        "gauss": {
                            "created_at": {
                                "origin": "now",
                                "scale":  "30d",   # Half decay at 30 days
                                "offset": "7d",    # No decay for last 7 days
                                "decay":  0.5,
                            }
                        },
                        "weight": 1.5,
                    },
                    # Boost highly-rated products
                    {
                        "field_value_factor": {
                            "field":   "avg_rating",
                            "factor":  1.2,
                            "modifier": "sqrt",     # sqrt(5.0) = 2.24x boost
                            "missing": 1,           # Default for unrated products
                        },
                        "weight": 2.0,
                    },
                    # Boost sponsored products
                    {
                        "filter": {"term": {"is_sponsored": True}},
                        "weight": 3.0,
                    }
                ],
                "score_mode": "multiply",  # Multiply all function scores together
                "boost_mode": "multiply",  # Multiply function score × query score
            }
        },
        "highlight": {
            "fields": {
                "name":        {"number_of_fragments": 0},   # Return full name highlighted
                "description": {"fragment_size": 200},        # 200-char snippet
            },
            "pre_tags":  ["<em>"],
            "post_tags": ["</em>"],
        },
        "aggs": {
            "categories": {
                "terms": {"field": "category", "size": 10}
            },
            "price_ranges": {
                "range": {
                    "field": "price",
                    "ranges": [
                        {"to": 25},
                        {"from": 25, "to": 100},
                        {"from": 100, "to": 500},
                        {"from": 500},
                    ]
                }
            }
        }
    }

    response = es.search(index="products", body=search_body)
    return {
        "total":       response["hits"]["total"]["value"],
        "hits":        response["hits"]["hits"],
        "aggregations": response.get("aggregations", {}),
    }
```

---

## Q4: Implement zero-downtime reindex with alias swap.

**Answer:**

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk
import time

es = Elasticsearch()

def zero_downtime_reindex(
    source_index: str,
    new_mapping: dict,
    alias: str,
    batch_size: int = 1000,
) -> str:
    """
    Reindex from source_index to a new index with updated mapping.
    Uses alias swap for zero-downtime cutover.
    Returns the name of the new index.
    """
    # Step 1: Create new index with correct mapping
    new_index = f"{source_index}_v{int(time.time())}"
    es.indices.create(
        index=new_index,
        body={
            "settings": {
                "number_of_shards":   5,
                "number_of_replicas": 0,         # Disable replicas during reindex
                "refresh_interval":  "-1",        # Disable refresh during reindex
            },
            "mappings": new_mapping,
        }
    )
    print(f"Created index: {new_index}")

    # Step 2: Reindex data (runs in background, check status with task API)
    reindex_body = {
        "source": {
            "index": source_index,
            "size":  batch_size,
        },
        "dest": {
            "index": new_index,
        },
    }

    task = es.reindex(body=reindex_body, wait_for_completion=False)
    task_id = task["task"]
    print(f"Reindex task: {task_id}")

    # Wait for reindex to complete
    while True:
        task_status = es.tasks.get(task_id=task_id)
        if task_status.get("completed"):
            break
        progress = task_status.get("task", {}).get("status", {})
        total    = progress.get("total", 1)
        created  = progress.get("created", 0)
        print(f"Progress: {created}/{total} ({created/max(total,1)*100:.1f}%)")
        time.sleep(10)

    # Step 3: Restore production settings
    es.indices.put_settings(
        index=new_index,
        body={
            "number_of_replicas": 1,
            "refresh_interval":  "1s",
        }
    )

    # Step 4: Force merge to optimise read performance
    es.indices.forcemerge(index=new_index, max_num_segments=5)

    # Step 5: Verify document count matches
    old_count = es.count(index=source_index)["count"]
    new_count = es.count(index=new_index)["count"]
    print(f"Document counts — old: {old_count}, new: {new_count}")
    if old_count != new_count:
        raise Exception(f"Document count mismatch! {old_count} vs {new_count}")

    # Step 6: Atomic alias swap (no downtime — old and new both readable during swap)
    alias_actions = {"actions": []}

    # Remove alias from old index (if it exists)
    if es.indices.exists_alias(index=source_index, name=alias):
        alias_actions["actions"].append(
            {"remove": {"index": source_index, "alias": alias}}
        )

    # Add alias to new index
    alias_actions["actions"].append(
        {"add": {"index": new_index, "alias": alias}}
    )

    es.indices.update_aliases(body=alias_actions)
    print(f"Alias '{alias}' now points to '{new_index}'")

    return new_index

# Usage:
new_index = zero_downtime_reindex(
    source_index="products",
    new_mapping={
        "properties": {
            "product_id":  {"type": "keyword"},   # Fixed: was 'text'
            "name":        {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "price":       {"type": "double"},
            "category":    {"type": "keyword"},
            "description": {"type": "text"},
        }
    },
    alias="products_current",
)
```

---

## Q5: Implement Index Lifecycle Management (ILM) for a logging cluster.

**Answer:**

```python
from elasticsearch import Elasticsearch

es = Elasticsearch()

def setup_logging_ilm():
    """Set up ILM policy and index template for log indices."""

    # Step 1: Create ILM policy
    es.ilm.put_lifecycle(
        policy="logs_policy",
        body={
            "policy": {
                "phases": {
                    "hot": {
                        "min_age": "0ms",
                        "actions": {
                            "rollover": {
                                "max_primary_shard_size": "50gb",
                                "max_age":  "1d",    # New index every day or 50GB
                                "max_docs": 50_000_000,
                            },
                            "set_priority": {"priority": 100}
                        }
                    },
                    "warm": {
                        "min_age": "2d",    # Move to warm 2 days after rollover
                        "actions": {
                            "shrink": {"number_of_shards": 1},  # Consolidate shards
                            "forcemerge": {"max_num_segments": 1},  # 1 segment = read-optimised
                            "set_priority": {"priority": 50},
                            "allocate": {
                                "number_of_replicas": 1,
                                "require": {"box_type": "warm"}  # Move to warm nodes
                            }
                        }
                    },
                    "cold": {
                        "min_age": "30d",   # Move to cold storage 30 days after rollover
                        "actions": {
                            "set_priority": {"priority": 0},
                            "allocate": {
                                "number_of_replicas": 0,  # No replicas in cold
                                "require": {"box_type": "cold"}
                            }
                            # In ES 7.x+: "freeze": {} (deprecated in 7.14)
                            # In ES 8.x: use "searchable_snapshot" action
                        }
                    },
                    "delete": {
                        "min_age": "90d",  # Delete 90 days after rollover
                        "actions": {
                            "delete": {}
                        }
                    }
                }
            }
        }
    )

    # Step 2: Create index template
    es.indices.put_index_template(
        name="logs_template",
        body={
            "index_patterns": ["logs-*"],
            "template": {
                "settings": {
                    "number_of_shards":   5,
                    "number_of_replicas": 1,
                    "index.lifecycle.name":         "logs_policy",
                    "index.lifecycle.rollover_alias": "logs-active",
                    "index.codec":        "best_compression",  # Compress cold data
                },
                "mappings": {
                    "properties": {
                        "@timestamp":  {"type": "date"},
                        "level":       {"type": "keyword"},
                        "service":     {"type": "keyword"},
                        "message":     {"type": "text"},
                        "trace_id":    {"type": "keyword"},
                        "host":        {"type": "keyword"},
                    }
                }
            }
        }
    )

    # Step 3: Create initial index with the rollover alias
    es.indices.create(
        index="logs-000001",
        body={
            "aliases": {
                "logs-active": {"is_write_index": True}
            }
        }
    )

    print("ILM setup complete. Write to 'logs-active' alias.")


# Monitor ILM status
def check_ilm_status(index_pattern: str = "logs-*"):
    response = es.ilm.explain_lifecycle(index=index_pattern)
    for index, info in response["indices"].items():
        phase      = info.get("phase", "unknown")
        step       = info.get("step", "unknown")
        age        = info.get("age", "unknown")
        action     = info.get("action", "unknown")
        print(f"{index}: phase={phase}, action={action}, step={step}, age={age}")

setup_logging_ilm()
check_ilm_status()
```

---

## Q6: Tune Elasticsearch for bulk indexing performance.

**Answer:**

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk, parallel_bulk
import json

es = Elasticsearch(
    hosts=["http://localhost:9200"],
    timeout=120,              # Long timeout for bulk operations
    retry_on_timeout=True,
    max_retries=3,
)

# Step 1: Pre-bulk optimisation settings
def configure_for_bulk_load(index: str):
    es.indices.put_settings(
        index=index,
        body={
            "number_of_replicas": 0,          # No replica writes during bulk
            "refresh_interval": "-1",          # No auto-refresh
            "index.translog.durability": "async",  # Async translog fsync
            "index.translog.sync_interval": "60s", # Fsync every 60s
        }
    )

# Step 2: Bulk indexing with proper batch sizing
def bulk_index_documents(index: str, documents: list, batch_size: int = 1000):
    """
    Bulk index documents with optimal batch size.
    Tune batch_size based on: target ~5-15MB per bulk request
    """
    def doc_generator(docs):
        for doc in docs:
            yield {
                "_index": index,
                "_id":    doc.get("id"),
                "_source": doc,
            }

    success_count = 0
    error_count   = 0

    # parallel_bulk: multiple concurrent bulk requests
    for success, info in parallel_bulk(
        es,
        doc_generator(documents),
        chunk_size=batch_size,
        thread_count=4,          # Parallel threads
        queue_size=8,            # Queue ahead of threads
        raise_on_error=False,    # Don't stop on individual doc errors
    ):
        if success:
            success_count += 1
        else:
            error_count += 1
            print(f"Error: {info}")

    return success_count, error_count

# Step 3: Post-bulk restoration
def restore_after_bulk_load(index: str):
    es.indices.put_settings(
        index=index,
        body={
            "number_of_replicas": 1,
            "refresh_interval":  "1s",
            "index.translog.durability": "request",
        }
    )
    # Force merge to reduce segment count (improves query performance)
    es.indices.forcemerge(
        index=index,
        max_num_segments=5,      # Not 1 — leaves room for future merges
        wait_for_completion=True,
    )
    # Refresh to make all documents searchable
    es.indices.refresh(index=index)

# Performance benchmark rule of thumb:
# Default:         ~5,000-10,000 docs/sec (with 1s refresh, replicas enabled)
# Optimised:       ~50,000-200,000 docs/sec (no replicas, no refresh, async translog)
# With parallel:   ×4 improvement per additional thread (up to I/O saturation)
```
