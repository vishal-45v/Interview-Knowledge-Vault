# Chapter 04 — DynamoDB: Structured Answers

Complete, interview-ready answers with working Python and DynamoDB JSON examples.

---

## Q1: Design a DynamoDB single-table schema for an e-commerce platform.

**Answer:**

**Access patterns identified first:**
1. Get customer by ID
2. Get all orders for a customer (sorted by date, newest first)
3. Get specific order by order ID
4. Get all items in an order
5. Get all orders with status PENDING (admin)
6. Get all products in a category

```python
# ─── SINGLE TABLE DESIGN ─────────────────────────────────────────────────────
# Table name: ECommerceTable
# Base table PK: PK (String), SK (String)
# GSI1: GSI1PK, GSI1SK
# GSI2: GSI2PK, GSI2SK

# Entity: CUSTOMER
customer_item = {
    "PK":          "CUST#alice",          # Partition key
    "SK":          "PROFILE",             # Sort key
    "GSI1PK":      "CUSTOMER",            # GSI1: all customers scan
    "GSI1SK":      "alice",               # Sort by username
    "entity_type": "Customer",
    "email":       "alice@example.com",
    "name":        "Alice Smith",
    "created_at":  "2024-01-15T10:00:00Z",
}

# Query pattern 1: GetItem(PK="CUST#alice", SK="PROFILE")

# Entity: ORDER
order_item = {
    "PK":           "CUST#alice",              # Same partition as customer
    "SK":           "ORDER#2024-01-15#001",    # Sort key: date#id for range queries
    "GSI1PK":       "ORDER#2024-01-15",        # GSI1: orders by date
    "GSI1SK":       "CUST#alice",              # Sort by customer
    "GSI2PK":       "STATUS#PENDING",          # GSI2: orders by status
    "GSI2SK":       "2024-01-15T10:00:00Z",    # Sort by timestamp
    "entity_type":  "Order",
    "order_id":     "ORDER#001",
    "status":       "PENDING",
    "total_amount": 129.99,
    "created_at":   "2024-01-15T10:00:00Z",
}

# Query pattern 2: Query(PK="CUST#alice", begins_with(SK, "ORDER#"), ScanIndexForward=False)
# Query pattern 3: GetItem(PK="ORDER#001", SK="METADATA")
# Query pattern 5: Query GSI2(GSI2PK="STATUS#PENDING")

# Entity: ORDER LINE ITEM
line_item = {
    "PK":           "ORDER#001",            # Same partition as order
    "SK":           "ITEM#SKU-001",         # Sort key: item in order
    "entity_type":  "OrderItem",
    "product_id":   "SKU-001",
    "product_name": "Wireless Headphones",
    "quantity":     1,
    "unit_price":   129.99,
}

# Query pattern 4: Query(PK="ORDER#001", begins_with(SK, "ITEM#"))

# Entity: PRODUCT
product_item = {
    "PK":           "PRODUCT#SKU-001",
    "SK":           "METADATA",
    "GSI1PK":       "CATEGORY#electronics",   # GSI1: products by category
    "GSI1SK":       "PRODUCT#SKU-001",
    "entity_type":  "Product",
    "name":         "Wireless Headphones",
    "price":        129.99,
    "stock":        250,
    "category":     "electronics",
}

# Query pattern 6: Query GSI1(GSI1PK="CATEGORY#electronics")
```

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr
from decimal import Decimal
import uuid
from datetime import datetime, date

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('ECommerceTable')

def get_customer(customer_id: str) -> dict:
    """Pattern 1: Get customer by ID."""
    resp = table.get_item(Key={'PK': f'CUST#{customer_id}', 'SK': 'PROFILE'})
    return resp.get('Item')

def get_customer_orders(customer_id: str, limit: int = 20, last_key=None) -> dict:
    """Pattern 2: Get customer orders, newest first, paginated."""
    kwargs = {
        'KeyConditionExpression': Key('PK').eq(f'CUST#{customer_id}') &
                                  Key('SK').begins_with('ORDER#'),
        'ScanIndexForward': False,
        'Limit': limit,
    }
    if last_key:
        kwargs['ExclusiveStartKey'] = last_key

    resp = table.query(**kwargs)
    return {
        'orders': resp['Items'],
        'next_key': resp.get('LastEvaluatedKey'),
    }

def get_order_items(order_id: str) -> list:
    """Pattern 4: Get all items in an order."""
    resp = table.query(
        KeyConditionExpression=Key('PK').eq(f'ORDER#{order_id}') &
                               Key('SK').begins_with('ITEM#'),
    )
    return resp['Items']

