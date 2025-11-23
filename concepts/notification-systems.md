# Notification Systems

## Overview

A **notification system** is responsible for delivering messages to users across multiple channels (push notifications, email, SMS, in-app) at scale. Building a reliable, scalable notification system is critical for user engagement in modern applications.

## Why Do We Need Notification Systems?

### Problems Without Proper Notification Architecture

❌ **Message loss** - Notifications fail silently without delivery guarantees
❌ **Poor scalability** - System crashes under high notification volume
❌ **No prioritization** - Critical alerts get delayed behind promotional messages
❌ **Duplicate notifications** - Users receive the same message multiple times
❌ **Lack of user control** - No way to manage notification preferences

### Goals

✅ **Reliable delivery** - Ensure messages reach users (at-least-once delivery)
✅ **Low latency** - Critical notifications delivered in seconds
✅ **Scalability** - Handle millions of notifications per minute
✅ **Multi-channel** - Support push, email, SMS, in-app notifications
✅ **User preferences** - Respect opt-outs and delivery preferences
✅ **Analytics** - Track delivery, open rates, and engagement

---

## Notification Channels

### 1. Push Notifications

**Platform-specific services:**
- **iOS** - Apple Push Notification Service (APNs)
- **Android** - Firebase Cloud Messaging (FCM)
- **Web** - Web Push API (service workers)

**Characteristics:**
- Real-time delivery
- Requires user permission
- Device-specific (need device token)
- High engagement rates

**Example Flow:**
```
App → Notification Service → FCM/APNs → User's Device
```

---

### 2. Email Notifications

**Email service providers:**
- SendGrid
- Amazon SES
- Mailgun
- Postmark

**Characteristics:**
- Asynchronous (not real-time)
- No permission required (if user signed up)
- Rich content (HTML, images, links)
- Spam filters can block

**Example Flow:**
```
App → Email Queue → Email Service (SendGrid) → User's Inbox
```

---

### 3. SMS Notifications

**SMS gateway providers:**
- Twilio
- Amazon SNS
- Nexmo/Vonage
- Plivo

**Characteristics:**
- Near real-time delivery
- Expensive (per-message cost)
- Character limits (160 chars)
- High open rates (98%+)

**Example Flow:**
```
App → SMS Queue → Twilio → Carrier → User's Phone
```

---

### 4. In-App Notifications

**Real-time communication:**
- WebSockets
- Server-Sent Events (SSE)
- Long polling

**Characteristics:**
- Only delivered when user is active
- Instant delivery
- No external dependencies
- Can store for later viewing

**Example Flow:**
```
App → WebSocket Server → Active User Session
```

---

## System Architecture

### High-Level Design

```
┌─────────────────┐
│  Application    │
│  Services       │
│  (Order, Chat)  │
└────────┬────────┘
         │
         ↓
┌─────────────────────────────────────────────┐
│       Notification Service API              │
│  - Validate requests                        │
│  - Check user preferences                   │
│  - Enqueue notifications                    │
└────────┬────────────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────────────┐
│         Message Queue (Kafka/RabbitMQ)      │
│  Topics: push_queue, email_queue, sms_queue │
└────────┬────────────────────────────────────┘
         │
         ↓
┌────────┴────────┬────────────┬──────────────┐
│                 │            │              │
↓                 ↓            ↓              ↓
┌─────────┐  ┌────────┐  ┌────────┐  ┌──────────┐
│  Push   │  │ Email  │  │  SMS   │  │  In-App  │
│ Worker  │  │ Worker │  │ Worker │  │  Worker  │
└────┬────┘  └───┬────┘  └───┬────┘  └────┬─────┘
     │           │           │            │
     ↓           ↓           ↓            ↓
┌────────┐  ┌─────────┐ ┌────────┐  ┌──────────┐
│ FCM/   │  │SendGrid │ │ Twilio │  │WebSocket │
│ APNs   │  │   SES   │ │  SNS   │  │  Server  │
└────────┘  └─────────┘ └────────┘  └──────────┘
```

---

## Database Schema

### Notifications Table

```sql
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    type VARCHAR(50) NOT NULL,  -- 'order', 'chat', 'promo'
    priority VARCHAR(20) NOT NULL,  -- 'critical', 'high', 'normal', 'low'
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    data JSONB,  -- Additional payload
    channels TEXT[],  -- ['push', 'email', 'sms']
    status VARCHAR(20) DEFAULT 'pending',  -- 'pending', 'sent', 'failed'
    scheduled_at TIMESTAMP,
    sent_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_status (user_id, status),
    INDEX idx_scheduled (scheduled_at) WHERE status = 'pending'
);
```

