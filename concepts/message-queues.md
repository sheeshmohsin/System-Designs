# Message Queues

## Overview

A **message queue** is an asynchronous communication pattern where messages are stored in a queue until the receiver processes them. It decouples producers (senders) from consumers (receivers).

## Why Use Message Queues?

### Problems Without Message Queues

❌ **Tight coupling** - Services directly depend on each other
❌ **Synchronous blocking** - Sender waits for receiver to process
❌ **Lost messages** - If receiver is down, messages are lost
❌ **Traffic spikes** - System can't handle sudden load

### Benefits of Message Queues

✅ **Decoupling** - Services don't need to know about each other
✅ **Asynchronous processing** - Sender doesn't wait for receiver
✅ **Reliability** - Messages persist until processed
✅ **Load leveling** - Queue absorbs traffic spikes
✅ **Scalability** - Add more consumers to process faster

---

## How Message Queues Work

```
Producer → Message Queue → Consumer
   ↓           ↓              ↓
(Sends)    (Stores)      (Processes)
```

**Flow:**
1. **Producer** sends message to queue
2. **Queue** stores message (in memory or disk)
3. **Consumer** pulls message from queue
4. **Consumer** processes message
5. **Consumer** acknowledges completion
6. **Queue** deletes message

---

## Message Queue Patterns

### 1. Point-to-Point (Queue)

**One message → One consumer**

```
Producer → Queue → Consumer
```

**Characteristics:**
- Each message is consumed by **exactly one** consumer
- Messages are removed after consumption
- Multiple consumers can listen, but only one gets each message

**Use Cases:**
- Task processing (email sending, image processing)
- Job queues (background jobs)
- Order processing

**Example:**
```python
# Producer
queue.send("process_image", {"image_id": 123})

# Consumer 1 (gets the message)
message = queue.receive()
process_image(message.data["image_id"])
queue.acknowledge(message.id)

# Consumer 2 (doesn't get this message)
```

---

### 2. Publish-Subscribe (Pub/Sub)

**One message → Multiple consumers**

```
                    → Consumer 1
Publisher → Topic   → Consumer 2
                    → Consumer 3
```

**Characteristics:**
- Each message is delivered to **all subscribers**
- Messages are copied to each subscriber
- Subscribers are independent

**Use Cases:**
- Notifications (send to email, SMS, push)
- Event broadcasting (user signup → analytics, email, CRM)
- Real-time updates

**Example:**
```python
# Publisher
topic.publish("user_created", {"user_id": 123, "email": "john@example.com"})

# Subscriber 1: Email Service
def on_user_created(data):
    send_welcome_email(data["email"])

# Subscriber 2: Analytics Service
def on_user_created(data):
    track_signup(data["user_id"])

# Subscriber 3: CRM Service
def on_user_created(data):
    create_crm_record(data)
```

---

### 3. Request-Reply

**Asynchronous request with reply**

```
Requester → Request Queue → Worker
   ↑                           ↓
Reply Queue ← Reply ← Worker processes
```

**Use Cases:**
- Microservices communication
- RPC-like async operations

---

## Message Queue Components

### 1. Producer (Publisher)

Sends messages to the queue.

```python
# Example: Kafka Producer
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers='localhost:9092')
producer.send('video_processing', b'{"video_id": 123}')
```

---

### 2. Queue/Topic

Stores messages until consumed.

**Queue Types:**
- **In-memory** - Fast but not durable (Redis)
- **Disk-based** - Slower but durable (Kafka, RabbitMQ)

---

### 3. Consumer (Subscriber)

Receives and processes messages.

```python
# Example: Kafka Consumer
from kafka import KafkaConsumer

consumer = KafkaConsumer('video_processing')
for message in consumer:
    process_video(message.value)
```

---

## Message Delivery Guarantees

### 1. At-Most-Once

Message delivered **0 or 1 time** (may be lost).

**How:**
- Send message
- Don't wait for acknowledgment

**Pros:** Fast
**Cons:** Messages can be lost

**Use Case:** Metrics, logs (losing a few is okay)

---

### 2. At-Least-Once

Message delivered **1 or more times** (may duplicate).

**How:**
- Send message
- Wait for acknowledgment
- Retry if no acknowledgment

**Pros:** No message loss
**Cons:** Duplicates possible

**Use Case:** Most systems (handle duplicates with idempotency)

---

### 3. Exactly-Once

Message delivered **exactly 1 time** (no loss, no duplicates).

**How:**
- Transactions + deduplication
- Most complex to implement

