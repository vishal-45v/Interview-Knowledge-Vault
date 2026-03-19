# Chapter 04 — DynamoDB: Follow-up Traps

The questions that reveal whether a candidate truly understands DynamoDB's constraints.

---

## Trap 1: "DynamoDB auto-scaling handles traffic spikes automatically."

**What most people say:**
"Yes, just set up auto-scaling and DynamoDB handles load changes automatically."

**Correct answer:**
DynamoDB auto-scaling has a critical limitation: it reacts to *sustained* high utilisation,
not to instantaneous spikes. The auto-scaling cooldown and scale-up cycle typically takes
60-90 seconds. A 60-second spike that exceeds provisioned capacity will cause throttling
for the entire duration of the spike before auto-scaling reacts.

DynamoDB also provides **burst capacity**: each partition has a token bucket that accumulates
capacity at 5 minutes of unused provisioned throughput. This absorbs short spikes (seconds to
a minute) but is not reliable for planned burst events.

For predictable spikes (sale events, marketing emails), the correct approach is to:
1. Pre-warm by manually increasing provisioned capacity before the spike
2. Use `UpdateTable` to set higher capacity 30+ minutes before the event
3. Set auto-scaling max capacity to the expected peak (auto-scaling cannot scale above max)

For truly unpredictable spikes, **on-demand mode** is the right choice — it scales to any
throughput instantly (within AWS's limits) but costs ~7x more per read/write unit than
provisioned.

```python
import boto3

dynamodb = boto3.client('dynamodb', region_name='us-east-1')
autoscaling = boto3.client('application-autoscaling')

def pre_warm_for_event(table_name: str, expected_writes: int):
    """Manually scale up before a known traffic event."""
    # Direct table update (immediate, not through auto-scaling)
    dynamodb.update_table(
        TableName=table_name,
        BillingMode='PROVISIONED',
        ProvisionedThroughput={
            'ReadCapacityUnits': expected_writes,
            'WriteCapacityUnits': expected_writes,
        }
    )
    print(f"Pre-warmed {table_name} to {expected_writes} WCU/RCU")
```

---

## Trap 2: "A GSI is just like a secondary index in a relational database."

**What most people say:**
"Yes, a GSI lets you query on any attribute, like a SQL index."

**Correct answer:**
GSIs are fundamentally different from SQL secondary indexes:

1. **Storage**: A GSI is a *separate distributed table* that stores a projection of your data.
   It consumes its own storage and its own capacity units independently from the base table.

2. **Consistency**: GSI reads are *always* eventually consistent. You cannot perform a
   strongly consistent read on a GSI (unlike the base table which supports strongly consistent
   reads). If you just wrote an item, a GSI query may not return it for up to seconds.

3. **Capacity**: GSI writes consume GSI write capacity units. Every write to the base table
   that changes a GSI-indexed attribute consumes 1 WCU on the GSI in addition to the base
   table WCU. A table with 5 GSIs where every write touches all 5 GSI attributes = 6x WCU cost.

4. **Sparse GSIs**: DynamoDB only indexes items in a GSI if the GSI key attributes *exist*
   on the item. Items without those attributes are *not indexed* — a feature, not a bug.
   This enables efficient "filtered" indexes.

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

# GSI query — eventually consistent, cannot be changed
response = table.query(
    IndexName='GSI_CustomerOrders',
    KeyConditionExpression='customer_id = :cid AND begins_with(order_date, :year)',
    ExpressionAttributeValues={
        ':cid': 'CUST#alice',
        ':year': '2024',
    },
    # ConsistentRead=True  ← this would FAIL for a GSI
)

# Sparse GSI example: only index items with status='PENDING'
# Items without 'pending_since' attribute are NOT in the GSI
# This creates an efficient "inbox" of only pending items
table.put_item(Item={
    'PK': 'ORDER#001',
    'SK': 'METADATA',
    'status': 'pending',
    'pending_since': '2024-01-15T10:00:00Z',  # GSI key — item IS indexed
})
table.put_item(Item={
    'PK': 'ORDER#002',
    'SK': 'METADATA',
    'status': 'shipped',
    # No 'pending_since' → NOT in GSI (sparse index)
})
```

---

## Trap 3: "Single-table design is always better than multi-table design in DynamoDB."

**What most people say:**
"Yes — single-table design is the DynamoDB best practice and should always be used."

**Correct answer:**
Single-table design is a *pattern*, not a universal rule. It has real trade-offs:

**Advantages**:
- Single read/write operation to get related entities (no round trips)
- No need for cross-table consistency management
- All data in one hot cache partition pattern

**Disadvantages**:
- Visibility and debugging: `GetItem` returns opaque generic keys (`PK`, `SK`, `GSI1PK`)
  that require documentation to interpret
- Capacity sharing: if one entity type has a burst, it competes with all other entity types
  for the same table's throughput
- DynamoDB Streams: a single stream mixes all entity types, making Lambda trigger logic complex
- Migration: moving to single-table from multi-table is complex; moving back is even harder
- Team friction: multiple teams sharing one table creates governance and deployment conflicts
- CloudFormation/CDK: a single table with many GSIs is harder to manage as infrastructure

**When multi-table is appropriate**:
- Different teams own different entity types
- Different entities have radically different capacity requirements
- Entities have different Stream consumers
- The entities have no natural access patterns that combine them

The real DynamoDB best practice is: design your access patterns first, then choose the
table topology that best serves those patterns.

---

## Trap 4: "DynamoDB transactions (TransactWriteItems) don't cost anything extra."

**What most people say:**
"Transactions are just grouped writes — same cost as individual writes."

**Correct answer:**
DynamoDB transactions cost **2x** the normal read/write capacity units. Each item in a
`TransactWriteItems` call consumes 2 WCU instead of 1. Each item in `TransactGetItems`
consumes 2 RCU instead of 1 (for strongly consistent reads).

This is because DynamoDB uses a two-phase commit (2PC) protocol under the hood. Each item
is prepared (phase 1: read + lock) and then committed (phase 2: write). Both phases consume
capacity.

Additionally:
- Max 25 items per `TransactWriteItems` call (as of 2024)
- Max 100 items per `TransactGetItems`
- Transactions involving items on the same partition key as another concurrent transaction
  will experience contention (one will fail with TransactionConflictException)

```python
import boto3
from botocore.exceptions import ClientError

dynamodb = boto3.client('dynamodb')

def transfer_funds(from_account: str, to_account: str, amount: int):
    """
    Atomic fund transfer using DynamoDB transactions.
    Costs: 2 WCU per item × 2 items = 4 WCU total (vs 2 WCU for two individual writes)
    """
    try:
        response = dynamodb.transact_write_items(
            TransactItems=[
                {
                    'Update': {
                        'TableName': 'Accounts',
                        'Key': {'PK': {'S': f'ACCOUNT#{from_account}'}},
                        'UpdateExpression': 'SET balance = balance - :amount',
                        'ConditionExpression': 'balance >= :amount',  # Prevent overdraft
                        'ExpressionAttributeValues': {
                            ':amount': {'N': str(amount)}
                        }
                    }
                },
                {
                    'Update': {
                        'TableName': 'Accounts',
                        'Key': {'PK': {'S': f'ACCOUNT#{to_account}'}},
                        'UpdateExpression': 'SET balance = balance + :amount',
                        'ExpressionAttributeValues': {
                            ':amount': {'N': str(amount)}
                        }
                    }
                },
            ]
        )
        return True
    except ClientError as e:
        if e.response['Error']['Code'] == 'TransactionCanceledException':
            reasons = e.response['CancellationReasons']
            if any(r['Code'] == 'ConditionalCheckFailed' for r in reasons):
                raise ValueError("Insufficient funds")
        raise
```

---

## Trap 5: "You can sort results from a DynamoDB Query in any order."

**What most people say:**
"Yes, use the ScanIndexForward parameter to control sort order."

**Correct answer:**
DynamoDB can only return results in the *natural sort order of the sort key* (ascending or
descending via `ScanIndexForward`). You CANNOT sort by any attribute other than the sort key
of the index being queried. If you need results sorted by a different attribute (e.g., sort
orders by `total_amount` rather than `created_at`), you have two options:

1. Create a GSI where the sort key is the attribute you want to sort by
2. Sort the results in your application (only acceptable for small result sets)

DynamoDB's sort key ordering is *byte-order* for strings (lexicographic), which means
numeric strings sort incorrectly (`"9" > "10"` lexicographically). For numeric date sorting
with string sort keys, use a padded format: `"2024-01-09"` sorts correctly because ISO
date format is naturally lexicographic.

```python
# Sort orders by creation date (correct — ISO format sorts lexicographically)
response = table.query(
    KeyConditionExpression='PK = :pk AND begins_with(SK, :prefix)',
    ExpressionAttributeValues={
        ':pk': 'CUST#alice',
        ':prefix': 'ORDER#',
    },
    ScanIndexForward=False,  # Descending sort (newest first)
)

# WRONG: trying to sort by total_amount (not the sort key)
# DynamoDB does NOT support this:
# response = table.query(..., OrderBy='total_amount')  # does not exist

# CORRECT: Create a GSI with total_amount as sort key
# GSI PK = customer_id, GSI SK = total_amount (stored as a padded string or number)
# Then query the GSI with ScanIndexForward=False for highest amounts first
```

---

## Trap 6: "Filter expressions in DynamoDB reduce the number of items read and the RCU consumed."

**What most people say:**
"Yes, filter expressions limit results to matching items and reduce capacity usage."

**Correct answer:**
Filter expressions are applied *after* DynamoDB reads the items and *after* capacity is
consumed. They do NOT reduce RCU consumption.

DynamoDB's Query/Scan operation:
1. Reads items matching the key condition (up to `Limit` items or 1MB)
2. Consumes RCUs for ALL read items (whether they pass the filter or not)
3. Applies the filter expression to the read items
4. Returns only the matching items to the client

This means a filter expression with a 1% match rate reads 100x more data than it returns,
wasting 99% of your RCUs. If you're paginating with a `Limit` of 100 and the filter keeps
only 1 item, you must make ~100 requests to get 100 matching items.

The correct approach: redesign the key schema (sort key, GSI) so the query itself filters
the data, not a post-read filter expression.

```python
# BAD: filter expression on a non-key attribute (reads all items, discards most)
response = table.query(
    KeyConditionExpression='PK = :pk',
    FilterExpression='#s = :status',  # applied AFTER reading ALL items for this PK
    ExpressionAttributeNames={'#s': 'status'},
    ExpressionAttributeValues={':pk': 'CUST#alice', ':status': 'PENDING'},
    Limit=100,  # reads 100 items, keeps only those with status=PENDING
)
# If only 2 of 100 items are PENDING, we burned 100 RCUs to get 2 items

# GOOD: use a GSI with status as part of the key
# GSI PK = customer_id, GSI SK = status#created_at
response = table.query(
    IndexName='CustomerStatusIndex',
    KeyConditionExpression='customer_id = :cid AND begins_with(status_created, :status)',
    ExpressionAttributeValues={
        ':cid': 'CUST#alice',
        ':status': 'PENDING#',  # reads ONLY pending orders
    }
)
# Reads only matching items → much lower RCU consumption
```

---

## Trap 7: "DynamoDB TTL deletes items exactly when the TTL expires."

**What most people say:**
"Items are deleted immediately when the TTL timestamp passes."

**Correct answer:**
DynamoDB TTL deletion is a **background process** that may lag by up to **48 hours** after
the TTL timestamp. AWS does not guarantee immediate deletion — it guarantees eventual deletion
"typically within a few days."

This has critical implications:
1. **Your application must filter out expired items** if you read before the background
   deletion runs. An item with `ttl = yesterday` may still be returned by a `GetItem`.
2. **Billing**: You are NOT charged for TTL deletions (they are free), but you ARE charged
   for storage until the item is physically deleted.
3. **Streams**: TTL deletions appear in DynamoDB Streams with a special marker
   (`userIdentity.type = "Service"` and `userIdentity.principalId = "dynamodb.amazonaws.com"`).
   Your Stream consumer should handle these differently from user-initiated deletes.

```python
from datetime import datetime, timezone
import time

def put_with_ttl(table, item: dict, ttl_seconds: int):
    """Store item with TTL — adds 'expires_at' epoch timestamp attribute."""
    expires_at = int(time.time()) + ttl_seconds
    item['expires_at'] = expires_at  # Must be a Number attribute for TTL to work
    table.put_item(Item=item)

def get_item_if_not_expired(table, key: dict) -> dict | None:
    """Get item but filter out TTL-expired items (TTL deletion may not have run yet)."""
    response = table.get_item(Key=key)
    item = response.get('Item')

    if item is None:
        return None

    # Check TTL manually in case background deletion hasn't run
    expires_at = item.get('expires_at')
    if expires_at and expires_at < int(time.time()):
        return None  # Expired but not yet deleted by DynamoDB

    return item

# Enable TTL on the table (one-time setup)
dynamodb = boto3.client('dynamodb')
dynamodb.update_time_to_live(
    TableName='Sessions',
    TimeToLiveSpecification={
        'Enabled': True,
        'AttributeName': 'expires_at'  # Must be a Number (epoch seconds)
    }
)
```

---

## Trap 8: "You can store arbitrarily large lists and maps in DynamoDB attribute values."

**What most people say:**
"DynamoDB supports List and Map types so you can nest as much data as you want."

**Correct answer:**
DynamoDB has a hard limit of **400KB per item** (all attributes combined). This applies to
the entire item including all nested List and Map attributes. Additionally:

1. **Nesting depth**: Maps and Lists can be nested up to 32 levels deep.
2. **List/Map size**: No explicit limit per list/map, but the total item size is 400KB.
3. **Atomic updates to lists**: DynamoDB's `list_append` in UpdateExpression appends to the
   end of a list atomically. But there is no way to atomically update a specific index in a
   list *conditionally*. For ordered collections where you need random-access updates, a
   `Map` with string keys is better.
4. **Reading cost**: DynamoDB charges for the *full item size* even if you project only a
   few attributes. Projection expressions reduce *data transfer* but not RCU consumption
   (unless using GSI with `INCLUDE` projection).

```python
# Design consideration: don't embed large arrays in DynamoDB items
# BAD: 1000 order line items in a single item
order = {
    'PK': 'ORDER#001',
    'line_items': [{'product': 'SKU-001', 'qty': 1}, ...] * 1000  # Could exceed 400KB!
}

# GOOD: Each line item is a separate item with sort key
# ORDER#001 + LINE#001
# ORDER#001 + LINE#002
# ...
# ORDER#001 + LINE#1000
# Retrieve all: query WHERE PK = 'ORDER#001' AND begins_with(SK, 'LINE#')
# This also enables individual line item updates without rewriting the whole order
```

---

## Trap 9: "DynamoDB Scan is always bad — never use it in production."

**What most people say:**
"Yes, Scan reads the entire table and is always an anti-pattern."

**Correct answer:**
Scan is appropriate in specific, controlled scenarios:

1. **Bulk export / ETL**: Exporting all data to S3, Redshift, or another system. In this
   case, use parallel scan with multiple workers, each scanning a segment.

2. **One-time migrations**: Updating all items to add a new attribute (but consider DAX
   caching impact and capacity planning).

3. **Small tables** (< 100 items): A table used for configuration or feature flags where
   a full scan is cheaper than maintaining an index.

The key safeguards when using Scan:
- Use `Limit` to control page size and prevent single requests from consuming massive capacity
- Use `ExclusiveStartKey` for pagination
- Use parallel scan for large tables (divide into N segments)
- Run during off-peak hours to avoid throttling production reads
- Set a rate limit in your scan loop

```python
import time

def parallel_scan_with_rate_limit(
    table,
    total_segments: int = 4,
    segment: int = 0,
    items_per_second: int = 100,
):
    """Perform a rate-limited parallel scan segment."""
    exclusive_start_key = None
    batch_size = 25

    while True:
        kwargs = {
            'Limit': batch_size,
            'TotalSegments': total_segments,
            'Segment': segment,
        }
        if exclusive_start_key:
            kwargs['ExclusiveStartKey'] = exclusive_start_key

        response = table.scan(**kwargs)
        items = response.get('Items', [])

        yield from items

        # Rate limiting: don't overwhelm the table
        time.sleep(batch_size / items_per_second)

        exclusive_start_key = response.get('LastEvaluatedKey')
        if not exclusive_start_key:
            break
```

---

## Trap 10: "DynamoDB strongly consistent reads are always better than eventually consistent reads."

**What most people say:**
"Yes, strongly consistent reads are more correct so I'll use them everywhere."

**Correct answer:**
Strongly consistent reads cost **2x the RCUs** of eventually consistent reads and have
**higher latency** (they require contacting the leader node for the partition, not just
any replica). Using strongly consistent reads everywhere is a common and expensive mistake.

Eventually consistent reads in DynamoDB typically lag by under 100ms. For most application
data (user profiles, product catalogues, session data), this is imperceptible to users and
2x cheaper.

Use strongly consistent reads ONLY when:
- You just wrote data and must immediately verify it (post-write verification)
- Financial or inventory data where stale reads cause business errors
- Read-modify-write patterns where stale reads can cause lost updates (use conditional
  writes as the better alternative)

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Products')

# Eventually consistent read — default, 1 RCU per 4KB
response = table.get_item(
    Key={'PK': 'PRODUCT#SKU-001'},
    # ConsistentRead is False by default
)

# Strongly consistent read — 2x RCU, use only when necessary
response = table.get_item(
    Key={'PK': 'PRODUCT#SKU-001'},
    ConsistentRead=True,  # 2 RCU per 4KB, can't use on GSI
)

# BETTER pattern for post-write read: use conditional write + return values
response = table.update_item(
    Key={'PK': 'PRODUCT#SKU-001'},
    UpdateExpression='SET stock = stock - :qty',
    ConditionExpression='stock >= :qty',
    ExpressionAttributeValues={':qty': 1},
    ReturnValues='ALL_NEW',  # Returns the updated item — no separate read needed
)
updated_item = response.get('Attributes')
```
