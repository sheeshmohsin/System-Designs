### **System Design: Rate Limiter (API Throttling)**

A **rate limiter** controls the rate of requests a client can make to an API, preventing abuse, ensuring fair usage, and protecting backend services from being overwhelmed.

## **1. Requirements Analysis**

### **Functional Requirements**
✅ Limit requests per client (by IP, user ID, API key)
✅ Support multiple time windows (per second, minute, hour, day)
✅ Support different rate limits for different endpoints
✅ Provide clear feedback when limit is exceeded (HTTP 429)
✅ Support different limit tiers (free, premium, enterprise)

### **Non-Functional Requirements**
✅ Low latency (< 5ms overhead)
✅ High availability (99.99% uptime)
✅ Distributed system support (multiple servers)
✅ Accurate counting (minimal false positives)
✅ Scalable to handle millions of requests

## **2. High-Level Architecture**

### **Key Components**

1. **Client Application**
   - Makes API requests
   - Receives rate limit headers in response

2. **API Gateway / Load Balancer**
   - First point of contact
   - Routes requests to rate limiter

3. **Rate Limiter Service**
   - Checks if request is within limits
   - Updates request counters
   - Returns allow/deny decision

4. **Redis Cluster**
   - Stores request counters
   - Fast in-memory operations
   - Distributed across multiple nodes

5. **Configuration Service**
   - Stores rate limit rules per endpoint/user
   - Allows dynamic updates without deployment

6. **Monitoring & Alerting**
   - Tracks rate limit violations
   - Alerts on unusual traffic patterns

## **3. Rate Limiting Algorithms**

### **Algorithm 1: Token Bucket**

**Most widely used** algorithm - allows bursts while maintaining average rate.

#### **How It Works**

1. Each client has a bucket with a maximum capacity of tokens
2. Tokens are added to the bucket at a fixed rate
3. Each request consumes one token
4. If bucket is empty, request is rejected

**Example:**
```
Bucket capacity: 10 tokens
Refill rate: 1 token/second

Time 0s: 10 tokens available
Request 1-10: Accepted (0 tokens left)
Request 11: Rejected (bucket empty)
Time 1s: 1 token added (1 token available)
Request 12: Accepted
```

**Pseudocode:**
```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = now()

    def allow_request(self):
        # Refill tokens based on time elapsed
        current_time = now()
        elapsed = current_time - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = current_time

        # Check if token available
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

# Usage
bucket = TokenBucket(capacity=10, refill_rate=1)
if bucket.allow_request():
    process_request()
else:
    return_429_error()
```

**Pros:**
- ✅ Allows traffic bursts (good UX)
- ✅ Memory efficient (2 values: tokens + timestamp)
- ✅ Simple to implement

**Cons:**
- ❌ Requires accurate time synchronization
- ❌ Can be difficult to tune bucket size

**Used by:** Amazon API Gateway, Stripe

---

### **Algorithm 2: Leaky Bucket**

**FIFO queue** - requests processed at constant rate.

#### **How It Works**

1. Incoming requests enter a queue (bucket)
2. Requests are processed at a fixed rate
3. If queue is full, new requests are rejected

**Example:**
```
Queue capacity: 10 requests
Process rate: 1 request/second

Time 0s: 10 requests arrive → Queue fills up
Time 1s: 1 request processed, 1 new request arrives
```

**Pseudocode:**
```python
from collections import deque
import time

class LeakyBucket:
    def __init__(self, capacity, leak_rate):
        self.capacity = capacity
        self.queue = deque()
        self.leak_rate = leak_rate  # requests per second
        self.last_leak = time.time()

    def leak(self):
        # Process requests at fixed rate
        current_time = time.time()
        elapsed = current_time - self.last_leak
        leaks = int(elapsed * self.leak_rate)

        for _ in range(min(leaks, len(self.queue))):
            self.queue.popleft()

        self.last_leak = current_time

    def allow_request(self):
        self.leak()

        if len(self.queue) < self.capacity:
            self.queue.append(time.time())
            return True
        return False
```