**Pros:** Perfect reliability
**Cons:** Slower, complex

**Use Case:** Financial transactions, critical operations

---

## Message Queue Technologies

### 1. RabbitMQ

**General-purpose message broker** with flexible routing.

**Architecture:**
```
Producer → Exchange → Binding → Queue → Consumer
```

**Exchange Types:**
- **Direct** - Route by exact routing key
- **Fanout** - Broadcast to all queues
- **Topic** - Route by pattern matching
- **Headers** - Route by message headers

**Example:**
```python
import pika

# Producer
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
channel.queue_declare(queue='tasks')
channel.basic_publish(exchange='', routing_key='tasks', body='Hello')

# Consumer
def callback(ch, method, properties, body):
    print(f"Received: {body}")
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_consume(queue='tasks', on_message_callback=callback)
channel.start_consuming()
```

**Pros:**
- ✅ Flexible routing
- ✅ Easy to use
- ✅ Good for complex workflows

**Cons:**
- ❌ Not designed for high throughput
- ❌ Limited ordering guarantees

**Best For:** Task queues, microservices communication

---

### 2. Apache Kafka

**Distributed event streaming platform** for high throughput.

**Architecture:**
```
Producer → Topic (Partitions) → Consumer Group
```

**Key Concepts:**
- **Topic** - Category of messages
- **Partition** - Ordered, immutable sequence of messages
- **Offset** - Position in partition
- **Consumer Group** - Group of consumers sharing load

**Example:**
```python
# Producer
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers='localhost:9092')
producer.send('user_events', key=b'user123', value=b'{"action": "login"}')

# Consumer
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'user_events',
    group_id='analytics',
    bootstrap_servers='localhost:9092'
)

for message in consumer:
    print(f"Partition: {message.partition}, Offset: {message.offset}")
    process_event(message.value)
```

**Partitioning:**
```
Topic: user_events
├── Partition 0: [msg1, msg4, msg7]
├── Partition 1: [msg2, msg5, msg8]
└── Partition 2: [msg3, msg6, msg9]

Consumer Group:
├── Consumer A → Partition 0
├── Consumer B → Partition 1
└── Consumer C → Partition 2
```

**Pros:**
- ✅ Very high throughput (millions/sec)
- ✅ Message persistence (replay possible)
- ✅ Strong ordering per partition
- ✅ Scalable (distributed)

**Cons:**
- ❌ Complex setup
- ❌ Overkill for simple use cases

**Best For:** Event streaming, log aggregation, real-time analytics

---

### 3. AWS SQS (Simple Queue Service)

**Fully managed** queue service by AWS.

**Types:**
- **Standard Queue** - At-least-once, no ordering
- **FIFO Queue** - Exactly-once, strict ordering

**Example:**
```python
import boto3

sqs = boto3.client('sqs')

# Send message
sqs.send_message(
    QueueUrl='https://sqs.us-east-1.amazonaws.com/123/my-queue',
    MessageBody='{"task": "process_video"}'
)

# Receive message
messages = sqs.receive_message(QueueUrl='...', MaxNumberOfMessages=10)
for msg in messages['Messages']:
    process(msg['Body'])
    sqs.delete_message(QueueUrl='...', ReceiptHandle=msg['ReceiptHandle'])
```

**Pros:**
- ✅ Fully managed (no infrastructure)
- ✅ Auto-scaling
- ✅ Easy integration with AWS

**Cons:**
- ❌ Vendor lock-in
- ❌ Limited throughput (compared to Kafka)

**Best For:** AWS-based applications, simple queuing

---

### 4. Redis (Pub/Sub & Streams)

**In-memory** message broker (fast but not durable).

#### Redis Pub/Sub
```python
import redis

r = redis.Redis()

# Publisher
r.publish('notifications', 'New message!')

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe('notifications')
for message in pubsub.listen():
    print(message)
```

#### Redis Streams (better for persistence)
```python
# Producer
r.xadd('events', {'action': 'login', 'user': '123'})

# Consumer
messages = r.xread({'events': '0'})
```

**Pros:**
- ✅ Very fast (in-memory)
- ✅ Simple

**Cons:**
- ❌ Not durable (data loss on crash)
- ❌ Limited scalability

**Best For:** Real-time notifications, caching + pub/sub

---

## Message Queue Patterns in System Design

### 1. Asynchronous Task Processing

**Problem:** Video upload takes 5 minutes to transcode.

**Without Queue:**
```
User uploads video → Server transcodes (5 min) → User waits
```

