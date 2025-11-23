# Load Balancing

## Overview

A **load balancer** distributes incoming network traffic across multiple servers to ensure no single server becomes overwhelmed.

## Why Load Balancing?

### Problems Without Load Balancing

❌ **Single point of failure** - If the server crashes, the entire service goes down
❌ **Performance bottleneck** - One server can only handle limited requests
❌ **Poor resource utilization** - Can't scale horizontally

### Benefits of Load Balancing

✅ **High availability** - If one server fails, traffic routes to healthy servers
✅ **Scalability** - Add more servers to handle more traffic
✅ **Better performance** - Distribute load evenly across servers
✅ **Flexibility** - Perform maintenance without downtime

---

## Load Balancing Architecture

```
                    Internet
                       ↓
                 Load Balancer
                  /    |    \
                 /     |     \
           Server 1  Server 2  Server 3
                 \     |     /
                  \    |    /
                   Database
```

**Flow:**
1. User sends request to Load Balancer
2. Load Balancer selects a server based on algorithm
3. Server processes request and returns response
4. Load Balancer sends response back to user

---

## Load Balancing Algorithms

### 1. Round Robin

Distributes requests **sequentially** to each server in order.

**Example:**
```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1  (cycles back)
Request 5 → Server 2
```

**Pseudocode:**
```python
servers = ["server1", "server2", "server3"]
current_index = 0

def round_robin():
    global current_index
    server = servers[current_index]
    current_index = (current_index + 1) % len(servers)
    return server
```

**Pros:**
- ✅ Simple to implement
- ✅ Even distribution if all requests take equal time

**Cons:**
- ❌ Doesn't account for server capacity
- ❌ Doesn't account for request complexity

**Best for:** Homogeneous servers with similar requests

---

### 2. Weighted Round Robin

Like round robin, but servers with **higher capacity** get more requests.

**Example:**
```
Server 1: Weight 3 (powerful server)
Server 2: Weight 2
Server 3: Weight 1 (less powerful)

Distribution:
S1, S1, S1, S2, S2, S3  (repeats)
```

**Pseudocode:**
```python
servers = [
    {"name": "server1", "weight": 3},
    {"name": "server2", "weight": 2},
    {"name": "server3", "weight": 1}
]

def weighted_round_robin():
    # Expand servers based on weight
    expanded = []
    for server in servers:
        expanded.extend([server["name"]] * server["weight"])

    # Round robin on expanded list
    return round_robin_on(expanded)
```

**Best for:** Servers with different capacities

---

### 3. Least Connections

Routes traffic to the server with the **fewest active connections**.

**Example:**
```
Server 1: 10 active connections
Server 2: 5 active connections
Server 3: 8 active connections

→ New request goes to Server 2
```

**Pseudocode:**
```python
servers = {
    "server1": {"connections": 10},
    "server2": {"connections": 5},
    "server3": {"connections": 8}
}

def least_connections():
    return min(servers, key=lambda s: servers[s]["connections"])
```

**Pros:**
- ✅ Accounts for varying request durations
- ✅ Better distribution for long-running requests

**Cons:**
- ❌ Requires tracking connection count
- ❌ More complex than round robin

**Best for:** Applications with varying request times (e.g., video uploads)

---

### 4. Least Response Time

Routes to the server with the **fastest response time** and fewest active connections.

**Formula:**
```
Score = (Average Response Time) × (Active Connections)
→ Route to server with lowest score
```

**Best for:** Performance-critical applications

---

### 5. IP Hash (Consistent Hashing)

Routes requests based on **client IP address hash**.

**Formula:**
```
Server = hash(client_ip) % num_servers
```

**Example:**
```python
def ip_hash(client_ip, num_servers):
    hash_value = hash(client_ip)
    return hash_value % num_servers

# Client 192.168.1.100 always goes to same server
server = ip_hash("192.168.1.100", 3)
```

**Pros:**
- ✅ **Session persistence** - Same client always hits same server
- ✅ Good for stateful sessions

**Cons:**
- ❌ Uneven distribution if IP addresses are not uniformly distributed
- ❌ Adding/removing servers disrupts all sessions

**Best for:** Session-based applications without sticky sessions

---

### 6. Random

Selects a server **randomly**.

**Pseudocode:**
```python
import random

def random_selection(servers):
    return random.choice(servers)
```

**Pros:**
- ✅ Extremely simple
- ✅ Statistically even over time

**Cons:**
- ❌ Can be uneven in short term
- ❌ No intelligence

**Best for:** Simple use cases, testing

---

## Types of Load Balancers

### Layer 4 (Transport Layer) Load Balancer