### User Preferences Table

```sql
CREATE TABLE notification_preferences (
    user_id BIGINT PRIMARY KEY,
    push_enabled BOOLEAN DEFAULT true,
    email_enabled BOOLEAN DEFAULT true,
    sms_enabled BOOLEAN DEFAULT false,

    -- Per-category preferences
    preferences JSONB DEFAULT '{
        "order_updates": {"push": true, "email": true, "sms": false},
        "chat_messages": {"push": true, "email": false, "sms": false},
        "promotions": {"push": false, "email": true, "sms": false}
    }'::jsonb,

    quiet_hours JSONB,  -- {"start": "22:00", "end": "08:00", "timezone": "America/New_York"}
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### Device Tokens Table

```sql
CREATE TABLE device_tokens (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    platform VARCHAR(20) NOT NULL,  -- 'ios', 'android', 'web'
    token TEXT NOT NULL UNIQUE,
    active BOOLEAN DEFAULT true,
    last_used_at TIMESTAMP DEFAULT NOW(),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_platform (user_id, platform, active)
);
```

---

## Fan-Out Patterns

### Fan-Out on Write

**Send notifications immediately when event occurs.**

**Example: Social Media Post**

When a user posts, immediately send notifications to all followers.

```python
def on_new_post(user_id, post_id):
    # Get all followers (could be millions)
    followers = db.query("SELECT user_id FROM followers WHERE following_id = ?", [user_id])

    # Fan-out: Send notification to each follower
    for follower in followers:
        notification_service.send({
            "user_id": follower.user_id,
            "type": "new_post",
            "title": f"{user.name} posted",
            "channels": ["push", "in-app"]
        })
```

**Pros:**
- ✅ Immediate delivery
- ✅ Simple to implement

**Cons:**
- ❌ Slow for users with many followers (celebrity problem)
- ❌ Wastes resources if users don't check notifications

**Best for:** Small fan-out (< 1000 recipients)

---

### Fan-Out on Read

**Generate notifications when user checks their feed.**

**Example: Social Media Feed**

When user opens app, fetch recent posts from people they follow.

```python
def get_user_feed(user_id):
    # Get users this person follows
    following = db.query("SELECT following_id FROM followers WHERE user_id = ?", [user_id])

    # Fetch recent posts (fan-out on read)
    posts = db.query("""
        SELECT * FROM posts
        WHERE user_id IN (?)
        ORDER BY created_at DESC
        LIMIT 50
    """, [following])

    return posts
```

**Pros:**
- ✅ Efficient for users with many followers
- ✅ No wasted work

**Cons:**
- ❌ Slow read queries (must aggregate from many users)
- ❌ Delayed notifications

**Best for:** Large fan-out (millions of recipients)

---

### Hybrid Approach

**Combine both strategies based on user type.**

```python
def on_new_post(user_id, post_id):
    follower_count = db.count("SELECT COUNT(*) FROM followers WHERE following_id = ?", [user_id])

    if follower_count < 1000:
        # Fan-out on write for regular users
        followers = db.query("SELECT user_id FROM followers WHERE following_id = ?", [user_id])
        for follower in followers:
            notification_service.send({
                "user_id": follower.user_id,
                "type": "new_post",
                "channels": ["push"]
            })
    else:
        # Fan-out on read for celebrities
        # Just store the post, users will fetch when they open app
        cache.sadd(f"new_posts:{user_id}", post_id)
```

**Best for:** Mixed workload (celebrities + regular users)

---

## Implementation Examples

### 1. Notification Service API

**Send notification:**

```python
from flask import Flask, request, jsonify
import redis
import json

app = Flask(__name__)
redis_client = redis.Redis(host='localhost', port=6379)

