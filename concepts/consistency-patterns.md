# Consistency Patterns & CAP Theorem

## Overview

In distributed systems, **consistency** refers to how all nodes see the same data at the same time. This document explores different consistency models and the trade-offs between consistency, availability, and partition tolerance.

---

## CAP Theorem

**CAP Theorem** states that a distributed system can only guarantee **2 out of 3** properties:

### C - Consistency
All nodes see the same data at the same time. Every read receives the most recent write.

### A - Availability
Every request receives a response (success or failure), without guarantee that it contains the most recent write.

### P - Partition Tolerance
The system continues to operate despite network partitions (communication breakdowns between nodes).

---

## CAP Trade-offs

Since network partitions **will happen** in distributed systems, you must choose between:

### CP (Consistency + Partition Tolerance)
**Sacrifice availability** - System may reject requests to maintain consistency.

**Example:** Bank transactions
```
User A transfers $100 to User B
→ Lock both accounts until transaction completes
→ If network partition occurs, reject new requests
→ Ensures balance is always consistent
```

**Systems:** MongoDB, HBase, Redis (with consistency mode)

---

### AP (Availability + Partition Tolerance)
**Sacrifice consistency** - System always responds, but data may be stale.

**Example:** Social media feed
```
User posts a status update
→ Some users see it immediately
→ Others see it after a delay
→ Eventually all users see it (Eventual Consistency)
```

**Systems:** Cassandra, DynamoDB, Riak

---

### CA (Consistency + Availability)
**Not realistic in distributed systems** - Network partitions will happen.

Only possible in single-node systems (traditional RDBMS on one server).

---

## Consistency Models

### 1. Strong Consistency

**Every read gets the latest write** - All nodes see the same data immediately.

**How it works:**
```
1. Client writes data to primary node
2. Primary replicates to all replicas synchronously
3. Write acknowledged only after ALL replicas confirm
4. Any subsequent read gets the latest data
```

**Example:**
```python
# Write
db.write("key", "value_v2")  # Blocks until all replicas updated

# Read (from any node)
db.read("key")  # Always returns "value_v2"
```

**Pros:**
- ✅ Simple reasoning (no stale data)
- ✅ Correct for financial transactions

**Cons:**
- ❌ High latency (wait for all nodes)
- ❌ Lower availability (if node is down, write fails)

**Used in:** Banking systems, inventory management

---

### 2. Eventual Consistency

**Nodes may temporarily disagree, but eventually converge** to the same value.

**How it works:**
```
1. Client writes data to one node
2. Node acknowledges immediately
3. Data asynchronously replicates to other nodes
4. Eventually (after some time) all nodes have the same data
```

**Example:**
```python
# Write to Node A
db.write("key", "value_v2")  # Returns immediately

# Read from Node B (immediately after write)
db.read("key")  # Might return "value_v1" (stale)

# Read from Node B (after replication)
db.read("key")  # Returns "value_v2" (consistent)
```

**Pros:**
- ✅ High availability
- ✅ Low latency
- ✅ Partition tolerant

**Cons:**
- ❌ Stale reads possible
- ❌ Conflicts need resolution

**Used in:** DNS, caching, social media, Amazon DynamoDB

---

### 3. Causal Consistency

**Causally related operations are seen in order** - Non-causal operations can be inconsistent.

**Example:**
```
User A posts: "I'm getting married!"
User A posts: "Here's my wedding photo"

All users see posts in this order (causally related)

But User B's comment may appear at different times (not causal)
```

**Pros:**
- ✅ Better than eventual consistency
- ✅ Preserves causal relationships

**Cons:**
- ❌ More complex to implement

**Used in:** Collaborative editing, chat systems

---

### 4. Read-Your-Writes Consistency

**Users see their own writes immediately**, but other users may see stale data.

**Example:**
```
User A updates profile picture
→ User A sees new picture immediately
→ User B might see old picture briefly
```

**Implementation:**
```python
def read(user_id, key):
    # Check if this user recently wrote this key
    if recent_write_by_user(user_id, key):
        return read_from_master()  # Guaranteed fresh
    else:
        return read_from_replica()  # Might be stale
```