def get_pending_orders(limit: int = 50, last_key=None) -> dict:
    """Pattern 5: Admin — get all PENDING orders via GSI2."""
    kwargs = {
        'IndexName': 'GSI2',
        'KeyConditionExpression': Key('GSI2PK').eq('STATUS#PENDING'),
        'ScanIndexForward': False,  # Newest pending first
        'Limit': limit,
    }
    if last_key:
        kwargs['ExclusiveStartKey'] = last_key

    resp = table.query(**kwargs)
    return {'orders': resp['Items'], 'next_key': resp.get('LastEvaluatedKey')}

def create_order(customer_id: str, items: list) -> str:
    """Create an order with line items as a batch write (not atomic — use transaction for atomic)."""
    order_id   = str(uuid.uuid4())[:8]
    order_date = date.today().isoformat()
    created_at = datetime.utcnow().isoformat() + 'Z'
    total      = sum(i['unit_price'] * i['quantity'] for i in items)

    with table.batch_writer() as batch:
        # Order item
        batch.put_item(Item={
            'PK':          f'CUST#{customer_id}',
            'SK':          f'ORDER#{order_date}#{order_id}',
            'GSI2PK':      'STATUS#PENDING',
            'GSI2SK':      created_at,
            'entity_type': 'Order',
            'order_id':    order_id,
            'status':      'PENDING',
            'total_amount': Decimal(str(total)),
            'created_at':   created_at,
        })
        # Line items
        for item in items:
            batch.put_item(Item={
                'PK':           f'ORDER#{order_id}',
                'SK':           f'ITEM#{item["product_id"]}',
                'entity_type':  'OrderItem',
                'product_id':   item['product_id'],
                'quantity':     item['quantity'],
                'unit_price':   Decimal(str(item['unit_price'])),
            })

    return order_id
```

---

## Q2: Explain RCU/WCU math with worked examples.

**Answer:**

```
READ CAPACITY UNITS (RCU):
- 1 RCU = 1 strongly consistent read of up to 4KB per second
- 1 RCU = 2 eventually consistent reads of up to 4KB per second
- Formula: RCU needed = ⌈item_size_KB / 4⌉ × (1 for strong, 0.5 for eventual)

WRITE CAPACITY UNITS (WCU):
- 1 WCU = 1 write of up to 1KB per second
- Formula: WCU needed = ⌈item_size_KB / 1⌉ per write

TRANSACTIONAL:
- TransactGetItems: 2 RCU per item (regardless of consistency)
- TransactWriteItems: 2 WCU per item
```

```python
import math

def calculate_rcu(
    item_size_kb: float,
    reads_per_second: int,
    consistent: bool = False
) -> int:
    """Calculate required RCUs."""
    rcu_per_read = math.ceil(item_size_kb / 4)
    if not consistent:
        rcu_per_read = rcu_per_read / 2  # eventually consistent = 0.5x
    return math.ceil(rcu_per_read * reads_per_second)

def calculate_wcu(item_size_kb: float, writes_per_second: int) -> int:
    """Calculate required WCUs."""
    wcu_per_write = math.ceil(item_size_kb / 1)
    return wcu_per_write * writes_per_second

# WORKED EXAMPLES:
# Q: 10,000 reads/sec of 6KB items, strongly consistent
rcu_strong = calculate_rcu(item_size_kb=6, reads_per_second=10_000, consistent=True)
print(f"Strongly consistent RCU: {rcu_strong}")
# item RCU = ceil(6/4) = 2
# total = 2 × 10,000 = 20,000 RCU

# Q: 10,000 reads/sec of 6KB items, eventually consistent (default for GSI)
rcu_eventual = calculate_rcu(item_size_kb=6, reads_per_second=10_000, consistent=False)
print(f"Eventually consistent RCU: {rcu_eventual}")
# item RCU = ceil(6/4) / 2 = 2 / 2 = 1
# total = 1 × 10,000 = 10,000 RCU

# Q: 5,000 writes/sec of 2.5KB items
wcu = calculate_wcu(item_size_kb=2.5, writes_per_second=5_000)
print(f"WCU needed: {wcu}")
# item WCU = ceil(2.5/1) = 3
# total = 3 × 5,000 = 15,000 WCU

# Monthly cost estimate (us-east-1, 2024 pricing):
rcu_monthly_cost = rcu_strong * 0.00013  # $0.00013 per RCU-hour → convert
wcu_monthly_cost = wcu * 0.00065

print(f"Provisioned RCU monthly: ${rcu_strong * 0.00013 * 730:.2f}")
print(f"Provisioned WCU monthly: ${wcu * 0.00065 * 730:.2f}")
# Note: actual pricing is per month for provisioned, per request for on-demand
```

---

## Q3: Implement write sharding for a hot partition key.

**Answer:**

```python
import boto3
import hashlib
import random
from decimal import Decimal
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('UserActivity')

