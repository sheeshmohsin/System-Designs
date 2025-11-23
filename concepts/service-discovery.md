# Service Discovery

## Overview

**Service discovery** is the automatic detection of services and their network locations in a distributed system. Instead of hardcoding IP addresses and ports, services dynamically find and communicate with each other.

## Why Service Discovery?

### Problems Without Service Discovery

❌ **Hardcoded endpoints** - IP addresses in config files become stale
❌ **Manual updates** - Every deployment requires config changes
❌ **No failover** - Dead services still in config
❌ **Scaling issues** - Can't dynamically add/remove instances
❌ **No health awareness** - Route traffic to unhealthy instances

### Goals

✅ **Dynamic registration** - Services auto-register on startup

✅ **Automatic updates** - No manual config changes

✅ **Health checking** - Only route to healthy instances

✅ **Load distribution** - Balance traffic across instances

✅ **Failover** - Automatically remove failed instances

---

## Service Discovery Patterns

### 1. Client-Side Discovery

**Client is responsible for finding service instances.**

```
┌────────────┐
│  Client    │
└─────┬──────┘
      │
      │ 1. Query
      ↓
┌─────────────────┐
│ Service Registry│
│  (Consul/Eureka)│
└─────────────────┘
      │
      │ 2. Returns [IP1, IP2, IP3]
      ↓
┌────────────┐
│  Client    │
└─────┬──────┘
      │ 3. Choose instance (load balance)
      ↓
┌─────────────────┐
│ Service Instance│
└─────────────────┘
```

**How it works:**

1. Client queries service registry
2. Registry returns list of available instances
3. Client chooses instance (using load balancing algorithm)
4. Client makes direct request to instance

**Example: Netflix Eureka**

```java
// Service Registration
@EnableEurekaClient
@SpringBootApplication
public class OrderService {
    public static void main(String[] args) {
        SpringApplication.run(OrderService.class, args);
    }
}

// application.yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    hostname: order-service
```

```java
// Client Discovery
@Autowired
private DiscoveryClient discoveryClient;

public void callPaymentService() {
    // Get instances of payment service
    List<ServiceInstance> instances =
        discoveryClient.getInstances("payment-service");

    if (!instances.isEmpty()) {
        // Choose first instance (could use load balancing)
        ServiceInstance instance = instances.get(0);
        String url = instance.getUri().toString() + "/api/charge";

        // Make HTTP request
        restTemplate.postForObject(url, payment, Response.class);
    }
}
```

**Pros:**
- ✅ Simple architecture
- ✅ Client controls load balancing

**Cons:**
- ❌ Client complexity (discovery logic in every client)
- ❌ Language-specific libraries needed
- ❌ Tighter coupling

**Used in:** Netflix OSS (Eureka), Spring Cloud

---

### 2. Server-Side Discovery

**Load balancer handles discovery and routing.**

```
┌────────────┐
│  Client    │
└─────┬──────┘
      │
      │ 1. Request to service name
      ↓
┌─────────────────┐
│  Load Balancer  │
│  (Nginx/AWS ELB)│
└─────┬───────────┘
      │
      │ 2. Query
      ↓
┌─────────────────┐
│ Service Registry│
└─────────────────┘
      │
      │ 3. Route to healthy instance
      ↓
┌─────────────────┐
│ Service Instance│
└─────────────────┘
```

**How it works:**

1. Client makes request to load balancer
2. Load balancer queries service registry
3. Load balancer routes to healthy instance
4. Response flows back through load balancer

**Example: Kubernetes + Service**

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment
  template:
    metadata:
      labels:
        app: payment
    spec:
      containers:
      - name: payment
        image: payment-service:1.0
        ports:
        - containerPort: 8080
---
# Service (acts as load balancer)
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

```python
# Client code - just use service name!
import requests

response = requests.post(
    "http://payment-service/api/charge",
    json={"amount": 100}
)
```

**Pros:**
- ✅ Simple client (no discovery logic)
- ✅ Language-agnostic
- ✅ Centralized load balancing

**Cons:**
- ❌ Extra network hop (through load balancer)
- ❌ Load balancer is single point of failure
- ❌ Additional infrastructure required

**Used in:** Kubernetes, AWS ELB, Nginx