@app.route('/notifications/send', methods=['POST'])
def send_notification():
    data = request.json

    # Validate request
    if not data.get('user_id') or not data.get('title'):
        return jsonify({"error": "Missing required fields"}), 400

    # Check user preferences
    preferences = get_user_preferences(data['user_id'])

    # Filter channels based on preferences
    allowed_channels = []
    notification_type = data.get('type', 'default')

    if preferences.get('push_enabled') and should_send_push(preferences, notification_type):
        allowed_channels.append('push')
    if preferences.get('email_enabled') and should_send_email(preferences, notification_type):
        allowed_channels.append('email')
    if preferences.get('sms_enabled') and should_send_sms(preferences, notification_type):
        allowed_channels.append('sms')

    if not allowed_channels:
        return jsonify({"message": "User opted out of all channels"}), 200

    # Create notification record
    notification = {
        "id": generate_id(),
        "user_id": data['user_id'],
        "type": notification_type,
        "priority": data.get('priority', 'normal'),
        "title": data['title'],
        "body": data['body'],
        "channels": allowed_channels,
        "created_at": now()
    }

    # Save to database
    db.insert_notification(notification)

    # Enqueue to message queue
    for channel in allowed_channels:
        redis_client.lpush(f"{channel}_queue", json.dumps(notification))

    return jsonify({"id": notification['id'], "status": "queued"}), 200

def should_send_push(preferences, notification_type):
    # Check quiet hours
    if is_quiet_hours(preferences.get('quiet_hours')):
        return False

    # Check per-category preferences
    category_prefs = preferences.get('preferences', {}).get(notification_type, {})
    return category_prefs.get('push', True)
```

---

### 2. Push Notification Worker

**Process push notifications:**

```python
import redis
import json
import requests
from firebase_admin import messaging

redis_client = redis.Redis(host='localhost', port=6379)

def push_worker():
    while True:
        # Block until message available
        _, message = redis_client.brpop('push_queue', timeout=0)
        notification = json.loads(message)

        try:
            send_push_notification(notification)
        except Exception as e:
            # Retry logic
            handle_failure(notification, e)

def send_push_notification(notification):
    # Get user's device tokens
    tokens = db.query("""
        SELECT token, platform FROM device_tokens
        WHERE user_id = ? AND active = true
    """, [notification['user_id']])

    if not tokens:
        print(f"No device tokens for user {notification['user_id']}")
        return

    # Send to FCM/APNs
    for token_info in tokens:
        if token_info['platform'] in ['android', 'web']:
            send_fcm(token_info['token'], notification)
        elif token_info['platform'] == 'ios':
            send_apns(token_info['token'], notification)

    # Update notification status
    db.execute("""
        UPDATE notifications SET status = 'sent', sent_at = NOW()
        WHERE id = ?
    """, [notification['id']])

def send_fcm(token, notification):
    message = messaging.Message(
        notification=messaging.Notification(
            title=notification['title'],
            body=notification['body']
        ),
        data=notification.get('data', {}),
        token=token
    )

    try:
        response = messaging.send(message)
        print(f"Successfully sent: {response}")
    except messaging.UnregisteredError:
        # Token is invalid, deactivate it
        db.execute("UPDATE device_tokens SET active = false WHERE token = ?", [token])
```

---

### 3. Email Worker

**Process email notifications:**

```python
import redis
import json
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import Mail

redis_client = redis.Redis(host='localhost', port=6379)
sendgrid_client = SendGridAPIClient(api_key='YOUR_API_KEY')

def email_worker():
    while True:
        _, message = redis_client.brpop('email_queue', timeout=0)
        notification = json.loads(message)

        try:
            send_email(notification)
        except Exception as e:
            handle_failure(notification, e)

def send_email(notification):
    # Get user's email
    user = db.query("SELECT email, name FROM users WHERE id = ?", [notification['user_id']])

    if not user.email:
        print(f"No email for user {notification['user_id']}")
        return

    # Build email
    message = Mail(
        from_email='notifications@example.com',
        to_emails=user.email,
        subject=notification['title'],
        html_content=render_email_template(notification)
    )

    # Send via SendGrid
    response = sendgrid_client.send(message)

    # Update notification status
    db.execute("""
        UPDATE notifications SET status = 'sent', sent_at = NOW()
        WHERE id = ?
    """, [notification['id']])

    print(f"Email sent to {user.email}: {response.status_code}")

def render_email_template(notification):
    return f"""
    <html>
        <body>
            <h2>{notification['title']}</h2>
            <p>{notification['body']}</p>
        </body>
    </html>
    """
```

---

### 4. Rate Limiting per User

**Prevent notification spam:**

```python
import redis

redis_client = redis.Redis(host='localhost', port=6379)