NUM_SHARDS = 10  # Distribute across 10 shards

def get_shard_key(user_id: str, num_shards: int) -> int:
    """Deterministic shard for reads; random shard for writes."""
    # For deterministic shard: hash-based (same user always on same shard)
    return int(hashlib.md5(user_id.encode()).hexdigest(), 16) % num_shards

def write_user_event_sharded(user_id: str, event: dict):
    """
    Write sharding: distribute a hot user's writes across multiple partitions.
    Random shard on write — scatter-gather on read.
    Use when the same partition key gets > 1000 WCU/sec.
    """
    shard = random.randint(0, NUM_SHARDS - 1)
    sharded_pk = f"USER#{user_id}#SHARD#{shard}"

    table.put_item(Item={
        'PK':        sharded_pk,
        'SK':        f"EVENT#{datetime.utcnow().isoformat()}Z",
        'user_id':   user_id,
        'shard':     shard,
        'event_type': event['type'],
        'metadata':  event.get('metadata', {}),
    })

def read_user_events_sharded(user_id: str, limit_per_shard: int = 10) -> list:
    """
    Scatter-gather read: query all shards and merge results.
    This is the cost of write sharding — reads become more complex.
    """
    all_events = []
    for shard in range(NUM_SHARDS):
        sharded_pk = f"USER#{user_id}#SHARD#{shard}"
        resp = table.query(
            KeyConditionExpression='PK = :pk',
            ExpressionAttributeValues={':pk': sharded_pk},
            ScanIndexForward=False,
            Limit=limit_per_shard,
        )
        all_events.extend(resp['Items'])

    # Sort across shards by timestamp
    all_events.sort(key=lambda e: e['SK'], reverse=True)
    return all_events[:limit_per_shard]

# Alternative: Deterministic sharding (same user always on same shard)
# Advantage: reads are single-shard (no scatter-gather)
# Disadvantage: doesn't solve hot user problem if all their writes still go to 1 shard
def write_deterministic_shard(user_id: str, event: dict):
    """Deterministic sharding — same user always goes to same shard."""
    shard = get_shard_key(user_id, NUM_SHARDS)
    sharded_pk = f"USER#{user_id}#SHARD#{shard:02d}"
    table.put_item(Item={
        'PK': sharded_pk,
        'SK': f"EVENT#{datetime.utcnow().isoformat()}Z",
        **event,
    })

def read_deterministic_shard(user_id: str) -> list:
    """Read is simple — always go to the deterministic shard."""
    shard = get_shard_key(user_id, NUM_SHARDS)
    sharded_pk = f"USER#{user_id}#SHARD#{shard:02d}"
    resp = table.query(
        KeyConditionExpression='PK = :pk',
        ExpressionAttributeValues={':pk': sharded_pk},
        ScanIndexForward=False,
        Limit=100,
    )
    return resp['Items']
```

---

## Q4: Implement optimistic locking with DynamoDB condition expressions.

**Answer:**

```python
import boto3
from botocore.exceptions import ClientError
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Inventory')

def update_inventory_with_lock(product_id: str, quantity_delta: int, retries: int = 3) -> bool:
    """
    Decrement inventory with optimistic locking using a version number.
    Prevents lost updates when multiple processes update the same item concurrently.
    """
    for attempt in range(retries):
        # Step 1: Read current state
        response = table.get_item(
            Key={'PK': f'PRODUCT#{product_id}'},
            ConsistentRead=True,  # Must use consistent read for OCC
        )
        item = response.get('Item')
        if not item:
            raise ValueError(f"Product {product_id} not found")

        current_version = item.get('version', 0)
        current_quantity = int(item['quantity'])

        if current_quantity + quantity_delta < 0:
            raise ValueError("Insufficient inventory")

        # Step 2: Conditional write — only succeeds if version hasn't changed
        try:
            table.update_item(
                Key={'PK': f'PRODUCT#{product_id}'},
                UpdateExpression='SET quantity = :new_qty, version = :new_version',
                ConditionExpression='version = :current_version OR attribute_not_exists(version)',
                ExpressionAttributeValues={
                    ':new_qty':          Decimal(str(current_quantity + quantity_delta)),
                    ':new_version':      Decimal(str(current_version + 1)),
                    ':current_version':  Decimal(str(current_version)),
                },
            )
            return True  # Success

        except ClientError as e:
            if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
                # Another process updated the item — retry
                if attempt < retries - 1:
                    continue
                raise RuntimeError(f"Failed after {retries} retries — high contention")
            raise

    return False

