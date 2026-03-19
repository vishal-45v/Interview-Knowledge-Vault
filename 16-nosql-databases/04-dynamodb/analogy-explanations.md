# Chapter 04 — DynamoDB: Analogy Explanations

Analogies for DynamoDB's core concepts, from partition design to single-table patterns.

---

## Analogy 1: DynamoDB Partitions — The Filing Cabinet System

**The Story:**
Imagine a massive office with 1,000 filing cabinets (partitions). Each cabinet is locked with
a key (the partition key hash). When a document arrives, a clerk hashes the document's subject
line to a number 1-1,000 and places it in that cabinet. Cabinet 42 can handle at most 1,000
documents per second being added or retrieved. If every document in the office is about
"Project Phoenix" (a hot partition key), all documents go into Cabinet 42 and everyone fights
for that one cabinet's lock while 999 other cabinets sit empty.

The sort key is like a divider tab within the cabinet — it keeps documents within the cabinet
in alphabetical (sorted) order, enabling you to quickly flip to "all documents about Project
Phoenix from Q4 2024."

**Technical Connection:**
DynamoDB distributes data across partitions using consistent hashing of the partition key.
Each partition has a throughput limit (approximately 3,000 RCU and 1,000 WCU per second).
A hot partition occurs when one partition key receives disproportionate traffic. The sort key
is only meaningful *within* a partition — it enables range queries on the second dimension.

```python
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Documents')

# Hot partition: ALL reads/writes go to partition "project:phoenix"
# Stays within Project Phoenix cabinet (1 partition)
for i in range(10000):
    table.put_item(Item={
        'project_id': 'project:phoenix',   # SAME partition key every time
        'doc_id':     f'doc-{i}',
        'content':    f'Document {i}',
    })

# Sort key enables range queries within the partition (fast, single partition)
response = table.query(
    KeyConditionExpression=Key('project_id').eq('project:phoenix') &
                           Key('doc_id').between('doc-100', 'doc-200'),
)
```

---

## Analogy 2: Single-Table Design — The Universal Spreadsheet

**The Story:**
A traditional relational database is like a well-organised office with separate filing rooms:
one room for Customers, one for Orders, one for Products. Each room has its own labeled
filing system. To find a customer's order, you go to the Customers room, note the customer
number, walk to the Orders room, and find their folder — two trips.

Single-table DynamoDB is like a single giant spreadsheet where all entity types live together.
Instead of labeled rooms, you use a colour-coded system: blue rows are Customers, green rows
are Orders, yellow rows are Products. The spreadsheet is sorted so a customer's row is always
immediately above their orders. You find a customer and their recent orders in one look-down.

The trade-off: the column headers say "PK" and "SK" instead of "customer_id" and "email" —
you need a codebook to know what each colour means.

**Technical Connection:**
Single-table design collocates related entities in the same DynamoDB partition, enabling
retrieval of a parent entity and its children with a single Query operation. The "overloaded"
generic key names (`PK`, `SK`, `GSI1PK`) serve multiple entity types with the same index.

```python
# Single table query: get customer AND their recent orders in one request
response = table.query(
    KeyConditionExpression=Key('PK').eq('CUST#alice'),
    # Returns: profile row + all ORDER# rows for alice in one response
    # Table sorted by SK, so PROFILE comes first, then ORDER#2024... in date order
)

items = response['Items']
customer = next((i for i in items if i['SK'] == 'PROFILE'), None)
orders   = [i for i in items if i['SK'].startswith('ORDER#')]

print(f"Customer: {customer['name']}")
print(f"Order count: {len(orders)}")
# One network round trip to DynamoDB — no JOIN needed
```

---

## Analogy 3: GSI vs LSI — The Library's Cross-Reference Indexes

**The Story:**
A library's main card catalogue (base table) organises books by a unique Book ID (partition
key) and Title (sort key). You can find any book instantly if you know its ID.

A **Local Secondary Index (LSI)** is like a secondary card for books *within the same author's
section* of the main catalogue — you added another sort key (publication year) but only within
each author's collection. It's local to the main catalogue's partitioning. The index cards are
filed right next to the main cards, so they're always perfectly consistent.

A **Global Secondary Index (GSI)** is like a completely separate catalogue in a different room,
organised entirely differently: by Genre (new partition key) and then Publication Year (new
sort key). This catalogue is maintained by a different librarian who updates it when the main
catalogue changes — there's always a slight delay (eventual consistency). This separate room
has its own filing system and can grow independently.

**Technical Connection:**
LSI: same partition key as base table, different sort key. Always strongly consistent. Must be
created at table creation time. Limited to 10GB per partition. GSI: completely different
partition + sort key. Always eventually consistent. Can be added/deleted at any time. Independent
capacity and storage.