**Pros:**
- ✅ Smooths out traffic bursts
- ✅ Predictable output rate

**Cons:**
- ❌ Can reject requests during bursts (poor UX)
- ❌ More memory overhead (queue storage)

**Used by:** Traffic shaping in networks

---

### **Algorithm 3: Fixed Window Counter**

**Simple counting** within fixed time windows.

#### **How It Works**

1. Divide time into fixed windows (e.g., every minute starts at :00)
2. Count requests in current window
3. If count exceeds limit, reject request
4. Counter resets at window boundary

**Example:**
```
Limit: 100 requests/minute

12:00:00 - 12:00:59 → Window 1
12:01:00 - 12:01:59 → Window 2

12:00:00 - 12:00:59: 100 requests accepted
12:01:00: Counter resets, 100 more requests accepted
```

**Pseudocode:**
```python
import time

class FixedWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size  # seconds
        self.counter = 0
        self.window_start = time.time()

    def allow_request(self):
        current_time = time.time()

        # Check if we're in a new window
        if current_time - self.window_start >= self.window_size:
            self.counter = 0
            self.window_start = current_time

        # Check limit
        if self.counter < self.limit:
            self.counter += 1
            return True
        return False
```

**Pros:**
- ✅ Very simple to implement
- ✅ Memory efficient (1 counter)

**Cons:**
- ❌ **Edge case:** Double traffic at window boundaries
  ```
  12:00:50 - 12:00:59: 100 requests (allowed)
  12:01:00 - 12:01:09: 100 requests (allowed)
  → 200 requests in 20 seconds! (2x rate limit)
  ```

**Used by:** Simple rate limiting systems

---

### **Algorithm 4: Sliding Window Log**

**Accurate tracking** by storing timestamp of each request.

#### **How It Works**

1. Store timestamp of each request in a log
2. When new request arrives, remove outdated entries
3. Count remaining entries
4. If count < limit, allow request

**Example:**
```
Limit: 5 requests/minute

12:00:00: Request 1
12:00:10: Request 2
12:00:20: Request 3
12:00:30: Request 4
12:00:40: Request 5
12:00:50: Request 6 → Rejected (5 requests in last 60s)
12:01:01: Request 7 → Allowed (Request 1 expired)
```

**Pseudocode:**
```python
import time

class SlidingWindowLog:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size  # seconds
        self.log = []  # List of request timestamps

    def allow_request(self):
        current_time = time.time()

        # Remove outdated entries
        cutoff_time = current_time - self.window_size
        self.log = [t for t in self.log if t > cutoff_time]

        # Check limit
        if len(self.log) < self.limit:
            self.log.append(current_time)
            return True
        return False
```

**Pros:**
- ✅ Very accurate (no edge cases)
- ✅ No boundary issues

**Cons:**
- ❌ High memory usage (store every timestamp)
- ❌ Slow for high traffic (filtering old entries)

**Used by:** When accuracy is critical

---

### **Algorithm 5: Sliding Window Counter**

**Best of both worlds** - combines fixed window efficiency with sliding window accuracy.

#### **How It Works**

1. Track counters for current and previous windows
2. Estimate request count using weighted average

**Formula:**
```
requests_in_sliding_window =
    requests_in_current_window +
    (requests_in_previous_window × overlap_percentage)
```

**Example:**
```
Limit: 10 requests/minute
Window: 1 minute

12:00:00 - 12:00:59: 8 requests (previous window)
12:01:00 - 12:01:59: 5 requests (current window)

At 12:01:30 (30 seconds into current window):
overlap_percentage = 30/60 = 0.5
estimated_count = 5 + (8 × 0.5) = 9 requests

Request at 12:01:30 → Allowed (9 < 10)
```