Operates at **TCP/UDP level** (doesn't inspect HTTP content).

**How it works:**
- Routes based on IP address and port
- Fast (no packet inspection)
- Protocol-agnostic

**Example:**
```
Client → Load Balancer (looks at IP:Port)
       → Routes to Server without reading HTTP headers
```

**Technologies:**
- AWS Network Load Balancer (NLB)
- HAProxy (in L4 mode)

**Best for:** High-throughput, low-latency scenarios

---

### Layer 7 (Application Layer) Load Balancer

Operates at **HTTP level** (can read request content).

**How it works:**
- Routes based on URL path, headers, cookies
- Can modify requests (add headers, rewrite URLs)
- More intelligent but slower than L4

**Example:**
```
Client → Load Balancer
       → Reads URL path:
           /api/* → API Servers
           /static/* → Static Content Servers
           /video/* → Video Streaming Servers
```

**Pseudocode:**
```python
def layer7_route(request):
    if request.path.startswith("/api"):
        return api_servers
    elif request.path.startswith("/video"):
        return video_servers
    else:
        return web_servers
```

**Technologies:**
- AWS Application Load Balancer (ALB)
- Nginx
- HAProxy

**Best for:** Content-based routing, microservices

---

## Advanced Load Balancing Features

### 1. Health Checks

Load balancer **periodically checks** if servers are healthy.

**Example:**
```python
def health_check():
    for server in servers:
        try:
            response = http.get(f"{server}/health")
            if response.status_code == 200:
                server.mark_healthy()
            else:
                server.mark_unhealthy()
        except:
            server.mark_unhealthy()
```

**Unhealthy servers are removed** from rotation until they recover.

---

### 2. Sticky Sessions (Session Affinity)

Ensures a user's requests **always go to the same server**.

**Implementation:**
```
1. User's first request → Server 2
2. Load balancer sets cookie: session_id=abc123, server=server2
3. User's next requests → Read cookie → Always route to Server 2
```

**When to use:**
- Applications with in-memory session state
- WebSocket connections

**Better alternative:** Use external session storage (Redis) to avoid sticky sessions

---

### 3. SSL Termination

Load balancer **handles SSL/TLS encryption**, servers work with plain HTTP.

**Benefits:**
- ✅ Reduces server CPU load (no encryption/decryption)
- ✅ Centralized certificate management
- ✅ Servers can be simpler

**Flow:**
```
Client (HTTPS) → Load Balancer → Server (HTTP)
                ↓
          SSL decryption happens here
```

---

### 4. Rate Limiting

Load balancer can **limit requests per client** to prevent abuse.

**Example:**
```python
# Allow max 100 requests per minute per IP
def rate_limit(client_ip):
    count = redis.incr(f"rate:{client_ip}")
    redis.expire(f"rate:{client_ip}", 60)

    if count > 100:
        return "429 Too Many Requests"
```

---

## Load Balancer Placement

### 1. Between Client and Web Servers
```
Users → Load Balancer → Web Servers
```
**Most common setup**

### 2. Between Web Servers and Application Servers
```
Users → Web Servers → Load Balancer → App Servers
```
**For microservices**

### 3. Between Application and Database
```
App Servers → Load Balancer → Database Replicas
```
**For read-heavy databases**

---

## Global Load Balancing (DNS-Based)

Distributes traffic across **data centers** in different regions.

**How it works:**
1. User requests `example.com`
2. DNS resolver returns IP based on user location
3. User connects to nearest data center

**Example:**
```
User in USA → DNS returns 52.1.2.3 (US East)
User in Europe → DNS returns 35.4.5.6 (EU West)
```

**Technologies:**
- AWS Route 53
- Cloudflare Load Balancing
- Google Cloud Load Balancing

**Benefits:**
- ✅ Low latency (users hit nearest server)
- ✅ Disaster recovery (failover to other regions)

---

## Load Balancing Technologies

| Technology | Type | Best For |
|-----------|------|----------|
| **Nginx** | L4/L7 | Web servers, reverse proxy |
| **HAProxy** | L4/L7 | High performance, TCP/HTTP |
| **AWS ALB** | L7 | AWS-hosted applications |
| **AWS NLB** | L4 | High throughput, low latency |
| **AWS ELB (Classic)** | L4/L7 | Legacy AWS applications |
| **Google Cloud Load Balancing** | L4/L7 | Google Cloud Platform |
| **Cloudflare** | L7 | DDoS protection, CDN |

---

## Real-World Examples

### 1. URL Shortener
**Setup:**
```
Users → Nginx Load Balancer → Multiple API Servers
```
**Algorithm:** Round Robin (all requests similar)
**Health Check:** `/health` endpoint

See: [URL Shortener - Load Balancing](../url-shorterner.md#6-scaling-strategies)

---

### 2. Video Streaming Platform
**Setup:**
```
Users → CDN → Regional Load Balancers → Video Servers
```
**Algorithm:** Geographic routing + Least connections
**Why:** Users get content from nearest location

See: [Video Streaming - Traffic Scaling](../video-streaming-platform.md#6-scaling-strategies)

---

### 3. Netflix
**Multi-tier load balancing:**
1. DNS-based (Route 53) → Nearest AWS region
2. AWS ELB → Microservice clusters
3. Internal load balancers → Service instances

---

## Common Challenges

### 1. Uneven Load Distribution

**Problem:** One server gets more traffic than others

**Solutions:**
- Use Least Connections instead of Round Robin
- Implement weighted routing based on server capacity
- Monitor and rebalance

### 2. Stateful Applications

**Problem:** Load balancing breaks session state

**Solutions:**
- Use sticky sessions (not recommended)
- Store sessions in Redis/database (recommended)
- Make application stateless

### 3. Database Bottleneck

**Problem:** All servers hit the same database

**Solutions:**
- Read replicas with load balancing
- Database sharding
- Caching layer (Redis)

---

## Monitoring Load Balancers

**Key metrics:**
- **Request rate** - Requests per second
- **Error rate** - 4xx, 5xx responses
- **Latency** - Response time (p50, p95, p99)
- **Backend health** - Number of healthy servers
- **Connection count** - Active connections per server

---

## Summary

✅ **Round Robin** - Simple, equal distribution
✅ **Least Connections** - Best for varying request times
✅ **IP Hash** - Session persistence
✅ **Layer 7** - Content-based routing
✅ **Health Checks** - Automatic failover
✅ **Use external session storage** - Avoid sticky sessions

**Golden Rule:** Load balancing enables horizontal scaling - add more servers to handle more traffic!