```python
# LSI: strongly consistent, same partition, different sort key
# (must be defined at table creation)
# Use case: get orders sorted by total_amount instead of created_at
response = table.query(
    IndexName='OrdersByAmount',  # LSI
    KeyConditionExpression=Key('PK').eq('CUST#alice') &
                           Key('total_amount').gte(Decimal('50.00')),
    ConsistentRead=True,  # ← LSIs support strongly consistent reads!
)

# GSI: eventually consistent, different partition key
# Use case: find all orders with a specific status, across all customers
response = table.query(
    IndexName='GSI_OrderStatus',  # GSI
    KeyConditionExpression=Key('status').eq('PENDING'),
    # ConsistentRead=True  ← NOT allowed on GSI, always eventually consistent
)
```

---

## Analogy 4: DynamoDB Transactions — The Bank Vault Double-Entry

**The Story:**
A bank processes a transfer: $100 moves from Alice's account to Bob's account. The bank uses
double-entry bookkeeping: debit Alice and credit Bob must happen together or not at all. The
bank's auditor (DynamoDB) doesn't just stamp both ledger entries at once — they go through a
two-step process: first, they "flag" both accounts as being in a pending transaction (Phase 1:
Prepare), then they complete both updates simultaneously (Phase 2: Commit). If anything goes
wrong between the flag and the commit, neither account is changed.

The cost: the double-entry process requires twice the work of a simple entry (2x WCU). And if
two auditors are trying to modify Alice's account at the same time, one of them has to wait
or retry.

**Technical Connection:**
DynamoDB uses a two-phase commit protocol for TransactWriteItems. Each item costs 2 WCU (prepare
+ commit). Concurrent transactions on the same item cause `TransactionConflictException` for one
of them. Max 25 items per transaction. Conditions across all items are checked atomically.

```python
import boto3
from botocore.exceptions import ClientError
from decimal import Decimal

dynamodb = boto3.client('dynamodb', region_name='us-east-1')

def atomic_order_and_inventory_update(
    order_id: str,
    customer_id: str,
    product_id: str,
    quantity: int,
    price: Decimal
):
    """
    Atomically: create order + decrement inventory.
    Both succeed or both fail — no partial state.
    Costs: 2 WCU × 2 items = 4 WCU total
    """
    try:
        dynamodb.transact_write_items(
            TransactItems=[
                {   # Create the order item
                    'Put': {
                        'TableName': 'ECommerceTable',
                        'Item': {
                            'PK': {'S': f'CUST#{customer_id}'},
                            'SK': {'S': f'ORDER#{order_id}'},
                            'status': {'S': 'PENDING'},
                            'total': {'N': str(price * quantity)},
                        },
                        'ConditionExpression': 'attribute_not_exists(PK)',  # No duplicates
                    }
                },
                {   # Decrement inventory
                    'Update': {
                        'TableName': 'ECommerceTable',
                        'Key': {
                            'PK': {'S': f'PRODUCT#{product_id}'},
                            'SK': {'S': 'INVENTORY'},
                        },
                        'UpdateExpression': 'SET stock = stock - :qty',
                        'ConditionExpression': 'stock >= :qty',  # Prevent overselling
                        'ExpressionAttributeValues': {
                            ':qty': {'N': str(quantity)}
                        }
                    }
                }
            ]
        )
        return True
    except ClientError as e:
        error_code = e.response['Error']['Code']
        if error_code == 'TransactionCanceledException':
            reasons = e.response.get('CancellationReasons', [])
            for reason in reasons:
                if reason['Code'] == 'ConditionalCheckFailed':
                    return False  # Order exists OR insufficient stock
        raise
```

---

## Analogy 5: DAX (DynamoDB Accelerator) — The Personal Research Assistant

**The Story:**
Every time you ask the company librarian (DynamoDB) a question, they walk to the filing
cabinet, flip through records, and bring you the answer — this takes a few milliseconds.
You hire a personal assistant (DAX) who sits next to you. The first time you ask a question,
they ask the librarian and remember the answer in a notebook. Every subsequent time you ask
the same question (or a similar one you asked before), they answer instantly from memory —
microseconds instead of milliseconds.

If you *update* the filing cabinet, your assistant also updates their notebook immediately
(write-through caching). But if the filing cabinet is updated by someone else (a direct
DynamoDB write), your assistant's notebook might be stale until the notebook's entry expires
(TTL on the cache entry).

**Technical Connection:**
DAX is an in-memory write-through cache cluster for DynamoDB. Item cache: caches individual
item reads (GetItem, BatchGetItem). Query cache: caches the *result sets* of Query and Scan
operations (not individual items, but the entire response). Write-through: writes go to both
DAX and DynamoDB, so the cache is always current for your writes. DAX does NOT cache strongly
consistent reads (they bypass DAX to go directly to DynamoDB).