**Pseudocode:**
```python
import time

class SlidingWindowCounter:
    def __init__(self, limit, window_size):
        self.limit = limit
        self.window_size = window_size
        self.current_window_count = 0
        self.previous_window_count = 0
        self.current_window_start = time.time()

    def allow_request(self):
        current_time = time.time()
        elapsed = current_time - self.current_window_start

        # Check if we're in a new window
        if elapsed >= self.window_size:
            self.previous_window_count = self.current_window_count
            self.current_window_count = 0
            self.current_window_start = current_time
            elapsed = 0

        # Calculate weighted count
        overlap_percentage = (self.window_size - elapsed) / self.window_size
        estimated_count = self.current_window_count + \
                         (self.previous_window_count * overlap_percentage)

        if estimated_count < self.limit:
            self.current_window_count += 1
            return True
        return False
```

**Pros:**
- ✅ Memory efficient (2 counters)
- ✅ Accurate (no boundary issues)
- ✅ Smooth rate limiting

**Cons:**
- ❌ Slightly complex logic

**Used by:** Cloudflare, Kong API Gateway

---

## **Algorithm Comparison**

| Algorithm | **Accuracy** | **Memory** | **Allows Bursts** | **Complexity** | **Best For** |
|-----------|-------------|-----------|------------------|---------------|-------------|
| **Token Bucket** | Good | Low | ✅ Yes | Low | General purpose |
| **Leaky Bucket** | Good | Medium | ❌ No | Medium | Traffic shaping |
| **Fixed Window** | Poor | Very Low | ❌ No | Very Low | Simple systems |
| **Sliding Log** | Excellent | High | ✅ Yes | High | Critical accuracy |
| **Sliding Counter** | Very Good | Low | ✅ Yes | Medium | **Recommended** |

---

## **4. Distributed Rate Limiting with Redis**

### **Redis Implementation (Sliding Window Counter)**

**Why Redis?**
- In-memory (fast: < 1ms operations)
- Atomic operations (INCR, EXPIRE)
- Distributed (Redis Cluster)
- Persistence options (RDB, AOF)

**Redis Data Structures:**
```
Key: rate_limit:{client_id}:{window_start_timestamp}
Value: Request count
TTL: 2 × window_size (to keep previous window)
```

**Lua Script for Atomic Operations:**
```lua
-- rate_limit.lua
local key_current = KEYS[1]   -- Current window key
local key_previous = KEYS[2]  -- Previous window key
local limit = tonumber(ARGV[1])
local window_size = tonumber(ARGV[2])
local current_time = tonumber(ARGV[3])
local window_start = tonumber(ARGV[4])

-- Get counts
local current_count = tonumber(redis.call('GET', key_current) or '0')
local previous_count = tonumber(redis.call('GET', key_previous) or '0')

-- Calculate elapsed time in current window
local elapsed = current_time - window_start
local overlap_percentage = (window_size - elapsed) / window_size

-- Calculate estimated count
local estimated_count = current_count + (previous_count * overlap_percentage)

if estimated_count < limit then
    -- Increment counter
    redis.call('INCR', key_current)
    redis.call('EXPIRE', key_current, window_size * 2)
    return {1, limit - estimated_count - 1}  -- {allowed, remaining}
else
    return {0, 0}  -- {denied, remaining}
end
```

**Python Implementation:**
```python
import redis
import time
import hashlib

class DistributedRateLimiter:
    def __init__(self, redis_client, limit, window_size):
        self.redis = redis_client
        self.limit = limit
        self.window_size = window_size  # seconds

        # Load Lua script
        self.script = self.redis.register_script(RATE_LIMIT_LUA_SCRIPT)

    def allow_request(self, client_id):
        current_time = int(time.time())
        current_window_start = (current_time // self.window_size) * self.window_size
        previous_window_start = current_window_start - self.window_size

        key_current = f"rate_limit:{client_id}:{current_window_start}"
        key_previous = f"rate_limit:{client_id}:{previous_window_start}"

        # Execute Lua script atomically
        result = self.script(
            keys=[key_current, key_previous],
            args=[self.limit, self.window_size, current_time, current_window_start]
        )

        allowed = result[0]
        remaining = result[1]

        return {
            "allowed": bool(allowed),
            "remaining": int(remaining),
            "reset_time": current_window_start + self.window_size
        }

# Usage
redis_client = redis.Redis(host='localhost', port=6379)
limiter = DistributedRateLimiter(redis_client, limit=100, window_size=60)

# Check if request allowed
response = limiter.allow_request(client_id="user_123")
if response["allowed"]:
    process_request()
else:
    return_429_error(response)
```

