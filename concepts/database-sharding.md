# Database Sharding

## Overview

**Sharding** is a database architecture pattern that **horizontally partitions data across multiple databases** (shards). Each shard contains a subset of the total data.

## Why Sharding?

When a single database can't handle the load, sharding distributes data across multiple servers.

### Problems with Single Database

❌ **Storage limits** - Single server can only hold so much data
❌ **Performance bottleneck** - CPU, memory, disk I/O limits
❌ **Single point of failure** - If the database goes down, everything stops

### Benefits of Sharding

✅ **Horizontal scalability** - Add more servers to handle more data
✅ **Better performance** - Distribute queries across multiple databases
✅ **Parallel processing** - Multiple queries run simultaneously
✅ **Reduced latency** - Smaller datasets = faster queries

---

## Sharding Strategies

### 1. Hash-Based Sharding (Key-Based)

Use a **hash function** on a key to determine which shard stores the data.

**Formula:**
```
Shard Number = hash(key) % N
```
Where `N` = number of shards

**Example:**
```python
def get_shard(user_id, num_shards=4):
    return hash(user_id) % num_shards

# user_id: 12345
# hash(12345) = 8291029
# 8291029 % 4 = 1
# → Stored in Shard 1
```

**Shard Distribution:**
```
Shard 0: user_ids where hash % 4 = 0
Shard 1: user_ids where hash % 4 = 1
Shard 2: user_ids where hash % 4 = 2
Shard 3: user_ids where hash % 4 = 3
```

**Pros:**
- ✅ Even data distribution
- ✅ Simple to implement
- ✅ Fast lookups

**Cons:**
- ❌ Adding/removing shards requires rehashing all data
- ❌ Range queries are difficult (data scattered across shards)

