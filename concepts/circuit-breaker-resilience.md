# Circuit Breaker & Resilience Patterns

## Overview

**Circuit breakers** and **resilience patterns** prevent cascading failures in distributed systems. When a service fails, these patterns ensure the failure doesn't bring down the entire system.

## Why Resilience Patterns?

### Problems Without Resilience

❌ **Cascading failures** - One service failure takes down entire system
❌ **Resource exhaustion** - Threads blocked waiting for failed service
❌ **Slow responses** - Users wait for timeout on every failed request
❌ **No recovery** - System doesn't automatically recover
❌ **Thundering herd** - All requests hit recovering service simultaneously

### Goals

✅ **Fail fast** - Don't wait for timeouts on known failures
✅ **Prevent cascades** - Isolate failures to single service
✅ **Preserve resources** - Don't block threads on failed calls
✅ **Auto-recovery** - Automatically test if service recovers
✅ **Graceful degradation** - Provide fallback responses

---

## Circuit Breaker Pattern

**Automatically "opens" to stop requests to failing service, then "closes" when service recovers.**

### Circuit States

```
┌──────────┐
│  CLOSED  │ ← Normal operation (requests allowed)
└────┬─────┘
     │ Failure threshold exceeded
     ↓
┌──────────┐
│   OPEN   │ ← Fail fast (requests rejected immediately)
└────┬─────┘
     │ After timeout period
     ↓
┌──────────┐
│HALF-OPEN │ ← Test if service recovered (limited requests)
└────┬─────┘
     │
     ├─→ Success → CLOSED
     └─→ Failure → OPEN
```

---

### State Details

#### 1. CLOSED (Normal)

- ✅ All requests pass through
- ✅ Track failure rate
- ✅ If failures exceed threshold → OPEN

**Example:** 5 failures in 10 seconds → OPEN

---

#### 2. OPEN (Blocking)

- ❌ All requests fail immediately (no network call)
- ❌ Returns error or fallback response
- ⏱️ After timeout (e.g., 30 seconds) → HALF-OPEN

---

#### 3. HALF-OPEN (Testing)

- ⚠️ Allow limited requests through (e.g., 1 request)
- If success → CLOSED (service recovered)
- If failure → OPEN (service still down)

---

### Implementation

**Basic Circuit Breaker in Python:**

```python
import time
from enum import Enum
from threading import Lock

class CircuitState(Enum):
    CLOSED = 1
    OPEN = 2
    HALF_OPEN = 3

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60, recovery_timeout=30):
        self.failure_threshold = failure_threshold  # Max failures before opening
        self.timeout = timeout  # Time window for counting failures
        self.recovery_timeout = recovery_timeout  # Time before trying half-open

        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
        self.lock = Lock()

    def call(self, func, *args, **kwargs):
        with self.lock:
            if self.state == CircuitState.OPEN:
                # Check if we should try half-open
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                    print("Circuit HALF-OPEN: Testing service")
                else:
                    raise Exception("Circuit breaker OPEN: Service unavailable")

        # Execute function
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        with self.lock:
            if self.state == CircuitState.HALF_OPEN:
                print("Circuit CLOSED: Service recovered")
                self.state = CircuitState.CLOSED

            self.failure_count = 0
            self.last_failure_time = None

    def _on_failure(self):
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()

            if self.state == CircuitState.HALF_OPEN:
                print("Circuit OPEN: Service still failing")
                self.state = CircuitState.OPEN
            elif self.failure_count >= self.failure_threshold:
                print(f"Circuit OPEN: {self.failure_count} failures detected")
                self.state = CircuitState.OPEN
```

**Usage:**

```python
import requests

# Create circuit breaker for payment service
payment_circuit = CircuitBreaker(failure_threshold=5, recovery_timeout=30)

def call_payment_service(amount):
    # This will be protected by circuit breaker
    response = requests.post("http://payment-service/charge", json={"amount": amount})
    return response.json()

# Use circuit breaker
try:
    result = payment_circuit.call(call_payment_service, 100)
    print(f"Payment successful: {result}")
except Exception as e:
    print(f"Payment failed: {e}")
    # Return fallback response
    return {"status": "pending", "message": "Payment service unavailable"}
```