**With Queue:**
```
User uploads video → Queue task → Return immediately
Background worker → Process from queue → User notified when done
```

**Example:**
```python
# API endpoint
@app.post("/upload")
def upload_video(file):
    video_id = save_to_storage(file)

    # Send to queue instead of processing now
    queue.send("video_processing", {"video_id": video_id})

    return {"status": "processing", "video_id": video_id}

# Background worker
def process_video_worker():
    while True:
        message = queue.receive("video_processing")
        transcode_video(message["video_id"])
        queue.acknowledge(message.id)
```

**Used in:** [Video Streaming Platform](../video-streaming-platform.md)

---

### 2. Load Leveling

**Problem:** Traffic spikes overwhelm servers.

**Solution:** Queue absorbs spikes, workers process at steady rate.

```
1000 req/s → Queue → Workers process at 100 req/s
Queue grows during spikes, shrinks during normal load
```

---

### 3. Event-Driven Architecture

**Example: User Signup**

```
User signs up
    ↓
Queue: user_created event
    ↓
Multiple consumers:
    ├── Send welcome email
    ├── Create analytics record
    ├── Add to CRM
    └── Start onboarding flow
```

Each service is independent and can fail/retry without affecting others.

---

### 4. Retry & Dead Letter Queues

**Problem:** Message processing fails (e.g., external API down).

**Solution:**
1. Retry failed messages automatically
2. After max retries, move to **Dead Letter Queue (DLQ)**
3. Alert engineers to investigate DLQ

```python
def process_message(msg):
    try:
        send_to_external_api(msg.data)
        queue.acknowledge(msg)
    except Exception as e:
        if msg.retry_count < 3:
            queue.retry(msg)  # Try again
        else:
            dead_letter_queue.send(msg)  # Give up
            alert_engineers(msg, e)
```

---

## Choosing a Message Queue

| Feature | **RabbitMQ** | **Kafka** | **SQS** | **Redis** |
|---------|-------------|-----------|---------|----------|
| **Throughput** | Medium | Very High | Medium | High |
| **Persistence** | Yes | Yes | Yes | Optional |
| **Ordering** | Per queue | Per partition | FIFO only | No |
| **Delivery** | At-most/At-least | At-least/Exactly | At-least/Exactly | At-most |
| **Complexity** | Medium | High | Low | Low |
| **Use Case** | Task queues | Event streaming | AWS apps | Real-time, cache |

---

## Message Queue Anti-Patterns

### ❌ Using Queue as Database

Don't store data in queues long-term - use a database.

### ❌ Ignoring Message Ordering

If order matters, use single partition or FIFO queue.

### ❌ Not Handling Duplicates

Always make consumers **idempotent** (safe to process same message twice).

**Example:**
```python
def process_order(order_id):
    # Check if already processed
    if db.order_exists(order_id):
        return  # Already done, skip

    # Process order
    create_order(order_id)
```

### ❌ Blocking in Consumers

Don't make consumers wait - process asynchronously.

---

## Monitoring Message Queues

**Key Metrics:**
- **Queue depth** - Number of messages waiting
- **Processing rate** - Messages/sec consumed
- **Error rate** - Failed messages
- **Latency** - Time from send to process
- **Dead letter queue size** - Failed messages

**Alerts:**
- Queue depth > threshold (consumers too slow)
- DLQ size increasing (processing errors)
- High latency (performance issue)

---

## Real-World Examples

### Video Streaming Platform

**Use Case:** Video transcoding

```
User uploads video
    ↓
Queue: transcode_job {video_id, resolutions: [1080p, 720p, 480p]}
    ↓
Workers: Transcode in parallel
    ↓
Queue: transcode_complete {video_id}
    ↓
Update database: status = "ready"
```

See: [Video Streaming - Transcoding Pipeline](../video-streaming-platform/video-transcoding-pipeline.md)

---

### E-commerce Order Processing

```
Order placed
    ↓
Queue: order_created
    ↓
Consumers:
    ├── Payment processing
    ├── Inventory update
    ├── Shipping label generation
    └── Email confirmation
```

---

## Summary

✅ **Use queues for async processing** - Don't make users wait
✅ **RabbitMQ** - General-purpose task queues
✅ **Kafka** - High-throughput event streaming
✅ **SQS** - Managed queues for AWS
✅ **Redis** - Fast, ephemeral pub/sub
✅ **Make consumers idempotent** - Handle duplicates gracefully
✅ **Monitor queue depth** - Detect processing bottlenecks

**Golden Rule:** If you can do it later, use a queue!
