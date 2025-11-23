# Caching Strategies

## Overview

Caching is one of the most important optimization techniques in system design. It stores frequently accessed data in fast storage (memory) to reduce latency and database load.

## Why Use Caching?

✅ **Reduce latency** - Memory access (1ms) vs Database access (10-100ms)
✅ **Lower database load** - Reduce expensive database queries
✅ **Handle traffic spikes** - Serve cached data during high traffic
✅ **Cost savings** - Fewer database operations = lower infrastructure costs

## Common Caching Patterns

### 1. Cache-Aside (Lazy Loading)

The **most common caching pattern**. Application checks cache first, then fetches from database if needed.

**Flow:**
1. Application requests data
2. Check cache - if found ✅ (Cache Hit), return immediately
3. If not found ❌ (Cache Miss), query database
4. Store result in cache for next time
5. Return data to application

**Pseudocode:**
```python
def get_data(key):
    # Try cache first
    data = cache.get(key)

    if data is not None:
        return data  # Cache hit

    # Cache miss - fetch from database
    data = database.query(key)

    # Store in cache for future requests
    cache.set(key, data, ttl=3600)  # 1 hour TTL

    return data
```

**Pros:**
- Only caches data that's actually requested
- Cache failures don't break the system

**Cons:**
- Initial requests are slower (cache miss)
- Can result in stale data

**Used in:**
- [URL Shortener](../url-shorterner.md) - Caching short URL → original URL mappings
- Most read-heavy applications

---

### 2. Write-Through Cache

Data is written to cache and database **simultaneously**.

**Flow:**
1. Application writes data
2. Cache is updated
3. Database is updated
4. Both operations complete before returning success

**Pseudocode:**
```python
def write_data(key, value):
    # Write to cache
    cache.set(key, value)

    # Write to database
    database.write(key, value)

    return success
```

**Pros:**
- Cache is always consistent with database
- No stale data

**Cons:**
- Higher write latency (two write operations)
- Writes data that may never be read

**Used in:**
- Systems requiring strong consistency
- Financial transactions

---

### 3. Write-Behind (Write-Back) Cache

Data is written to cache immediately, then **asynchronously** written to database.

**Flow:**
1. Application writes data to cache
2. Return success immediately
3. Background process writes to database later

**Pseudocode:**
```python
def write_data(key, value):
    # Write to cache (fast)
    cache.set(key, value)

    # Queue for async database write
    write_queue.enqueue(key, value)

    return success  # Immediate response

# Background worker
def background_worker():
    while True:
        item = write_queue.dequeue()
        database.write(item.key, item.value)
```

**Pros:**
- Very fast writes
- Can batch database writes
- Reduces database load

**Cons:**
- Risk of data loss if cache fails before database write
- More complex to implement

**Used in:**
- High-throughput write systems
- Analytics data collection

---

### 4. Read-Through Cache

Cache sits **between** application and database. All reads go through cache.

**Flow:**
1. Application requests data from cache
2. Cache checks if it has the data
3. If not, cache fetches from database and stores it
4. Cache returns data

**Difference from Cache-Aside:**
- Application only talks to cache
- Cache is responsible for loading from database

**Pros:**
- Simplifies application logic
- Cache controls loading strategy

**Cons:**
- Tight coupling between cache and database
- Cache becomes a critical dependency

---

## Cache Eviction Policies

When cache is full, which items should be removed?

### LRU (Least Recently Used)
Removes items that haven't been accessed in the longest time.

**Best for:** General-purpose caching

**Example:** Redis default eviction policy
```python
cache.set("key1", "value1")  # Last used: now
cache.set("key2", "value2")  # Last used: now
cache.get("key1")            # key1 is now most recent
# If cache is full, key2 gets evicted first
```

### LFU (Least Frequently Used)
Removes items accessed the fewest times.

**Best for:** When access patterns are predictable

### FIFO (First In, First Out)
Removes oldest items first (like a queue).

**Best for:** Time-series data

### TTL (Time To Live)
Items expire after a set time period.

**Best for:** Data with natural expiration (sessions, temporary tokens)

**Example:**
```python
cache.set("session:user123", session_data, ttl=3600)  # Expires in 1 hour
```