---

## Service Registry Tools

### 1. Consul

**HashiCorp's service mesh with built-in discovery, health checking, and KV store.**

**Service Registration:**

```bash
# Register service via HTTP API
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -d '{
    "ID": "payment-1",
    "Name": "payment-service",
    "Address": "192.168.1.10",
    "Port": 8080,
    "Check": {
      "HTTP": "http://192.168.1.10:8080/health",
      "Interval": "10s"
    }
  }'
```

**Service Discovery:**

```python
import consul

# Connect to Consul
c = consul.Consul(host='localhost', port=8500)

# Discover healthy instances
index, services = c.health.service('payment-service', passing=True)

for service in services:
    address = service['Service']['Address']
    port = service['Service']['Port']
    print(f"Found instance: {address}:{port}")
```

**Features:**
- ✅ Health checking (HTTP/TCP/script)
- ✅ Multi-datacenter support
- ✅ DNS interface (`payment-service.service.consul`)
- ✅ Key-Value store
- ✅ Service mesh capabilities

---

### 2. Eureka (Netflix)

**Service registry optimized for AWS cloud.**

**Configuration:**

```yaml
# eureka-server (Registry)
server:
  port: 8761

eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
```

```java
// Service registration (automatic with @EnableEurekaClient)
@SpringBootApplication
@EnableEurekaClient
public class PaymentService {
    // Spring Boot auto-registers on startup
}
```

**Features:**
- ✅ Self-preservation mode (handles network partitions)
- ✅ Region/zone awareness (AWS)
- ✅ Dashboard UI
- ✅ REST API

**Cons:**
- ❌ Java-centric
- ❌ Eventually consistent (AP in CAP)
- ❌ No built-in health checks (relies on heartbeats)

---

### 3. etcd

**Distributed KV store used by Kubernetes for service discovery.**

**Service Registration:**

```bash
# Store service instance
etcdctl put /services/payment/instance1 '{"host":"192.168.1.10","port":8080}'
etcdctl put /services/payment/instance2 '{"host":"192.168.1.11","port":8080}'
```

**Service Discovery:**

```go
package main

import (
    "context"
    "go.etcd.io/etcd/client/v3"
)

func discoverServices() {
    cli, _ := clientv3.New(clientv3.Config{
        Endpoints: []string{"localhost:2379"},
    })
    defer cli.Close()

    // Get all payment service instances
    resp, _ := cli.Get(context.Background(), "/services/payment/", clientv3.WithPrefix())

    for _, kv := range resp.Kvs {
        fmt.Printf("Instance: %s\n", kv.Value)
    }
}
```

**Features:**
- ✅ Strongly consistent (Raft consensus)
- ✅ Watch API (real-time updates)
- ✅ TTL-based expiration
- ✅ Used by Kubernetes

---

### 4. Zookeeper

**Mature distributed coordination service.**

**Service Registration:**

```java
ZooKeeper zk = new ZooKeeper("localhost:2181", 3000, null);

// Create ephemeral node (disappears on disconnect)
String path = zk.create(
    "/services/payment/instance-",
    "192.168.1.10:8080".getBytes(),
    ZooDefs.Ids.OPEN_ACL_UNSAFE,
    CreateMode.EPHEMERAL_SEQUENTIAL
);
```

**Service Discovery:**

```java
// Get all instances
List<String> instances = zk.getChildren("/services/payment", false);

for (String instance : instances) {
    byte[] data = zk.getData("/services/payment/" + instance, false, null);
    System.out.println("Instance: " + new String(data));
}
```

**Features:**
- ✅ Battle-tested (used by Kafka, Hadoop)
- ✅ Strong consistency
- ✅ Watches for real-time updates

**Cons:**
- ❌ Complexity
- ❌ CP in CAP (sacrifices availability)

---

## Health Checking

**Ensure only healthy instances receive traffic.**

### Health Check Types

#### 1. HTTP Health Check

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/health')
def health():
    # Check dependencies
    db_healthy = check_database_connection()
    cache_healthy = check_redis_connection()

    if db_healthy and cache_healthy:
        return jsonify({"status": "healthy"}), 200
    else:
        return jsonify({"status": "unhealthy"}), 503