---

## **5. API Response Headers**

**Standard Rate Limit Headers:**
```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200
Retry-After: 42
```

**When Rate Limited:**
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640995200
Retry-After: 42

{
  "error": "Rate limit exceeded",
  "message": "You have exceeded 100 requests per minute",
  "retry_after": 42
}
```

---

## **6. Multi-Tier Rate Limiting**

**Different limits for different user tiers:**

```python
class TieredRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.tiers = {
            "free": {"limit": 100, "window": 3600},     # 100/hour
            "basic": {"limit": 1000, "window": 3600},   # 1,000/hour
            "premium": {"limit": 10000, "window": 3600}, # 10,000/hour
            "enterprise": {"limit": 100000, "window": 3600} # 100,000/hour
        }

    def get_user_tier(self, user_id):
        # Query database or cache
        return db.query(f"SELECT tier FROM users WHERE id={user_id}")

    def allow_request(self, user_id):
        tier = self.get_user_tier(user_id)
        config = self.tiers[tier]

        limiter = DistributedRateLimiter(
            self.redis,
            limit=config["limit"],
            window_size=config["window"]
        )

        return limiter.allow_request(user_id)
```

---

## **7. Rate Limiting Strategies**

### **Strategy 1: Per-IP Rate Limiting**
```python
client_id = request.ip_address
limiter.allow_request(client_id)
```

**Pros:** Prevents abuse from single source
**Cons:** NAT/proxy users share same IP

---

### **Strategy 2: Per-User Rate Limiting**
```python
client_id = request.user_id  # From auth token
limiter.allow_request(client_id)
```

**Pros:** Fair for individual users
**Cons:** Requires authentication

---

### **Strategy 3: Per-API-Key Rate Limiting**
```python
client_id = request.headers.get("X-API-Key")
limiter.allow_request(client_id)
```

**Pros:** Easy tracking per application
**Cons:** Key sharing bypasses limits

---

### **Strategy 4: Composite Rate Limiting**
```python
# Multiple limits
if not limiter.allow_request(f"ip:{request.ip}", limit=1000):
    return 429

if not limiter.allow_request(f"user:{request.user_id}", limit=100):
    return 429

if not limiter.allow_request(f"endpoint:{request.path}", limit=500):
    return 429