---

### Advanced: Hystrix-Style Circuit Breaker

**Netflix Hystrix pattern with sliding window.**

```python
import time
from collections import deque
from threading import Lock

class HystrixCircuitBreaker:
    def __init__(self,
                 failure_threshold=50,  # 50% failure rate
                 request_volume_threshold=20,  # Min 20 requests in window
                 window_size=10,  # 10 second window
                 sleep_window=5):  # 5 seconds before half-open

        self.failure_threshold = failure_threshold
        self.request_volume_threshold = request_volume_threshold
        self.window_size = window_size
        self.sleep_window = sleep_window

        self.state = CircuitState.CLOSED
        self.requests = deque()  # Sliding window of (timestamp, success)
        self.last_open_time = None
        self.lock = Lock()

    def call(self, func, *args, **kwargs):
        with self.lock:
            self._remove_old_requests()

            if self.state == CircuitState.OPEN:
                if time.time() - self.last_open_time > self.sleep_window:
                    self.state = CircuitState.HALF_OPEN
                else:
                    raise Exception("Circuit breaker OPEN")

            if self.state == CircuitState.HALF_OPEN:
                # Only allow one request in half-open
                if len([r for r in self.requests if r[0] > time.time() - 1]) > 0:
                    raise Exception("Circuit breaker HALF-OPEN: Testing in progress")

        # Execute function
        try:
            result = func(*args, **kwargs)
            self._record_success()
            return result
        except Exception as e:
            self._record_failure()
            raise e

    def _record_success(self):
        with self.lock:
            self.requests.append((time.time(), True))

            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.CLOSED
                print("Circuit CLOSED: Service recovered")

    def _record_failure(self):
        with self.lock:
            self.requests.append((time.time(), False))

            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.OPEN
                self.last_open_time = time.time()
                return

            # Check if we should open
            total = len(self.requests)
            if total >= self.request_volume_threshold:
                failures = sum(1 for _, success in self.requests if not success)
                failure_rate = (failures / total) * 100

                if failure_rate >= self.failure_threshold:
                    self.state = CircuitState.OPEN
                    self.last_open_time = time.time()
                    print(f"Circuit OPEN: {failure_rate:.1f}% failure rate")

    def _remove_old_requests(self):
        cutoff = time.time() - self.window_size
        while self.requests and self.requests[0][0] < cutoff:
            self.requests.popleft()
```

---

## Retry Pattern

**Automatically retry failed requests with exponential backoff.**

### Simple Retry

```python
import time

def retry(func, max_attempts=3, delay=1):
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts - 1:
                raise e
            print(f"Attempt {attempt + 1} failed, retrying in {delay}s...")
            time.sleep(delay)

# Usage
result = retry(lambda: requests.get("http://api.example.com/data"))
```

---

### Exponential Backoff

**Increase delay between retries exponentially.**

```python
import time
import random

def exponential_backoff_retry(func, max_attempts=5, base_delay=1, max_delay=60):
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts - 1:
                raise e

            # Calculate delay: base_delay * 2^attempt + jitter
            delay = min(base_delay * (2 ** attempt), max_delay)
            jitter = random.uniform(0, delay * 0.1)  # Add 10% jitter
            total_delay = delay + jitter

            print(f"Attempt {attempt + 1} failed, retrying in {total_delay:.2f}s...")
            time.sleep(total_delay)

# Usage
result = exponential_backoff_retry(
    lambda: requests.post("http://payment-service/charge", json={"amount": 100}),
    max_attempts=5,
    base_delay=1
)
```

**Delay progression:**
- Attempt 1: 1s
- Attempt 2: 2s
- Attempt 3: 4s
- Attempt 4: 8s
- Attempt 5: 16s