```

**Consul configuration:**

```json
{
  "Check": {
    "HTTP": "http://localhost:8080/health",
    "Interval": "10s",
    "Timeout": "5s"
  }
}
```

---

#### 2. TCP Health Check

```json
{
  "Check": {
    "TCP": "localhost:8080",
    "Interval": "10s"
  }
}
```

**Used when:** Service doesn't expose HTTP endpoint

---

#### 3. Script/Command Health Check

```json
{
  "Check": {
    "Args": ["/bin/check_health.sh"],
    "Interval": "30s"
  }
}
```

```bash
#!/bin/bash
# check_health.sh
if [ $(ps aux | grep my-app | wc -l) -gt 1 ]; then
    exit 0  # Healthy
else
    exit 1  # Unhealthy
fi
```

---

#### 4. TTL Health Check (Heartbeat)

**Service must send heartbeat before TTL expires.**

```python
import consul
import time

c = consul.Consul()

# Register with TTL check
c.agent.service.register(
    'payment-service',
    check=consul.Check.ttl('30s')
)

# Send heartbeat every 10 seconds
while True:
    c.agent.check.ttl_pass('service:payment-service')
    time.sleep(10)
```

---

## DNS-Based Service Discovery

**Use DNS for service discovery (simple, universal).**

### Internal DNS (Consul DNS)

```bash
# Query Consul DNS
dig @localhost -p 8600 payment-service.service.consul

# Returns:
# payment-service.service.consul. 0 IN A 192.168.1.10
# payment-service.service.consul. 0 IN A 192.168.1.11
```

**Application code:**

```python
import requests

# DNS resolves to one of the IPs
response = requests.get("http://payment-service.service.consul:8080/api/charge")
```

**Pros:**
- ✅ Universal (all languages support DNS)
- ✅ No special libraries needed

**Cons:**
- ❌ DNS caching issues
- ❌ No client-side load balancing control

---

### Kubernetes DNS

**Every service gets DNS entry automatically.**

```bash
# Format: <service-name>.<namespace>.svc.cluster.local
curl http://payment-service.default.svc.cluster.local/api/charge
```

---

## Service Mesh

**Advanced service discovery with traffic management, security, and observability.**

### Istio Example

```yaml
# Service automatically discovered via sidecar proxy
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
  - port: 8080
---
# Traffic routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-routing
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        version:
          exact: v2
    route:
    - destination:
        host: payment-service
        subset: v2
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90
    - destination:
        host: payment-service
        subset: v2
      weight: 10
```

**Features:**
- ✅ Automatic discovery via sidecar injection
- ✅ Advanced traffic routing (A/B testing, canary)
- ✅ Mutual TLS between services
- ✅ Distributed tracing

---

## Registration Patterns

### 1. Self-Registration

**Service registers itself on startup.**

```python
import consul
import atexit

c = consul.Consul()

# Register on startup
c.agent.service.register(
    name='payment-service',
    service_id='payment-1',
    address='192.168.1.10',
    port=8080,
    check=consul.Check.http('http://192.168.1.10:8080/health', '10s')
)

# Deregister on shutdown
def deregister():
    c.agent.service.deregister('payment-1')

atexit.register(deregister)
```

**Pros:** Simple, service controls registration
**Cons:** Service must implement registration logic

---

### 2. Third-Party Registration

**Separate registrar service handles registration.**

```
┌─────────────┐
│  Service    │
└──────┬──────┘
       │
       │ Health checks
       ↓
┌─────────────┐      ┌──────────────┐
│ Registrar   │─────→│  Registry    │
│ (Registrator)│      │  (Consul)    │
└─────────────┘      └──────────────┘
```

**Example: Registrator (Docker)**

```bash
# Run Registrator (watches Docker events)
docker run -d \
  --name=registrator \
  --net=host \
  --volume=/var/run/docker.sock:/tmp/docker.sock \
  gliderlabs/registrator:latest \
  consul://localhost:8500

# Start service (auto-registered by Registrator)
docker run -d \
  --name payment-1 \
  -p 8080:8080 \
  -e "SERVICE_NAME=payment-service" \
  payment-service:latest
```

**Pros:** No registration code in service
**Cons:** Additional component to manage

---

## Real-World Examples

### 1. Microservices Communication

```python
# Order Service discovers Payment Service
import consul