```

**Most robust:** Check all limits

---

## **8. Scaling Strategies**

### **Database Scaling**
- **Redis Cluster:** Shard across multiple nodes
- **Read Replicas:** For high read traffic
- **Persistence:** Enable RDB/AOF for durability

### **Application Scaling**
- **Stateless API servers:** Scale horizontally
- **Load balancers:** Distribute traffic
- **CDN:** Cache rate limit configurations

### **Optimization**
- **Lua scripts:** Atomic operations in Redis
- **Pipelining:** Batch Redis commands
- **Connection pooling:** Reuse Redis connections

---

## **9. Security Considerations**

| **Issue** | **Solution** |
|-----------|-------------|
| DDoS attacks | Layer 7 rate limiting + WAF |
| Distributed attacks | IP reputation scoring |
| Credential stuffing | Stricter limits on auth endpoints |
| Bypassing via proxies | CAPTCHAs after threshold |

---

## **10. Capacity Estimation & Performance**

### **Traffic Estimation**

**Assumptions:**
- 1 million API calls per second
- Average 10% of requests hit rate limiter

**Rate Limiter Traffic:**
```
1M requests/sec × 10% = 100,000 rate limit checks/sec
```

### **Redis Memory Estimation**

**Per rate limit entry:**
```
Key: "rate_limit:user123:1640995200" ≈ 40 bytes
Value: Counter (4 bytes)
TTL: 8 bytes
Total: ~52 bytes
```

**For 10 million active users (1-minute windows):**
```
10M users × 2 windows (current + previous) × 52 bytes
= 20M entries × 52 bytes
= 1.04 GB
```

**Redis Memory Needed: ~2 GB (with overhead)**

### **Performance Metrics**

**Target Latency:**
- Rate limit check: < 1ms (Redis operation)
- Total API overhead: < 5ms

**Throughput:**
- Single Redis node: 100,000 ops/sec
- Redis Cluster (10 nodes): 1,000,000 ops/sec

---

## **11. Interview Questions & Answers**

### **Q1: How do you handle clock skew in distributed systems?**

**A:** Use Redis's server time instead of application server time:
```python
# Use Redis TIME command
redis_time = redis.time()  # Returns server timestamp
current_time = redis_time[0]  # Use Redis's time
```

Or use a distributed time service like **NTP** or **TrueTime (Google Spanner)**.

---

### **Q2: What happens if Redis goes down?**

**A:** **Graceful degradation:**

```python
def allow_request(client_id):
    try:
        return limiter.allow_request(client_id)
    except redis.ConnectionError:
        # Fail open (allow requests) or fail closed (deny requests)
        logger.error("Redis down - failing open")
        return {"allowed": True, "remaining": -1}
```

**Options:**
- **Fail Open:** Allow all requests (risk: no rate limiting)
- **Fail Closed:** Deny all requests (risk: service outage)

**Best:** Fail open + alert engineers

**Prevention:**
- Redis Sentinel (auto-failover)
- Redis Cluster (high availability)

---

### **Q3: How do you rate limit across multiple data centers?**

**A:** **Two approaches:**

**1. Global Redis Cluster (Consistent but Slower):**
```
DC1 → Redis Cluster (Global)
DC2 → Redis Cluster (Global)
DC3 → Redis Cluster (Global)
```
- Accurate global limits
- Higher latency (cross-region)

**2. Local Rate Limiting + Eventual Sync:**
```
DC1 → Redis (Local) → Sync to Global
DC2 → Redis (Local) → Sync to Global
DC3 → Redis (Local) → Sync to Global

Global Limit: 1000/min
Per DC Limit: 1000/3 ≈ 333/min
```
- Lower latency
- Approximate limits (can exceed global limit temporarily)

**Recommended:** Local with periodic sync

---

### **Q4: How do you prevent users from gaming the rate limiter?**

**A:** Multiple defenses:

1. **Multiple identifiers:**
   ```python
   # Check all of: IP, user_id, API key
   if not check_ip_limit() or not check_user_limit() or not check_key_limit():
       return 429
   ```

2. **Progressive penalties:**
   ```python
   if violations > 5:
       ban_duration = min(violations * 60, 3600)  # Up to 1 hour
       redis.setex(f"ban:{user_id}", ban_duration, 1)
   ```

3. **CAPTCHA challenges:**
   ```python
   if limiter.remaining < 10:
       require_captcha()
   ```

---

### **Q5: How do you implement rate limiting for WebSockets?**

**A:** Rate limit on WebSocket events:

```python
def on_message(websocket, message):
    client_id = websocket.user_id

    if not limiter.allow_request(f"ws:{client_id}"):
        websocket.send({
            "error": "Rate limit exceeded",
            "type": "RATE_LIMIT_ERROR"
        })
        return

    process_message(message)
