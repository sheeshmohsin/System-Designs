# Distributed Transactions

## Overview

A **distributed transaction** is a transaction that spans multiple databases, services, or systems. Ensuring **ACID properties** (Atomicity, Consistency, Isolation, Durability) across distributed systems is one of the hardest problems in system design.

## Why Distributed Transactions?

### Problems in Distributed Systems

❌ **Partial failures** - One service succeeds, another fails
❌ **Data inconsistency** - Different services have different data
❌ **Network partitions** - Services can't communicate
❌ **Lost updates** - Concurrent updates conflict

### Goals

✅ **Atomicity** - All operations succeed or all fail
✅ **Consistency** - Data remains valid across systems
✅ **Isolation** - Concurrent transactions don't interfere
✅ **Durability** - Committed changes persist

---

## The Problem: Distributed Transaction Example

**E-commerce Order System:**

```
1. Deduct inventory (Inventory Service)
2. Charge payment (Payment Service)
3. Create order (Order Service)
4. Send confirmation email (Email Service)
```

**What if payment succeeds but inventory deduction fails?**
- Money charged ✅
- Product not reserved ❌
- → Customer charged for nothing! 💥

---

## Solution Patterns

### 1. Two-Phase Commit (2PC)

**Classic distributed transaction protocol** - all participants agree before committing.

#### How 2PC Works

**Phase 1: Prepare (Voting)**
```
Coordinator: "Can you all commit?"
Service A: "Yes, I can commit"
Service B: "Yes, I can commit"
Service C: "No, I have an error"
```

**Phase 2: Commit or Abort**
```
Coordinator: "Abort" (because Service C said no)
Service A: Rollback
Service B: Rollback
Service C: Rollback
```

#### Visual Flow

```
Coordinator
    ↓
1. PREPARE →  [Service A]  [Service B]  [Service C]
                 ↓ Yes       ↓ Yes        ↓ No
2. ABORT   →  [Rollback]  [Rollback]  [Rollback]
```

#### Implementation Example

**Coordinator:**
```python
class TwoPhaseCommitCoordinator:
    def __init__(self, participants):
        self.participants = participants  # List of services
        self.transaction_id = generate_id()

    def execute_transaction(self, operations):
        # Phase 1: Prepare
        prepared = []
        for participant, operation in zip(self.participants, operations):
            response = participant.prepare(self.transaction_id, operation)

            if response == "VOTE_COMMIT":
                prepared.append(participant)
            else:
                # Abort if any participant can't commit
                self.abort(prepared)
                return False

        # Phase 2: Commit
        return self.commit(prepared)

    def commit(self, participants):
        for participant in participants:
            try:
                participant.commit(self.transaction_id)
            except Exception as e:
                # Coordinator failure - transaction in limbo!
                log_error(f"Failed to commit {participant}: {e}")
                return False
        return True

    def abort(self, participants):
        for participant in participants:
            try:
                participant.abort(self.transaction_id)
            except Exception as e:
                log_error(f"Failed to abort {participant}: {e}")
```

**Participant (Service):**
```python
class Participant:
    def __init__(self):
        self.prepared_transactions = {}

    def prepare(self, transaction_id, operation):
        try:
            # Validate operation
            if not self.can_execute(operation):
                return "VOTE_ABORT"

            # Lock resources
            self.lock_resources(operation)

            # Store operation for later commit
            self.prepared_transactions[transaction_id] = operation

            # Write to log (for recovery)
            self.write_log(transaction_id, "PREPARED")

            return "VOTE_COMMIT"

        except Exception as e:
            return "VOTE_ABORT"

    def commit(self, transaction_id):
        operation = self.prepared_transactions.get(transaction_id)

        if operation:
            # Execute operation
            self.execute(operation)

            # Release locks
            self.release_locks(operation)

            # Write to log
            self.write_log(transaction_id, "COMMITTED")

            del self.prepared_transactions[transaction_id]

    def abort(self, transaction_id):
        if transaction_id in self.prepared_transactions:
            operation = self.prepared_transactions[transaction_id]

            # Release locks
            self.release_locks(operation)

            # Write to log
            self.write_log(transaction_id, "ABORTED")

            del self.prepared_transactions[transaction_id]
```

#### Pros & Cons