c = consul.Consul()

def process_order(order):
    # Discover payment service
    _, services = c.health.service('payment-service', passing=True)

    if services:
        instance = services[0]['Service']
        payment_url = f"http://{instance['Address']}:{instance['Port']}/charge"

        response = requests.post(payment_url, json={
            "order_id": order.id,
            "amount": order.total
        })

        return response.json()
```

---

### 2. Database Connection Pooling

```python
# Discover database replicas
_, replicas = c.health.service('postgres-replica', passing=True)

# Create connection pool with all healthy replicas
connection_strings = [
    f"postgresql://{svc['Service']['Address']}:{svc['Service']['Port']}/mydb"
    for svc in replicas
]

# Round-robin across replicas
pool = ConnectionPool(connection_strings)
```

---

### 3. Dynamic Load Balancer Configuration

**Consul Template + Nginx:**

```nginx
# nginx.conf.tmpl
upstream payment_backend {
{{range service "payment-service"}}
    server {{.Address}}:{{.Port}};
{{end}}
}

server {
    location /api/payment {
        proxy_pass http://payment_backend;
    }
}
```

```bash
# Consul Template watches Consul and regenerates nginx.conf
consul-template \
  -template "nginx.conf.tmpl:nginx.conf:nginx -s reload"
```

---

## Comparison Table

| Tool | **Type** | **Consistency** | **Health Checks** | **Multi-DC** | **Best For** |
|------|---------|----------------|------------------|-------------|-------------|
| **Consul** | Registry | CP | HTTP/TCP/Script | Yes | Service mesh, multi-DC |
| **Eureka** | Registry | AP | Heartbeat only | No | AWS, Spring Cloud |
| **etcd** | KV Store | CP | TTL | Yes | Kubernetes, config |
| **Zookeeper** | Coordination | CP | Ephemeral nodes | Yes | Kafka, Hadoop |
| **Kubernetes** | Orchestrator | CP | HTTP/TCP/Exec | Yes | Container orchestration |

---

## Best Practices

### 1. Use Health Checks

```python
@app.route('/health')
def health():
    return jsonify({
        "status": "healthy",
        "checks": {
            "database": check_db(),
            "redis": check_redis(),
            "disk_space": check_disk()
        }
    }), 200
```

---

### 2. Implement Graceful Shutdown

```python
import signal
import sys

def graceful_shutdown(signum, frame):
    # Deregister from service registry
    consul_client.agent.service.deregister('payment-1')

    # Stop accepting new requests
    server.shutdown()

    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

---

### 3. Cache Service Locations

```python
from functools import lru_cache
import time

@lru_cache(maxsize=128)
def get_service_instances(service_name, ttl=30):
    _, services = consul_client.health.service(service_name, passing=True)
    return services

# Cache expires after 30 seconds
```

---

### 4. Handle Failures Gracefully

```python
def call_payment_service(data):
    instances = get_service_instances('payment-service')

    for instance in instances:
        try:
            url = f"http://{instance['Address']}:{instance['Port']}/charge"
            response = requests.post(url, json=data, timeout=5)
            return response.json()
        except Exception as e:
            # Try next instance
            continue

    raise Exception("All payment service instances failed")
```

---

## Summary

✅ **Client-Side Discovery** - Client queries registry, controls load balancing

✅ **Server-Side Discovery** - Load balancer handles discovery and routing

✅ **Service Registry** - Consul, Eureka, etcd, Zookeeper

✅ **Health Checking** - HTTP, TCP, Script, TTL heartbeats

✅ **DNS Discovery** - Simple, universal, but limited control

✅ **Service Mesh** - Advanced discovery with Istio, Linkerd

✅ **Self vs Third-Party Registration** - Service registers vs external registrar

✅ **Graceful shutdown** - Deregister before terminating

✅ **Cache service locations** - Reduce registry load

✅ **Handle failures** - Retry with different instances

**Best Practices:**
- Use health checks to avoid routing to unhealthy instances
- Implement graceful shutdown to deregister cleanly
- Cache service locations with TTL to reduce registry load
- Handle failures by trying alternative instances
- Choose registry based on consistency needs (CP vs AP)