def check_rate_limit(user_id, channel):
    """
    Rate limit: Max 10 notifications per hour per channel
    """
    key = f"rate_limit:{channel}:{user_id}"

    # Get current count
    count = redis_client.get(key)

    if count and int(count) >= 10:
        return False  # Rate limit exceeded

    # Increment counter
    pipe = redis_client.pipeline()
    pipe.incr(key)
    pipe.expire(key, 3600)  # 1 hour TTL
    pipe.execute()

    return True

# Usage in notification service
@app.route('/notifications/send', methods=['POST'])
def send_notification():
    data = request.json

    for channel in data['channels']:
        if not check_rate_limit(data['user_id'], channel):
            print(f"Rate limit exceeded for user {data['user_id']} on {channel}")
            continue

        # Enqueue notification
        redis_client.lpush(f"{channel}_queue", json.dumps(data))
```

---

## Priority Queues

**Ensure critical notifications are delivered first.**

```python
import heapq
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PriorityNotification:
    priority: int
    notification: Any = field(compare=False)

class PriorityQueue:
    def __init__(self):
        self.queue = []
        self.priority_map = {
            'critical': 0,
            'high': 1,
            'normal': 2,
            'low': 3
        }

    def push(self, notification):
        priority = self.priority_map.get(notification.get('priority', 'normal'), 2)
        heapq.heappush(self.queue, PriorityNotification(priority, notification))

    def pop(self):
        if self.queue:
            return heapq.heappop(self.queue).notification
        return None

# Worker with priority queue
priority_queue = PriorityQueue()

def priority_worker():
    while True:
        notification = priority_queue.pop()
        if notification:
            send_notification(notification)
        else:
            time.sleep(0.1)
```

---

## Delivery Guarantees

### At-Least-Once Delivery

**Ensure messages are delivered, even if duplicated.**

```python
def process_notification(notification):
    # Process notification
    send_push_notification(notification)

    # Acknowledge only after successful delivery
    redis_client.lrem('push_queue', 1, json.dumps(notification))

# If worker crashes before acknowledgment, message stays in queue
```

---

### Idempotency

**Prevent duplicate notifications to users.**

```python
def send_notification(notification):
    # Generate idempotency key
    idempotency_key = f"{notification['id']}:{notification['user_id']}"

    # Check if already processed
    if redis_client.exists(f"sent:{idempotency_key}"):
        print("Notification already sent, skipping")
        return

    # Send notification
    send_push_notification(notification)

    # Mark as sent (with 24-hour TTL)
    redis_client.setex(f"sent:{idempotency_key}", 86400, "1")
```

---

## Real-World Examples

### 1. E-commerce Order Updates

**Multi-channel notification for order status:**

```python
def on_order_shipped(order_id):
    order = db.get_order(order_id)

    notification_service.send({
        "user_id": order.user_id,
        "type": "order_updates",
        "priority": "high",
        "title": "Your order has shipped!",
        "body": f"Order #{order_id} is on its way",
        "channels": ["push", "email", "sms"],
        "data": {
            "order_id": order_id,
            "tracking_number": order.tracking_number
        }
    })
```

---

### 2. Ride-Sharing Driver Match

**Critical real-time notification:**

```python
def on_driver_matched(ride_id, driver_id, rider_id):
    # Send to rider
    notification_service.send({
        "user_id": rider_id,
        "type": "ride_updates",
        "priority": "critical",  # High priority
        "title": "Driver found!",
        "body": "Your driver will arrive in 3 minutes",
        "channels": ["push", "sms"],  # SMS as backup
        "data": {
            "ride_id": ride_id,
            "driver_name": driver.name,
            "driver_photo": driver.photo_url,
            "eta": 3
        }
    })

    # Send to driver
    notification_service.send({
        "user_id": driver_id,
        "type": "ride_updates",
        "priority": "critical",
        "title": "New ride request",
        "body": f"Pickup: {ride.pickup_address}",
        "channels": ["push"],
        "data": {
            "ride_id": ride_id,
            "pickup_location": ride.pickup_location
        }
    })