**Used in:** User profiles, settings

---

### 5. Monotonic Reads

**If a user reads a value, subsequent reads never return older values.**

**Example:**
```
User reads comment count: 10
User refreshes page: 15  ✅
User refreshes again: 12  ❌ (Violation - went backward!)
```

**Implementation:**
- Route user's requests to the same replica (sticky sessions)
- Use versioning/timestamps

**Used in:** Analytics dashboards, comment counts

---

## Consistency Patterns in Practice

### 1. Write-Ahead Log (WAL)

**Durability** - Writes are logged before applied.

**Flow:**
```
1. Write operation arrives
2. Append to log (on disk)
3. Apply to data structure (in memory)
4. Acknowledge to client
```

**Benefits:**
- Crash recovery (replay log)
- Replication (send log to replicas)

**Used in:** PostgreSQL, Kafka, MySQL

---

### 2. Quorum Reads/Writes

**Majority voting** to ensure consistency.

**Formula:**
```
W + R > N

W = Write quorum (nodes that must acknowledge write)
R = Read quorum (nodes that must respond to read)
N = Total replicas
```

**Example (N=5, W=3, R=3):**
```
Write: Must succeed on 3 out of 5 nodes
Read: Must read from 3 out of 5 nodes
→ At least 1 node has latest data (overlap)
```

**Tuning:**
- **Strong consistency:** W=3, R=3 (slower)
- **Fast writes:** W=1, R=5 (read all nodes)
- **Fast reads:** W=5, R=1 (write to all nodes)

**Used in:** Cassandra, DynamoDB, Riak

---

### 3. Two-Phase Commit (2PC)

**Atomic distributed transactions** - All nodes commit or all rollback.

**Phases:**

**Phase 1: Prepare**
```
Coordinator: "Are you ready to commit?"
Node A: "Yes, ready"
Node B: "Yes, ready"
Node C: "No, error"
```

**Phase 2: Commit/Abort**
```
Coordinator: "Abort" (because Node C said no)
All nodes: Rollback changes
```

**Pros:**
- ✅ Strong consistency
- ✅ ACID transactions

**Cons:**
- ❌ Blocking (nodes wait for coordinator)
- ❌ Coordinator is single point of failure

**Used in:** Distributed databases (MySQL Cluster, PostgreSQL with foreign tables)

---

### 4. Saga Pattern

**Long-running transactions** as a sequence of smaller transactions.

**Example: E-commerce Order**

**Success Flow:**
```
1. Reserve inventory  ✅
2. Charge payment     ✅
3. Ship order         ✅
```

**Failure Flow (Compensating Transactions):**
```
1. Reserve inventory  ✅
2. Charge payment     ❌ (Card declined)
3. Undo inventory reservation  ✅ (Compensate)
```

**Implementation:**
```python
def process_order(order_id):
    try:
        inventory_id = reserve_inventory(order_id)
        try:
            payment_id = charge_payment(order_id)
            try:
                ship_order(order_id)
            except ShippingError:
                refund_payment(payment_id)
                release_inventory(inventory_id)
        except PaymentError:
            release_inventory(inventory_id)
    except InventoryError:
        cancel_order(order_id)
```

**Pros:**
- ✅ Works across microservices
- ✅ No blocking

**Cons:**
- ❌ Complex error handling
- ❌ Eventual consistency

**Used in:** Microservices, distributed workflows

---

### 5. Conflict Resolution

**When two nodes update the same data** - How to resolve?

#### Last-Write-Wins (LWW)
```
Node A writes "value_A" at timestamp 100
Node B writes "value_B" at timestamp 105

Result: "value_B" wins (latest timestamp)
```

**Pros:** Simple
**Cons:** Data loss possible

---

#### Vector Clocks

**Track causality** to detect concurrent writes.

**Example:**
```
Initial: key = "value" [A:0, B:0, C:0]

Node A writes: "value_A" [A:1, B:0, C:0]
Node B writes: "value_B" [A:0, B:1, C:0]

Conflict detected! (neither happened-before the other)

Resolution: Application decides (merge, pick one, etc.)
```