def reserve_seat(event_id: str, seat_id: str, user_id: str) -> bool:
    """
    Seat reservation using IF NOT EXISTS condition — prevents double-booking.
    The condition fails if the seat is already reserved (seat_reserved attribute exists).
    """
    try:
        table.update_item(
            Key={'PK': f'EVENT#{event_id}', 'SK': f'SEAT#{seat_id}'},
            UpdateExpression='SET reserved_by = :user, reserved_at = :ts',
            ConditionExpression='attribute_not_exists(reserved_by)',
            ExpressionAttributeValues={
                ':user': user_id,
                ':ts':   datetime.utcnow().isoformat() + 'Z',
            },
        )
        return True  # Reservation successful

    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            return False  # Seat already reserved
        raise
```

---

## Q5: Explain DynamoDB Streams and implement a change data capture pattern.

**Answer:**

```python
import boto3
import json
from decimal import Decimal

# Enable DynamoDB Streams (in CloudFormation/CDK, or via boto3)
dynamodb = boto3.client('dynamodb')
dynamodb.update_table(
    TableName='Orders',
    StreamSpecification={
        'StreamEnabled': True,
        'StreamViewType': 'NEW_AND_OLD_IMAGES',  # Best for CDC: see before/after state
        # Other options:
        # 'KEYS_ONLY'        — only changed keys (cheapest, least info)
        # 'NEW_IMAGE'        — new state only (no delete detection)
        # 'OLD_IMAGE'        — old state only (good for delete recovery)
        # 'NEW_AND_OLD_IMAGES' — full diff (best for CDC, more expensive)
    }
)

# Lambda handler for DynamoDB Streams CDC
def lambda_handler(event, context):
    """
    Process DynamoDB Stream events and sync to Elasticsearch.
    Stream records are delivered in order per partition key.
    Lambda invokes this with at-least-once delivery.
    """
    for record in event['Records']:
        event_id    = record['eventID']          # Unique ID for deduplication
        event_name  = record['eventName']        # INSERT | MODIFY | REMOVE
        new_image   = record.get('dynamodb', {}).get('NewImage', {})
        old_image   = record.get('dynamodb', {}).get('OldImage', {})

        # Skip TTL deletions (automated — not user-initiated)
        user_identity = record.get('userIdentity', {})
        if (user_identity.get('type') == 'Service' and
                user_identity.get('principalId') == 'dynamodb.amazonaws.com'):
            print(f"Skipping TTL deletion: {event_id}")
            continue

        # Deserialize DynamoDB's typed JSON format
        new_item = deserialize_dynamodb_item(new_image) if new_image else None
        old_item = deserialize_dynamodb_item(old_image) if old_image else None

        # Route to appropriate handler
        if event_name == 'INSERT':
            sync_to_elasticsearch('create', new_item)
        elif event_name == 'MODIFY':
            sync_to_elasticsearch('update', new_item)
        elif event_name == 'REMOVE':
            sync_to_elasticsearch('delete', old_item)

    return {'statusCode': 200}

def deserialize_dynamodb_item(dynamo_item: dict) -> dict:
    """Convert DynamoDB typed JSON to plain Python dict."""
    from boto3.dynamodb.types import TypeDeserializer
    deserializer = TypeDeserializer()
    return {k: deserializer.deserialize(v) for k, v in dynamo_item.items()}

def sync_to_elasticsearch(action: str, item: dict):
    """Sync item to Elasticsearch (simplified)."""
    import urllib.request
    import urllib.parse

    index = item.get('entity_type', 'unknown').lower()
    doc_id = item.get('PK', '')

    if action == 'delete':
        print(f"Delete from ES: index={index}, id={doc_id}")
    else:
        print(f"Upsert to ES: action={action}, index={index}, id={doc_id}")
        # In production: use elasticsearch-py or opensearch-py client
```

**Stream record anatomy:**

```json
{
    "eventID": "1",
    "eventVersion": "1.0",
    "dynamodb": {
        "Keys": {
            "PK": {"S": "CUST#alice"},
            "SK": {"S": "ORDER#2024-01-15#001"}
        },
        "NewImage": {
            "PK": {"S": "CUST#alice"},
            "SK": {"S": "ORDER#2024-01-15#001"},
            "status": {"S": "SHIPPED"},
            "total_amount": {"N": "129.99"}
        },
        "OldImage": {
            "PK": {"S": "CUST#alice"},
            "SK": {"S": "ORDER#2024-01-15#001"},
            "status": {"S": "PENDING"},
            "total_amount": {"N": "129.99"}
        },
        "StreamViewType": "NEW_AND_OLD_IMAGES",
        "SequenceNumber": "111",
        "SizeBytes": 100
    },
    "awsRegion": "us-east-1",
    "eventName": "MODIFY",
    "eventSourceARN": "arn:aws:dynamodb:us-east-1:123456789:table/Orders/stream/...",
    "eventSource": "aws:dynamodb"
}
```