**Pros:**
- ✅ Strong consistency (ACID compliant)
- ✅ All-or-nothing guarantee

**Cons:**
- ❌ **Blocking** - Participants wait for coordinator
- ❌ **Single point of failure** - Coordinator failure leaves system in limbo
- ❌ **Poor performance** - Multiple round trips
- ❌ **Locks held during network calls** - Deadlocks possible

**Rarely used in practice** due to blocking nature.

---

### 2. Saga Pattern

**Long-running transactions** as a sequence of local transactions with compensations.

#### How Saga Works

Instead of one distributed transaction, execute a series of local transactions. If one fails, run **compensating transactions** to undo previous steps.

**Example: Order Processing Saga**

**Success Flow:**
```
1. Reserve inventory     ✅
2. Charge payment        ✅
3. Create order          ✅
4. Send confirmation     ✅
```

**Failure Flow (Payment Fails):**
```
1. Reserve inventory     ✅
2. Charge payment        ❌ (FAILED)
3. Compensate: Release inventory ✅ (Undo step 1)
```

#### Saga Coordination Patterns

**Pattern A: Choreography (Event-Driven)**

Services listen to events and react accordingly.

```
Order Service
    ↓ (publishes OrderCreated)
Inventory Service
    ↓ (reserves inventory, publishes InventoryReserved)
Payment Service
    ↓ (charges payment)
    → If success: publishes PaymentCompleted
    → If failure: publishes PaymentFailed

Inventory Service (listens to PaymentFailed)
    ↓ (releases inventory)
```

**Implementation:**
```python
# Order Service
def create_order(order_data):
    order = db.create_order(order_data)
    event_bus.publish("OrderCreated", {
        "order_id": order.id,
        "items": order.items
    })
    return order

# Inventory Service
@event_bus.subscribe("OrderCreated")
def reserve_inventory(event):
    try:
        inventory.reserve(event["items"])
        event_bus.publish("InventoryReserved", {
            "order_id": event["order_id"]
        })
    except Exception:
        event_bus.publish("InventoryReservationFailed", {
            "order_id": event["order_id"]
        })

# Payment Service
@event_bus.subscribe("InventoryReserved")
def charge_payment(event):
    try:
        payment.charge(event["order_id"])
        event_bus.publish("PaymentCompleted", {
            "order_id": event["order_id"]
        })
    except Exception:
        event_bus.publish("PaymentFailed", {
            "order_id": event["order_id"]
        })

# Inventory Service (Compensation)
@event_bus.subscribe("PaymentFailed")
def release_inventory(event):
    inventory.release(event["order_id"])
```

**Pros:**
- ✅ Simple
- ✅ Decoupled services

**Cons:**
- ❌ Hard to track overall state
- ❌ Cyclic dependencies possible
- ❌ Difficult to debug

---

**Pattern B: Orchestration (Centralized Coordinator)**

A coordinator service manages the saga workflow.

```
Saga Coordinator
    ↓
1. Call Inventory Service → Reserve
    ↓ Success
2. Call Payment Service → Charge
    ↓ Failure
3. Call Inventory Service → Release (Compensate)
```

**Implementation:**
```python
class OrderSagaOrchestrator:
    def __init__(self):
        self.inventory_service = InventoryService()
        self.payment_service = PaymentService()
        self.order_service = OrderService()

    def create_order(self, order_data):
        saga_state = {
            "order_id": None,
            "inventory_reserved": False,
            "payment_charged": False
        }

        try:
            # Step 1: Reserve inventory
            inventory_id = self.inventory_service.reserve(order_data["items"])
            saga_state["inventory_reserved"] = True

            try:
                # Step 2: Charge payment
                payment_id = self.payment_service.charge(order_data["payment"])
                saga_state["payment_charged"] = True

                try:
                    # Step 3: Create order
                    order = self.order_service.create(order_data)
                    saga_state["order_id"] = order.id

                    return {"success": True, "order_id": order.id}

                except Exception as e:
                    # Compensate: Refund payment
                    self.payment_service.refund(payment_id)
                    raise

            except Exception as e:
                # Compensate: Release inventory
                self.inventory_service.release(inventory_id)
                raise

        except Exception as e:
            # All compensations completed
            return {"success": False, "error": str(e)}
```