**Jitter:** Random variation to prevent thundering herd (all clients retrying at same time)

---

### Retry with Circuit Breaker

```python
class ResilientClient:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker(failure_threshold=5)

    def call_with_retry(self, func, max_attempts=3):
        def wrapped():
            return exponential_backoff_retry(func, max_attempts=max_attempts)

        return self.circuit_breaker.call(wrapped)

# Usage
client = ResilientClient()
result = client.call_with_retry(
    lambda: requests.get("http://unreliable-service/data")
)
```

---

## Timeout Pattern

**Don't wait forever for slow services.**

```python
import requests
from requests.exceptions import Timeout

try:
    response = requests.get(
        "http://slow-service/data",
        timeout=5  # 5 second timeout
    )
except Timeout:
    print("Request timed out")
    # Return cached data or default response
    return get_cached_data()
```

---

## Bulkhead Pattern

**Isolate resources to prevent one failing service from exhausting all resources.**

### Thread Pool Isolation

**Separate thread pools for each service.**

```python
from concurrent.futures import ThreadPoolExecutor, TimeoutError
import time

class BulkheadExecutor:
    def __init__(self):
        # Separate thread pools for each service
        self.payment_pool = ThreadPoolExecutor(max_workers=10, thread_name_prefix="payment")
        self.inventory_pool = ThreadPoolExecutor(max_workers=5, thread_name_prefix="inventory")
        self.email_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="email")

    def call_payment(self, func, *args, **kwargs):
        future = self.payment_pool.submit(func, *args, **kwargs)
        try:
            return future.result(timeout=5)
        except TimeoutError:
            raise Exception("Payment service timeout")

    def call_inventory(self, func, *args, **kwargs):
        future = self.inventory_pool.submit(func, *args, **kwargs)
        try:
            return future.result(timeout=3)
        except TimeoutError:
            raise Exception("Inventory service timeout")

# Usage
bulkhead = BulkheadExecutor()

def process_order(order):
    # Even if payment service is slow/failing, inventory calls still work
    payment_result = bulkhead.call_payment(charge_payment, order.total)
    inventory_result = bulkhead.call_inventory(reserve_inventory, order.items)
```

**Benefit:** If payment service hangs, it only blocks 10 threads (payment pool), not entire system.

---

### Semaphore-Based Bulkhead

**Limit concurrent requests per service.**

```python
from threading import Semaphore

class SemaphoreBulkhead:
    def __init__(self):
        self.payment_semaphore = Semaphore(10)  # Max 10 concurrent payment calls
        self.inventory_semaphore = Semaphore(5)  # Max 5 concurrent inventory calls

    def call_payment(self, func, *args, **kwargs):
        if not self.payment_semaphore.acquire(blocking=False):
            raise Exception("Payment service at max capacity")

        try:
            return func(*args, **kwargs)
        finally:
            self.payment_semaphore.release()
```

---

## Fallback Pattern

**Provide default response when service fails.**

```python
def get_user_recommendations(user_id):
    try:
        # Try personalized recommendations
        response = requests.get(
            f"http://recommendation-service/users/{user_id}/recommendations",
            timeout=2
        )
        return response.json()
    except Exception as e:
        print(f"Recommendation service failed: {e}")
        # Fallback: Return popular items
        return get_popular_items()

def get_popular_items():
    # Return cached popular items
    return [
        {"id": 1, "name": "Popular Item 1"},
        {"id": 2, "name": "Popular Item 2"},
        {"id": 3, "name": "Popular Item 3"}
    ]
```

---

## Rate Limiting (Client-Side)

**Prevent overwhelming downstream services.**

```python
import time
from threading import Lock

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.refill_rate = refill_rate  # Tokens per second
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = Lock()

    def acquire(self):
        with self.lock:
            self._refill()

            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate

        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now

# Usage
rate_limiter = TokenBucket(capacity=10, refill_rate=2)  # 2 requests/second

def call_api():
    if rate_limiter.acquire():
        return requests.get("http://api.example.com/data")
    else:
        raise Exception("Rate limit exceeded")
```