---

## Cache Hierarchies

Multiple cache layers for better performance.

### Multi-Level Caching

```
User Request
    ↓
L1: Application Cache (In-Memory)
    ↓ (miss)
L2: Distributed Cache (Redis)
    ↓ (miss)
L3: Database
```

**Example:** [Video Streaming Platform](../video-streaming-platform.md)

```
User Request
    ↓
Browser Cache (HTML5 local storage)
    ↓
CDN Edge Cache (nearest location)
    ↓
Regional CDN Cache
    ↓
Origin Storage (AWS S3)
```

---

## Cache Key Design

### Good Key Design
- **Predictable:** Same input = same key
- **Unique:** Different data = different key
- **Readable:** Easy to debug

**Examples:**
```
user:123:profile
short_url:abc123
video:456:1080p:segment5
product:789:inventory
```

### Bad Key Design
```
user_data           # Too generic
123                 # No context
random_hash_xyz     # Not predictable
```

---

## Cache Warming

**Problem:** Empty cache after restart causes all requests to hit database (Cache Stampede).

**Solution:** Pre-populate cache with critical data.

```python
def warm_cache():
    # Load most popular items into cache
    popular_items = database.query("SELECT * FROM items ORDER BY views DESC LIMIT 1000")

    for item in popular_items:
        cache.set(f"item:{item.id}", item, ttl=86400)
```

---

## Cache Invalidation

**"There are only two hard things in Computer Science: cache invalidation and naming things."** - Phil Karlton

### Strategies

#### 1. TTL-Based Invalidation
Let cache expire naturally.
```python
cache.set("key", "value", ttl=3600)  # Auto-expires in 1 hour
```

#### 2. Event-Based Invalidation
Invalidate when data changes.
```python
def update_user(user_id, new_data):
    database.update(user_id, new_data)
    cache.delete(f"user:{user_id}")  # Invalidate cache
```

#### 3. Version-Based Invalidation
Add version to cache key.
```python
cache.set(f"user:{user_id}:v2", user_data)
```

---

## Common Caching Technologies

| Technology | Type | Best For | Used In |
|-----------|------|----------|---------|
| **Redis** | In-memory key-value | General-purpose caching | URL Shortener, Session storage |
| **Memcached** | In-memory key-value | Simple caching | High-speed caching |
| **CDN (CloudFront, Akamai)** | Distributed edge cache | Static content, videos | Video Streaming |
| **Varnish** | HTTP cache | Web page caching | High-traffic websites |
| **Local Cache (dict, hashmap)** | In-process memory | Application-level caching | Small datasets |

---

## Real-World Examples

### 1. URL Shortener
**Pattern:** Cache-Aside
**Key:** `short_url:abc123`
**Value:** `https://original-url.com`
**TTL:** 24 hours
**Why:** 90% of traffic hits popular URLs - caching reduces database load by 90%

See: [URL Shortener - Caching Strategy](../url-shorterner.md#5-caching-strategy-redis)

### 2. Video Streaming Platform
**Pattern:** Multi-tier caching (CDN)
**Cached:** Video chunks, playlists, metadata
**Why:** Reduces storage read operations by 95-99%

See: [Video Streaming - CDN Caching](../video-streaming-platform/CDN-Caching.md)

---

## Cache Performance Metrics

### Cache Hit Ratio
```
Cache Hit Ratio = (Cache Hits) / (Total Requests) × 100%
```

**Good ratios:**
- 80-90%: Acceptable
- 90-95%: Good
- 95%+: Excellent

### Cache Miss Impact
Every 10% decrease in cache hit ratio can double database load.

---

## When NOT to Use Caching

❌ Data changes very frequently (cache invalidation overhead)
❌ Data is rarely accessed (wastes memory)
❌ Strong consistency required (stale data not acceptable)
❌ Data is already fast to fetch (no performance gain)

---

## Summary

✅ **Cache-Aside** is the most common pattern
✅ **LRU** is the most common eviction policy
✅ **TTL** prevents stale data automatically
✅ **Multi-tier caching** handles massive scale
✅ **Good cache key design** is critical for debugging

**Golden Rule:** Cache what's expensive to compute and frequently accessed!