```

**Different limits for:**
- Connection rate (new connections/min)
- Message rate (messages/sec)
- Bandwidth (bytes/sec)

---

### **Q6: How do you implement custom rate limits per endpoint?**

**A:** Endpoint-specific configuration:

```python
RATE_LIMITS = {
    "/api/auth/login": {"limit": 5, "window": 300},      # 5/5min (stricter)
    "/api/users/profile": {"limit": 100, "window": 60},  # 100/min
    "/api/search": {"limit": 50, "window": 60},          # 50/min (expensive)
    "default": {"limit": 1000, "window": 60}             # 1000/min
}

def allow_request(client_id, endpoint):
    config = RATE_LIMITS.get(endpoint, RATE_LIMITS["default"])
    limiter = DistributedRateLimiter(redis, config["limit"], config["window"])
    return limiter.allow_request(f"{client_id}:{endpoint}")
```

---

### **Q7: Should rate limiting happen at API Gateway or application level?**

**A:** **Both (defense in depth):**

**API Gateway (Layer 7):**
- Protects all services
- Centralized configuration
- Faster (before hitting application)
- Use for: Global limits, DDoS protection

**Application Level:**
- More granular (per endpoint, per resource)
- Business logic aware
- Use for: Feature-specific limits, user tier limits

**Example:**
```
Client → API Gateway (1000 req/sec global)
       → Application (100 req/sec per user)
       → Database
```

---

### **Q8: How do you test rate limiting?**

**A:** Multiple testing strategies:

```python
# Unit tests
def test_rate_limiter():
    limiter = TokenBucket(capacity=5, refill_rate=1)

    # Should allow 5 requests
    for i in range(5):
        assert limiter.allow_request() == True

    # 6th request should be denied
    assert limiter.allow_request() == False

    # Wait 1 second for refill
    time.sleep(1)
    assert limiter.allow_request() == True

# Load tests
def load_test_rate_limiter():
    for i in range(1000):
        response = requests.get("/api/test")
        if response.status_code == 429:
            assert "Retry-After" in response.headers
```

**Tools:** Locust, JMeter, k6

---

### **Q9: How do you implement rate limiting for GraphQL?**

**A:** Rate limit on query complexity:

```python
from graphql import parse

def calculate_query_cost(query):
    # Parse GraphQL query
    ast = parse(query)

    cost = 0
    for field in ast.fields:
        if field.name == "users":
            cost += 10  # Expensive query
        elif field.name == "user":
            cost += 1   # Cheap query

    return cost

def allow_graphql_request(client_id, query):
    cost = calculate_query_cost(query)

    # Deduct cost from rate limit
    if not limiter.allow_request(client_id, cost=cost):
        return {"error": "Rate limit exceeded"}

    return execute_query(query)
```

---

### **Q10: How do you migrate from one rate limiting algorithm to another?**

**A:** **Dual-mode deployment:**

**Phase 1: Run both algorithms (log only)**
```python
def allow_request(client_id):
    old_result = old_limiter.allow_request(client_id)
    new_result = new_limiter.allow_request(client_id)

    # Log differences for analysis
    if old_result != new_result:
        log_difference(client_id, old_result, new_result)

    return old_result  # Still use old algorithm
```

**Phase 2: Canary rollout**
```python
if random.random() < 0.1:  # 10% traffic
    return new_limiter.allow_request(client_id)
else:
    return old_limiter.allow_request(client_id)
```

**Phase 3: Full migration**
```python
return new_limiter.allow_request(client_id)
```

---

## **12. Summary: Key Takeaways**

✅ **Sliding Window Counter** - Best balance of accuracy and efficiency
✅ **Redis** - Fast, distributed, atomic operations
✅ **Lua scripts** - Atomic multi-step operations in Redis
✅ **Multiple identifiers** - Rate limit by IP + user + API key
✅ **Graceful degradation** - Fail open if Redis is down
✅ **Custom limits** - Per endpoint, per tier, per user
✅ **Standard headers** - X-RateLimit-*, Retry-After

🚀 **This is how Stripe, GitHub, Twitter, and Cloudflare implement rate limiting!**