---

## Combining Patterns

**Production-ready resilient client.**

```python
import requests
from typing import Callable, Any

class ResilientHTTPClient:
    def __init__(self, service_name):
        self.service_name = service_name
        self.circuit_breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=30)
        self.rate_limiter = TokenBucket(capacity=100, refill_rate=10)

    def call(self,
             url: str,
             fallback: Callable = None,
             max_retries: int = 3,
             timeout: int = 5) -> Any:

        # 1. Check rate limit
        if not self.rate_limiter.acquire():
            raise Exception(f"{self.service_name}: Rate limit exceeded")

        # 2. Define request function
        def make_request():
            return requests.get(url, timeout=timeout)

        # 3. Wrap with retry
        def with_retry():
            return exponential_backoff_retry(make_request, max_attempts=max_retries)

        # 4. Wrap with circuit breaker
        try:
            return self.circuit_breaker.call(with_retry)
        except Exception as e:
            # 5. Use fallback if available
            if fallback:
                print(f"{self.service_name} failed, using fallback: {e}")
                return fallback()
            raise e

# Usage
payment_client = ResilientHTTPClient("payment-service")

result = payment_client.call(
    url="http://payment-service/charge",
    fallback=lambda: {"status": "pending"},
    max_retries=3,
    timeout=5
)
```

---

## Health Checks

**Monitor service health and dependencies.**

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/health')
def health():
    checks = {
        "database": check_database(),
        "redis": check_redis(),
        "payment_service": check_payment_service()
    }

    all_healthy = all(checks.values())
    status_code = 200 if all_healthy else 503

    return jsonify({
        "status": "healthy" if all_healthy else "unhealthy",
        "checks": checks
    }), status_code

def check_database():
    try:
        db.execute("SELECT 1")
        return True
    except:
        return False

def check_redis():
    try:
        redis_client.ping()
        return True
    except:
        return False

def check_payment_service():
    try:
        response = requests.get("http://payment-service/health", timeout=2)
        return response.status_code == 200
    except:
        return False
```

---

## Graceful Degradation

**Reduce functionality instead of complete failure.**

```python
def get_product_details(product_id):
    product = db.get_product(product_id)

    # Core data (required)
    result = {
        "id": product.id,
        "name": product.name,
        "price": product.price
    }

    # Try to add recommendations (optional)
    try:
        recommendations = requests.get(
            f"http://recommendation-service/products/{product_id}/similar",
            timeout=1
        ).json()
        result["recommendations"] = recommendations
    except:
        # Gracefully degrade: Continue without recommendations
        result["recommendations"] = []

    # Try to add reviews (optional)
    try:
        reviews = requests.get(
            f"http://review-service/products/{product_id}/reviews",
            timeout=1
        ).json()
        result["reviews"] = reviews
    except:
        # Gracefully degrade: Continue without reviews
        result["reviews"] = []

    return result
```

---

## Real-World Example: E-commerce Order

```python
class OrderService:
    def __init__(self):
        self.payment_client = ResilientHTTPClient("payment")
        self.inventory_client = ResilientHTTPClient("inventory")
        self.email_client = ResilientHTTPClient("email")

    def create_order(self, order_data):
        # 1. Reserve inventory (critical)
        try:
            inventory_result = self.inventory_client.call(
                url=f"http://inventory-service/reserve",
                max_retries=3,
                timeout=5
            )
        except Exception as e:
            # Inventory failed - cannot proceed
            return {"error": "Inventory service unavailable", "status": "failed"}

        # 2. Charge payment (critical)
        try:
            payment_result = self.payment_client.call(
                url=f"http://payment-service/charge",
                max_retries=3,
                timeout=10,
                fallback=lambda: {"status": "pending", "async": True}
            )
        except Exception as e:
            # Release inventory
            self.inventory_client.call(
                url=f"http://inventory-service/release/{inventory_result['id']}",
                max_retries=1
            )
            return {"error": "Payment service unavailable", "status": "failed"}

        # 3. Send confirmation email (non-critical)
        try:
            self.email_client.call(
                url=f"http://email-service/send",
                max_retries=1,
                timeout=3,
                fallback=lambda: {"status": "queued"}  # Queue for later
            )
        except Exception as e:
            # Email failed, but order succeeded
            print(f"Email failed (non-critical): {e}")

        return {
            "status": "success",
            "order_id": generate_order_id(),
            "payment": payment_result,
            "inventory": inventory_result
        }