**Pros:**
- ✅ Clear workflow
- ✅ Easy to track state
- ✅ Easier to debug

**Cons:**
- ❌ Coordinator is single point of failure
- ❌ More complex coordinator logic

---

#### Compensating Transactions

**Important:** Compensations may not be exact inverses!

**Examples:**

| Original Transaction | Compensation | Notes |
|---------------------|--------------|-------|
| Reserve inventory | Release inventory | ✅ Exact inverse |
| Charge $100 | Refund $100 | ✅ Exact inverse |
| Send email | (Cannot unsend) | ❌ Send apology email instead |
| Update external API | Call undo API | ✅ If API supports it |

**Semantic Compensation:**
```python
# Original: Send order confirmation email
send_email(user, "Your order #123 is confirmed")

# Compensation: Send cancellation email
send_email(user, "Your order #123 has been cancelled")
```

---

### 3. Try-Confirm/Cancel (TCC)

**Two-phase commit alternative** with business logic in each phase.

#### How TCC Works

**Phase 1: Try**
- Reserve resources
- Don't make permanent changes

**Phase 2: Confirm or Cancel**
- **Confirm:** Make changes permanent
- **Cancel:** Release reservations

#### Example: Money Transfer

**Try Phase:**
```python
def try_transfer(from_account, to_account, amount):
    # Reserve money (don't actually transfer yet)
    db.execute("""
        UPDATE accounts
        SET reserved = reserved + ?
        WHERE id = ? AND balance >= ?
    """, [amount, from_account, amount])

    return "TRY_SUCCESS"
```

**Confirm Phase:**
```python
def confirm_transfer(from_account, to_account, amount):
    # Actually transfer money
    db.execute("""
        UPDATE accounts
        SET balance = balance - ?,
            reserved = reserved - ?
        WHERE id = ?
    """, [amount, amount, from_account])

    db.execute("""
        UPDATE accounts
        SET balance = balance + ?
        WHERE id = ?
    """, [amount, to_account])

    return "CONFIRMED"
```

**Cancel Phase:**
```python
def cancel_transfer(from_account, amount):
    # Release reserved money
    db.execute("""
        UPDATE accounts
        SET reserved = reserved - ?
        WHERE id = ?
    """, [amount, from_account])

    return "CANCELLED"
```

**Pros:**
- ✅ No blocking (unlike 2PC)
- ✅ Explicit business logic

**Cons:**
- ❌ Requires schema changes (reserved fields)
- ❌ More complex than Saga

---

### 4. Event Sourcing

**Store all state changes as events** instead of current state.

#### How Event Sourcing Works

Instead of:
```sql
UPDATE accounts SET balance = 150 WHERE id = 123
```

Store events:
```json
{"event": "MoneyDeposited", "account": 123, "amount": 50, "timestamp": "..."}
{"event": "MoneyWithdrawn", "account": 123, "amount": 20, "timestamp": "..."}
```

**Current state = replay all events**

#### Implementation

**Event Store:**
```python
class EventStore:
    def append(self, aggregate_id, event):
        db.execute("""
            INSERT INTO events (aggregate_id, event_type, data, sequence)
            VALUES (?, ?, ?, ?)
        """, [aggregate_id, event["type"], json.dumps(event["data"]), self.get_next_sequence(aggregate_id)])

    def get_events(self, aggregate_id):
        return db.query("""
            SELECT event_type, data
            FROM events
            WHERE aggregate_id = ?
            ORDER BY sequence
        """, [aggregate_id])
```

**Rebuilding State:**
```python
def get_account_balance(account_id):
    events = event_store.get_events(account_id)
    balance = 0

    for event in events:
        if event["event_type"] == "MoneyDeposited":
            balance += event["data"]["amount"]
        elif event["event_type"] == "MoneyWithdrawn":
            balance -= event["data"]["amount"]

    return balance
```

**Pros:**
- ✅ Complete audit trail
- ✅ Can replay to any point in time
- ✅ Easy to add new projections

**Cons:**
- ❌ More complex queries
- ❌ Event schema evolution
- ❌ Can't delete data (GDPR concerns)

**Used in:** Financial systems, audit-heavy applications

---

### 5. Outbox Pattern

**Ensure database writes and message publishing are atomic.**