**Used in:** Riak, Dynamo

---

#### CRDTs (Conflict-Free Replicated Data Types)

**Data structures that automatically resolve conflicts.**

**Example: Counter CRDT**
```
Node A: increment (counter = 1)
Node B: increment (counter = 1)

Merge: Add both → counter = 2  ✅
```

**Types:**
- **G-Counter** - Grow-only counter
- **PN-Counter** - Increment/decrement counter
- **LWW-Register** - Last-write-wins register
- **OR-Set** - Observed-remove set

**Used in:** Collaborative editing (Google Docs), Redis

---

## Consistency in Different Systems

### Relational Databases (ACID)

**Strong consistency** with transactions.

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**All or nothing** - Both updates succeed or both rollback.

---

### NoSQL Databases (BASE)

**Basically Available, Soft state, Eventual consistency**

**Cassandra Example:**
```python
# Write with consistency level
session.execute(query, consistency_level=ConsistencyLevel.QUORUM)

# Read with consistency level
session.execute(query, consistency_level=ConsistencyLevel.ONE)
```

**Consistency Levels:**
- `ONE` - Fast, might be stale
- `QUORUM` - Majority, strong consistency
- `ALL` - Slowest, strongest consistency

---

### Caching Layers

**Eventual consistency** - Cache may be stale.

**Cache Invalidation Strategies:**

#### Time-based (TTL)
```python
cache.set("user:123", user_data, ttl=3600)  # Expires in 1 hour
```

#### Event-based
```python
def update_user(user_id, data):
    db.update(user_id, data)
    cache.delete(f"user:{user_id}")  # Invalidate cache
```

#### Write-through
```python
def update_user(user_id, data):
    cache.set(f"user:{user_id}", data)  # Update cache
    db.update(user_id, data)             # Update DB
```

See: [Caching Strategies](caching-strategies.md)

---

## Choosing a Consistency Model

| Use Case | Consistency Model | Why |
|----------|------------------|-----|
| **Banking** | Strong | Money must be accurate |
| **Social media feed** | Eventual | Stale posts are okay |
| **User profile** | Read-your-writes | User sees own changes |
| **Analytics dashboard** | Monotonic reads | Counts don't go backward |
| **Collaborative editing** | Causal | Preserve operation order |
| **Inventory system** | Strong | Avoid overselling |
| **DNS** | Eventual | Updates can propagate slowly |

---

## Real-World Examples

### URL Shortener

**Eventual consistency** for analytics (click counts).

```
User clicks short URL
→ Redirect immediately (don't wait for count update)
→ Increment counter asynchronously
→ Analytics dashboard shows count with slight delay
```

**Strong consistency** for URL lookup (mapping short → original).

See: [URL Shortener](../url-shorterner.md)

---

### Video Streaming Platform

**Eventual consistency** for video metadata (views, likes).

**Strong consistency** for user authentication.

```
User likes video
→ UI shows like immediately (optimistic update)
→ Background job updates database
→ If it fails, revert UI change
```

See: [Video Streaming Platform](../video-streaming-platform.md)

---

## Monitoring Consistency

**Key Metrics:**
- **Replication lag** - Time delay between write and replication
- **Conflict rate** - Frequency of concurrent writes
- **Stale read percentage** - How often reads return old data

**Alerts:**
- Replication lag > threshold
- High conflict rate (indicates hot data)

---

## Summary

✅ **CAP Theorem** - Choose 2: Consistency, Availability, Partition Tolerance
✅ **CP systems** - Sacrifice availability for consistency (banking)
✅ **AP systems** - Sacrifice consistency for availability (social media)
✅ **Strong consistency** - All nodes agree (slow, guaranteed correct)
✅ **Eventual consistency** - Nodes eventually agree (fast, might be stale)
✅ **Quorum** - Majority voting for consistency
✅ **2PC** - Distributed transactions (strong but blocking)
✅ **Saga** - Long-running transactions (eventual, with compensation)

**Golden Rule:** Choose the weakest consistency model that satisfies your requirements - stronger consistency = lower performance!