```

---

### 3. Chat Application

**Real-time message notifications:**

```python
def on_new_message(chat_id, sender_id, message):
    # Get all chat participants except sender
    participants = db.query("""
        SELECT user_id FROM chat_participants
        WHERE chat_id = ? AND user_id != ?
    """, [chat_id, sender_id])

    sender = db.get_user(sender_id)

    for participant in participants:
        # Check if user is online
        if is_user_online(participant.user_id):
            # Send via WebSocket (in-app)
            websocket_service.send(participant.user_id, {
                "type": "new_message",
                "chat_id": chat_id,
                "message": message
            })
        else:
            # Send push notification
            notification_service.send({
                "user_id": participant.user_id,
                "type": "chat_messages",
                "priority": "normal",
                "title": f"New message from {sender.name}",
                "body": message.text[:100],  # Preview
                "channels": ["push"],
                "data": {
                    "chat_id": chat_id,
                    "message_id": message.id
                }
            })
```

---

## Scaling Strategies

### 1. Horizontal Scaling

**Run multiple workers per channel:**

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Push    │  │  Push    │  │  Push    │
│ Worker 1 │  │ Worker 2 │  │ Worker 3 │
└──────────┘  └──────────┘  └──────────┘
      ↑             ↑             ↑
      └─────────────┴─────────────┘
              Push Queue
```

**Each worker pulls from same queue (competing consumers).**

---

### 2. Partitioning by User

**Shard users across queues:**

```python
def get_queue_name(user_id, channel):
    partition = hash(user_id) % NUM_PARTITIONS
    return f"{channel}_queue_{partition}"

# Send to specific partition
queue_name = get_queue_name(notification['user_id'], 'push')
redis_client.lpush(queue_name, json.dumps(notification))
```

---

### 3. Batching

**Send multiple notifications in one API call:**

```python
def email_worker_with_batching():
    batch = []
    batch_size = 100
    batch_timeout = 5  # seconds

    while True:
        try:
            _, message = redis_client.brpop('email_queue', timeout=batch_timeout)
            notification = json.loads(message)
            batch.append(notification)

            if len(batch) >= batch_size:
                send_batch_emails(batch)
                batch = []
        except:
            # Timeout - send partial batch
            if batch:
                send_batch_emails(batch)
                batch = []

def send_batch_emails(notifications):
    # SendGrid supports batch sending
    messages = [build_message(n) for n in notifications]
    sendgrid_client.send_multiple(messages)
```

---

## Monitoring & Analytics

### Track Delivery Metrics

```sql
CREATE TABLE notification_analytics (
    id BIGSERIAL PRIMARY KEY,
    notification_id BIGINT NOT NULL,
    event_type VARCHAR(50) NOT NULL,  -- 'sent', 'delivered', 'opened', 'clicked'
    channel VARCHAR(20) NOT NULL,
    timestamp TIMESTAMP DEFAULT NOW(),
    INDEX idx_notification (notification_id),
    INDEX idx_timestamp (timestamp)
);
```

**Track events:**

```python
def track_event(notification_id, event_type, channel):
    db.execute("""
        INSERT INTO notification_analytics (notification_id, event_type, channel)
        VALUES (?, ?, ?)
    """, [notification_id, event_type, channel])

# Usage
track_event(notification_id, 'sent', 'push')
track_event(notification_id, 'delivered', 'push')
track_event(notification_id, 'opened', 'push')
```

---

## Comparison Table

| Channel | **Latency** | **Cost** | **Reliability** | **Rich Content** | **Permission Required** |
|---------|------------|---------|----------------|-----------------|------------------------|
| **Push** | < 1s | Free | High | Medium | Yes |
| **Email** | 1-10s | Low | Medium | High | No |
| **SMS** | 1-5s | High | Very High | Low | No |
| **In-App** | < 100ms | Free | High (if online) | High | No |

---

## Summary

✅ **Multi-channel delivery** - Push, email, SMS, in-app
✅ **Fan-out strategies** - Fan-out on write vs read vs hybrid
✅ **Message queues** - Decouple notification sending from application logic
✅ **Priority queues** - Ensure critical notifications delivered first
✅ **User preferences** - Respect opt-outs and quiet hours
✅ **Rate limiting** - Prevent notification spam
✅ **Idempotency** - Prevent duplicate notifications
✅ **Delivery guarantees** - At-least-once delivery with retries
✅ **Scaling** - Horizontal scaling, partitioning, batching
✅ **Analytics** - Track sent, delivered, opened, clicked events

**Best Practices:**
- Use message queues for reliability and scalability
- Implement idempotency to prevent duplicates
- Respect user preferences and quiet hours
- Prioritize critical notifications (ride matches, security alerts)
- Batch requests to external services (SendGrid, Twilio)
- Monitor delivery metrics and failure rates
- Implement retry logic with exponential backoff