```python
import amazondax  # DAX client for Python
import boto3

# DAX client — drop-in replacement for DynamoDB resource
dax = amazondax.AmazonDaxClient(endpoints=['dax-cluster.abc123.dax-clusters.us-east-1.amazonaws.com'])
table = dax.Table('Products')

# GetItem — served from DAX item cache if present (microseconds)
# If miss: DAX fetches from DynamoDB and caches it
response = table.get_item(Key={'PK': 'PRODUCT#SKU-001'})
item = response.get('Item')
# Subsequent calls with same key: ~microseconds from DAX cache

# PutItem — writes to DynamoDB AND updates DAX item cache
table.put_item(Item={'PK': 'PRODUCT#SKU-001', 'stock': 249})

# Strongly consistent read — bypasses DAX, goes direct to DynamoDB
# DAX does NOT cache ConsistentRead=True calls
response = table.get_item(
    Key={'PK': 'PRODUCT#SKU-001'},
    ConsistentRead=True  # Goes directly to DynamoDB, ignores DAX cache
)
```

---

## Analogy 6: DynamoDB Streams — The Security Camera Footage Archive

**The Story:**
Every DynamoDB table has an invisible security camera (Streams) recording every change:
new items added, existing items modified, and items removed. Each recording is kept for
exactly 24 hours in a rolling archive. You can review the footage from any point in the
last 24 hours, but anything older is automatically erased.

Multiple security teams (Lambda functions, consumer applications) can each watch the footage
independently and rewind to different timestamps in the 24-hour window. Each team maintains
their own position in the footage (stream shard iterator). The footage shows you what the
room looked like before each change and what it looks like after (NEW_AND_OLD_IMAGES).

**Technical Connection:**
DynamoDB Streams capture item-level changes with 24-hour retention. Stream records include
the old and new images of changed items. Multiple independent consumers can read the same
stream. Lambda event source mappings allow serverless processing. Shard iterators track
each consumer's position. Stream records are ordered *within a shard* (per partition key).

```python
import boto3
import json

def get_stream_records(stream_arn: str, limit: int = 100) -> list:
    """
    Manually read DynamoDB Stream records (useful for testing or custom consumers).
    In production, use Lambda event source mapping for automatic invocation.
    """
    dynamodb_streams = boto3.client('dynamodbstreams')

    # Get the stream's shards
    stream_desc = dynamodb_streams.describe_stream(StreamArn=stream_arn)
    shards = stream_desc['StreamDescription']['Shards']

    all_records = []
    for shard in shards:
        shard_id = shard['ShardId']

        # Get an iterator for the shard (TRIM_HORIZON = from beginning)
        iterator_resp = dynamodb_streams.get_shard_iterator(
            StreamArn=stream_arn,
            ShardId=shard_id,
            ShardIteratorType='TRIM_HORIZON',  # Start from 24h ago
            # ShardIteratorType='LATEST'        # Only new records
            # ShardIteratorType='AT_SEQUENCE_NUMBER' + SequenceNumber=...  # From specific point
        )
        iterator = iterator_resp['ShardIterator']

        # Read records from this shard
        while iterator:
            records_resp = dynamodb_streams.get_records(
                ShardIterator=iterator,
                Limit=limit,
            )
            all_records.extend(records_resp['Records'])
            iterator = records_resp.get('NextShardIterator')
            if not records_resp['Records']:
                break  # No more records in this shard

    return all_records
```

---

## Analogy 7: Write Sharding — The Checkout Line Multiplier

**The Story:**
A popular product (the new iPhone) is released and 100,000 customers try to buy it at once.
If there is only one checkout lane (one partition key = product_id), 100,000 people queue
for one cashier — total chaos, maximum slowness.

Write sharding adds 10 checkout lanes for the same product. Each customer is randomly directed
to one of the 10 lanes (random shard suffix). Each lane moves 10x faster. When you want to
know the *total* inventory sold, the manager walks all 10 lanes and adds up the counts
(scatter-gather read). The individual lanes are fast; the aggregate query does more work.

**Technical Connection:**
Write sharding appends a random suffix (0-N) to the partition key, distributing writes across
N partitions. Each partition handles 1/N of the write traffic. Reads must query all N shards
and merge. This is the fundamental write sharding trade-off: write throughput scales linearly
with N shards, but read complexity increases by N.

```python
import random

def write_sharded_counter(table, key: str, increment: int = 1, num_shards: int = 10):
    """
    Distribute writes across shards for a high-write counter.
    """
    shard = random.randint(0, num_shards - 1)
    sharded_key = f"{key}#SHARD#{shard:02d}"
    table.update_item(
        Key={'PK': sharded_key, 'SK': 'COUNTER'},
        UpdateExpression='ADD #count :inc',
        ExpressionAttributeNames={'#count': 'count'},
        ExpressionAttributeValues={':inc': increment},
    )

def read_sharded_counter(table, key: str, num_shards: int = 10) -> int:
    """
    Aggregate all shards to get total count.
    Requires num_shards reads — scatter-gather.
    """
    total = 0
    for shard in range(num_shards):
        sharded_key = f"{key}#SHARD#{shard:02d}"
        resp = table.get_item(Key={'PK': sharded_key, 'SK': 'COUNTER'})
        item = resp.get('Item', {})
        total += int(item.get('count', 0))
    return total

# For a viral product with 100k writes/sec:
# Without sharding: all writes → 1 partition → throttled at ~1000 WCU/partition
# With 10 shards: writes spread across 10 partitions → 10,000 WCU effective throughput
```