#### The Problem

```python
def create_order(order_data):
    # Write to database
    order = db.create_order(order_data)

    # Publish event
    message_queue.publish("OrderCreated", order)  # What if this fails?
```

If message queue publish fails, order is created but event not sent!

#### Outbox Solution

**Write both to same database transaction:**

```python
def create_order(order_data):
    with db.transaction():
        # Insert order
        order = db.execute("""
            INSERT INTO orders (user_id, total) VALUES (?, ?)
        """, [order_data["user_id"], order_data["total"]])

        # Insert outbox message (same transaction)
        db.execute("""
            INSERT INTO outbox (aggregate_id, event_type, payload)
            VALUES (?, ?, ?)
        """, [order.id, "OrderCreated", json.dumps(order.to_dict())])

    return order
```

**Background worker publishes from outbox:**
```python
def outbox_worker():
    while True:
        # Get unpublished messages
        messages = db.query("""
            SELECT * FROM outbox WHERE published = false
            ORDER BY created_at LIMIT 100
        """)

        for msg in messages:
            try:
                # Publish to message queue
                message_queue.publish(msg["event_type"], msg["payload"])

                # Mark as published
                db.execute("""
                    UPDATE outbox SET published = true WHERE id = ?
                """, [msg["id"]])

            except Exception as e:
                log_error(f"Failed to publish {msg['id']}: {e}")
                # Will retry on next iteration

        time.sleep(1)
```

**Pros:**
- ✅ Guaranteed delivery (eventually)
- ✅ Atomic database + message queue
- ✅ Idempotent (can retry safely)

**Cons:**
- ❌ Eventual consistency
- ❌ Requires outbox cleanup

---

## Comparison Table

| Pattern | **Consistency** | **Performance** | **Complexity** | **Blocking** | **Best For** |
|---------|----------------|----------------|---------------|-------------|-------------|
| **2PC** | Strong | Poor | Medium | Yes | Small, critical transactions |
| **Saga** | Eventual | Good | High | No | Long-running workflows |
| **TCC** | Strong | Medium | High | No | Explicit reservations |
| **Event Sourcing** | Eventual | Good | High | No | Audit trails |
| **Outbox** | Eventual | Good | Low | No | Database + messaging |

---

## Real-World Examples

### E-commerce Order (Saga)

```
1. Reserve inventory
2. Charge payment
3. Create order
4. Send confirmation

If payment fails → Release inventory
```

### Bank Transfer (2PC or TCC)

```
1. Deduct from Account A
2. Add to Account B

Both must succeed or both fail
```

### Ride-Sharing (Saga)

```
1. Match driver
2. Reserve driver
3. Charge rider
4. Notify driver

If charge fails → Release driver
```

---

## Best Practices

### 1. Idempotency

**Make operations safe to retry:**

```python
def reserve_inventory(order_id, items):
    # Check if already reserved
    if db.exists("SELECT * FROM reservations WHERE order_id = ?", [order_id]):
        return "ALREADY_RESERVED"

    # Reserve
    db.insert_reservation(order_id, items)
    return "RESERVED"
```

---

### 2. Timeouts & Retry

**Don't wait forever:**

```python
try:
    response = service.call(timeout=5000)  # 5 second timeout
except TimeoutError:
    # Log and retry or compensate
    compensate()
```

---

### 3. Monitoring

**Track saga state:**

```python
class SagaMonitor:
    def record_step(self, saga_id, step, status):
        db.execute("""
            INSERT INTO saga_log (saga_id, step, status, timestamp)
            VALUES (?, ?, ?, ?)
        """, [saga_id, step, status, now()])
```

---

## Summary

✅ **2PC** - Strong consistency but blocking (rarely used)
✅ **Saga** - Most common pattern for distributed transactions
✅ **Choreography** - Event-driven sagas (decoupled)
✅ **Orchestration** - Coordinator-based sagas (easier to manage)
✅ **TCC** - Explicit try/confirm/cancel phases
✅ **Event Sourcing** - Complete audit trail
✅ **Outbox Pattern** - Atomic database + messaging
✅ **Idempotency** - Make operations safe to retry
✅ **Compensations** - Undo operations when failures occur

**Golden Rule:** Prefer Saga pattern with compensating transactions over 2PC!