```

---

## Monitoring & Observability

**Track circuit breaker metrics.**

```python
import prometheus_client as prom

# Metrics
circuit_state_gauge = prom.Gauge(
    'circuit_breaker_state',
    'Circuit breaker state (0=closed, 1=open, 2=half-open)',
    ['service']
)

circuit_failures_counter = prom.Counter(
    'circuit_breaker_failures_total',
    'Total failures detected by circuit breaker',
    ['service']
)

class ObservableCircuitBreaker(CircuitBreaker):
    def __init__(self, service_name, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.service_name = service_name

    def _on_failure(self):
        super()._on_failure()
        circuit_failures_counter.labels(service=self.service_name).inc()

        if self.state == CircuitState.OPEN:
            circuit_state_gauge.labels(service=self.service_name).set(1)

    def _on_success(self):
        super()._on_success()

        if self.state == CircuitState.CLOSED:
            circuit_state_gauge.labels(service=self.service_name).set(0)
```

---

## Libraries & Tools

### Python

**1. Pybreaker**
```python
from pybreaker import CircuitBreaker

breaker = CircuitBreaker(fail_max=5, timeout_duration=60)

@breaker
def call_external_service():
    return requests.get("http://api.example.com/data")
```

**2. Tenacity (Retry)**
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10)
)
def call_api():
    return requests.get("http://api.example.com/data")
```

---

### Java

**1. Resilience4j**
```java
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("payment-service");

Supplier<String> decoratedSupplier = CircuitBreaker
    .decorateSupplier(circuitBreaker, () -> callPaymentService());

String result = decoratedSupplier.get();
```

**2. Hystrix (Deprecated, but widely used)**
```java
public class PaymentCommand extends HystrixCommand<String> {
    protected String run() {
        return callPaymentService();
    }

    protected String getFallback() {
        return "Payment pending";
    }
}

String result = new PaymentCommand().execute();
```

---

### Go

**1. gobreaker**
```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 3,
    Interval:    60 * time.Second,
    Timeout:     30 * time.Second,
})

result, err := cb.Execute(func() (interface{}, error) {
    return callPaymentService()
})
```

---

## Summary

✅ **Circuit Breaker** - Fail fast when service is down, auto-recover
✅ **Retry with Exponential Backoff** - Retry transient failures with increasing delays
✅ **Timeout** - Don't wait forever for slow services
✅ **Bulkhead** - Isolate thread pools to prevent resource exhaustion
✅ **Fallback** - Provide default response when service fails
✅ **Rate Limiting** - Prevent overwhelming downstream services
✅ **Health Checks** - Monitor service and dependency health
✅ **Graceful Degradation** - Reduce functionality instead of complete failure
✅ **Combine patterns** - Use circuit breaker + retry + timeout + fallback together
✅ **Monitor metrics** - Track circuit state, failure rates, latency

**Best Practices:**
- Use circuit breaker for all external service calls
- Implement exponential backoff with jitter for retries
- Set appropriate timeouts (don't use default infinite timeout)
- Isolate critical vs non-critical dependencies
- Provide fallback responses for better UX
- Monitor circuit breaker state changes
- Test failure scenarios regularly (chaos engineering)