**Used in:** [URL Shortener](../url-shorterner.md#93-sharding-short_key-hash-mod-n-servers-for-distribution)

---

### 2. Range-Based Sharding

Partition data based on **value ranges**.

**Example: User IDs**
```
Shard 0: user_id 1 - 1,000,000
Shard 1: user_id 1,000,001 - 2,000,000
Shard 2: user_id 2,000,001 - 3,000,000
Shard 3: user_id 3,000,001 - 4,000,000
```

**Example: Time-Based**
```
Shard 0: Data from Jan 2024 - Mar 2024
Shard 1: Data from Apr 2024 - Jun 2024
Shard 2: Data from Jul 2024 - Sep 2024
```

**Pros:**
- ✅ Range queries are fast (data is together)
- ✅ Easy to add new shards (just extend range)

**Cons:**
- ❌ **Uneven distribution** (hotspots possible)
- ❌ Certain shards can become overloaded

**Example of Hotspot:**
If 80% of users have IDs in range 1-1M, Shard 0 gets 80% of traffic!

**Used in:** Time-series databases, analytics systems

---

### 3. Geographic Sharding (Location-Based)

Partition data by **geographic region**.

**Example:**
```
Shard US-East: Users in USA East Coast
Shard US-West: Users in USA West Coast
Shard EU: Users in Europe
Shard Asia: Users in Asia
```

**Pros:**
- ✅ **Low latency** (data stored near users)
- ✅ Compliance with data residency laws (GDPR)

**Cons:**
- ❌ Uneven distribution (some regions have more users)
- ❌ Cross-region queries are complex

**Used in:** Global applications (Facebook, WhatsApp)

---

### 4. Directory-Based Sharding (Lookup Table)

Use a **lookup table** (directory) to map keys to shards.

**Example Lookup Table:**
```
| Key (user_id) | Shard   |
|--------------|---------|
| 12345        | Shard 1 |
| 67890        | Shard 2 |
| 11111        | Shard 0 |
```

**Pros:**
- ✅ Flexible - can move data between shards easily
- ✅ No rehashing needed when adding shards

**Cons:**
- ❌ Lookup table becomes a bottleneck
- ❌ Extra read operation before every query

**Used in:** Systems requiring dynamic shard assignment

---

## Consistent Hashing

A **better alternative to hash-based sharding** that minimizes data movement when shards are added/removed.

### Problem with Simple Hashing

```python
# With 4 shards:
shard = hash(key) % 4

# Add 1 shard (now 5 total):
shard = hash(key) % 5

# Problem: ALL keys now map to different shards!
# Must re-distribute ALL data.
```

### Consistent Hashing Solution

Uses a **hash ring** where servers and keys are mapped to points on a circle.

**How it works:**
1. Hash servers onto a ring (0 to 2^32)
2. Hash keys onto the same ring
3. Each key goes to the **next server clockwise**

**Benefit:**
- Adding/removing a server only affects **adjacent keys**, not all keys
- Only ~1/N keys need to move (N = number of servers)

**Example:**
```
Ring positions (0-360°):
Server A: 90°
Server B: 180°
Server C: 270°

Key "user123" hashes to 100°
→ Goes to Server B (next clockwise at 180°)

Add Server D at 135°:
→ Only keys between 90°-135° move from B to D
→ Other keys stay in place!
```

**Used in:** Distributed caches (Redis Cluster), Cassandra, DynamoDB

**See also:** Consistent hashing for URL shortener load balancing

---

## Shard Key Selection

Choosing the **right shard key** is critical!

### Good Shard Keys

✅ **High cardinality** - Many unique values (user_id, email)
✅ **Even distribution** - No hotspots
✅ **Frequently queried** - Minimize cross-shard queries

**Examples:**
- `user_id` - Good for user-centric apps
- `tenant_id` - Good for multi-tenant SaaS
- `order_id` - Good for e-commerce

### Bad Shard Keys

❌ **Low cardinality** - Few unique values (country, status)
❌ **Monotonically increasing** - Sequential IDs create hotspots
❌ **Rarely queried** - Causes cross-shard queries

**Bad Example: Country as Shard Key**
```
Shard US: 80% of users
Shard India: 15% of users
Shard Other: 5% of users

→ Uneven load, US shard is overloaded!
```

---

## Challenges with Sharding

### 1. Cross-Shard Queries

**Problem:** Joining data across multiple shards is slow.

**Example:**
```sql
-- User profile in Shard 1
-- User orders in Shard 2

-- This query hits BOTH shards:
SELECT users.name, orders.total
FROM users
JOIN orders ON users.id = orders.user_id
```

**Solutions:**
- Denormalize data (duplicate user name in orders table)
- Use application-level joins (fetch from both, join in code)
- Avoid cross-shard queries by design

### 2. Distributed Transactions

**Problem:** Ensuring ACID across multiple shards is hard.

**Solutions:**
- **2-Phase Commit (2PC)** - Coordinate transaction across shards
- **Saga Pattern** - Break into smaller, independent transactions
- **Eventual consistency** - Accept temporary inconsistency

### 3. Rebalancing Shards

**Problem:** When adding/removing shards, data must move.

**Solutions:**
- **Consistent hashing** - Minimize data movement
- **Virtual shards** - Use more logical shards than physical servers
- **Background migration** - Move data gradually

---

## Sharding + Replication

Combine sharding (horizontal) with replication (copies) for high availability.

```
Shard 0: Primary + 2 Replicas
Shard 1: Primary + 2 Replicas
Shard 2: Primary + 2 Replicas
```

**Benefits:**
- Reads can go to replicas (load distribution)
- If primary fails, replica takes over (high availability)

---

## When to Shard?

### Shard When:
✅ Single database can't handle the load (CPU, memory, disk)
✅ Data size exceeds single server capacity
✅ Need to scale horizontally
✅ Geographic distribution required

### Don't Shard When:
❌ Data is small (< 1TB) - vertical scaling is simpler
❌ Complex queries are common - sharding makes this harder
❌ Application can't handle distributed data
❌ Team lacks distributed systems expertise

---

## Real-World Examples

### 1. URL Shortener
**Shard Key:** `short_key`
**Strategy:** Hash-based (`hash(short_key) % N`)
**Why:** Even distribution, fast lookups
**Challenge:** Avoid cross-shard analytics queries

See: [URL Shortener - Sharding](../url-shorterner.md#93-sharding-short_key-hash-mod-n-servers-for-distribution)

### 2. Instagram
**Shard Key:** `user_id` for user data, `media_id` for photos
**Strategy:** Consistent hashing
**Scale:** Thousands of shards

### 3. Twitter
**Shard Key:** `user_id` for tweets
**Strategy:** Range-based + temporal sharding
**Optimization:** Hot users get dedicated shards

---

## Partitioning vs Sharding

Often used interchangeably, but there's a difference:

| Feature | **Partitioning** | **Sharding** |
|---------|-----------------|-------------|
| **Scope** | Within a single database | Across multiple databases |
| **Purpose** | Organize data for performance | Scale beyond single server |
| **Example** | Table partitions in PostgreSQL | Separate database servers |

**Example Partitioning:**
```sql
-- PostgreSQL partitioning (still one database)
CREATE TABLE orders_2024_01 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

**Example Sharding:**
```
Database Server 1: orders_shard_0 (user_id % 4 = 0)
Database Server 2: orders_shard_1 (user_id % 4 = 1)
Database Server 3: orders_shard_2 (user_id % 4 = 2)
Database Server 4: orders_shard_3 (user_id % 4 = 3)
```

See: [URL Shortener - Partitioning](../url-shorterner.md#94-partitioning-time-based-partitioning-for-expiry-cleanup)

---

## Implementation Approaches

### Application-Level Sharding
Application decides which shard to query.

```python
def get_user(user_id):
    shard_num = hash(user_id) % NUM_SHARDS
    db = get_database_connection(shard_num)
    return db.query(f"SELECT * FROM users WHERE id = {user_id}")
```

**Pros:** Full control, flexible
**Cons:** Application complexity, hard to change

### Database-Level Sharding
Database system handles sharding automatically.

**Examples:**
- **Vitess** (for MySQL)
- **Citus** (for PostgreSQL)
- **MongoDB** (auto-sharding)

**Pros:** Transparent to application, easier to manage
**Cons:** Less control, vendor lock-in

---

## Summary

✅ **Hash-based sharding** - Best for even distribution
✅ **Range-based sharding** - Best for range queries
✅ **Consistent hashing** - Best for dynamic sharding
✅ **Choose shard key carefully** - High cardinality, even distribution
✅ **Avoid cross-shard queries** - Design around shard boundaries
✅ **Combine with replication** - For high availability

**Golden Rule:** Shard on the most frequently queried field with high cardinality!
