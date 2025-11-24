# Real-Time Chat Application (WhatsApp/Slack)

Design a scalable real-time messaging platform that supports one-on-one chat, group messaging, message delivery guarantees, online presence, and media sharing.

---

## 1. Requirements Analysis

### Functional Requirements

✅ **One-on-one messaging** - Send and receive text messages in real-time

✅ **Group chat** - Support group conversations with multiple participants

✅ **Message delivery status** - Sent, delivered, and read receipts

✅ **Online presence** - Show when users are online/offline/typing

✅ **Message history** - Store and retrieve past conversations

✅ **Media sharing** - Send images, videos, documents

✅ **Push notifications** - Notify users of new messages when offline

✅ **Search** - Search messages and conversations

✅ **User profiles** - Profile photos, status, last seen

### Non-Functional Requirements

✅ **Low latency** - Messages delivered in < 1 second

✅ **High availability** - 99.99% uptime

✅ **Scalability** - Handle millions of concurrent users

✅ **Reliability** - Guarantee message delivery (at-least-once)

✅ **Real-time** - Bidirectional communication with WebSockets

✅ **Security** - End-to-end encryption (optional)

✅ **Consistency** - Messages appear in same order for all users

---

## 2. High-Level Architecture

### Key Components

1. **Client Apps** (Mobile/Web) - iOS, Android, Web interfaces
2. **API Gateway** - Routes HTTP requests, handles authentication
3. **WebSocket Server** - Maintains persistent connections for real-time messaging
4. **Chat Service** - Business logic for sending/receiving messages
5. **Presence Service** - Tracks online/offline/typing status
6. **Message Store** - PostgreSQL for recent messages, Cassandra for history
7. **Media Service** - Handles file uploads/downloads
8. **Push Notification Service** - Sends notifications to offline users
9. **Redis Cache** - Caches user sessions, recent messages, presence
10. **Message Queue** (Kafka) - Async processing, message fan-out
11. **Search Service** (Elasticsearch) - Full-text search on messages

### Architecture Diagram

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Mobile App  │     │   Web App    │     │  Mobile App  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼────────┐
                     │  API Gateway  │
                     │ (Auth, HTTP)  │
                     └──────┬────────┘
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
┌──────▼──────┐      ┌──────▼──────┐     ┌──────▼──────┐
│  WebSocket  │      │    Chat     │     │  Presence   │
│   Server    │      │   Service   │     │   Service   │
└──────┬──────┘      └──────┬──────┘     └──────┬──────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
┌──────▼──────┐      ┌──────▼──────┐     ┌──────▼──────┐
│    Redis    │      │ PostgreSQL  │     │   Kafka     │
│   (Cache)   │      │ (Messages)  │     │   (Queue)   │
└─────────────┘      └─────────────┘     └─────────────┘
```

---

## 3. Database Schema

### PostgreSQL (Recent Messages & Metadata)

#### Users Table

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    phone VARCHAR(20) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE,
    display_name VARCHAR(100),
    profile_photo_url TEXT,
    status_message VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    last_seen TIMESTAMP,
    is_online BOOLEAN DEFAULT false,
    INDEX idx_phone (phone),
    INDEX idx_username (username)
);
```

#### Conversations Table

```sql
CREATE TABLE conversations (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(20) NOT NULL, -- 'direct' or 'group'
    name VARCHAR(100), -- Group name (NULL for direct)
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_type (type)
);
```

#### Conversation Members Table

```sql
CREATE TABLE conversation_members (
    id BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    joined_at TIMESTAMP DEFAULT NOW(),
    last_read_message_id BIGINT, -- For unread count
    is_admin BOOLEAN DEFAULT false,
    notifications_enabled BOOLEAN DEFAULT true,
    UNIQUE(conversation_id, user_id),
    INDEX idx_user_conversations (user_id, conversation_id),
    INDEX idx_conversation_members (conversation_id)
);
```

#### Messages Table (Recent - Last 30 Days)

```sql
CREATE TABLE messages (
    id BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT REFERENCES conversations(id) ON DELETE CASCADE,
    sender_id BIGINT REFERENCES users(id),
    message_type VARCHAR(20) NOT NULL, -- 'text', 'image', 'video', 'file'
    content TEXT, -- Message text
    media_url TEXT, -- URL for images/videos/files
    created_at TIMESTAMP DEFAULT NOW(),
    is_deleted BOOLEAN DEFAULT false,
    INDEX idx_conversation_time (conversation_id, created_at DESC),
    INDEX idx_sender (sender_id)
);
```

#### Message Status Table

```sql
CREATE TABLE message_status (
    id BIGSERIAL PRIMARY KEY,
    message_id BIGINT REFERENCES messages(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES users(id),
    status VARCHAR(20) NOT NULL, -- 'sent', 'delivered', 'read'
    timestamp TIMESTAMP DEFAULT NOW(),
    UNIQUE(message_id, user_id),
    INDEX idx_message_status (message_id, user_id)
);
```

---

### Cassandra (Message History - Long-term Storage)

```sql
CREATE TABLE message_history (
    conversation_id BIGINT,
    created_at TIMESTAMP,
    message_id BIGINT,
    sender_id BIGINT,
    message_type TEXT,
    content TEXT,
    media_url TEXT,
    PRIMARY KEY ((conversation_id), created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at DESC, message_id DESC);

-- Query: Get last 50 messages from conversation
SELECT * FROM message_history
WHERE conversation_id = 123
ORDER BY created_at DESC
LIMIT 50;
```

---

### Redis (Real-Time Data)

#### User Sessions (WebSocket Connections)

```redis
# Map user to WebSocket server
HSET user:123:session server "ws-server-5"
HSET user:123:session connection_id "conn-abc-123"
HSET user:123:session connected_at "1699920000"

# Set expiry (if no heartbeat, user is offline)
EXPIRE user:123:session 300  # 5 minutes
```

#### Online Presence

```redis
# Set user online
SET presence:user:123 "online" EX 60

# Typing indicator (expires after 5 seconds)
SET typing:conversation:456:user:123 "1" EX 5

# Get all typing users in conversation
KEYS typing:conversation:456:*
```

#### Unread Message Count

```redis
# Increment unread count
HINCRBY unread:user:123 conversation:456 1

# Get unread counts for all conversations
HGETALL unread:user:123

# Clear unread count when user reads messages
HDEL unread:user:123 conversation:456
```

#### Recent Messages Cache

```redis
# Cache last 100 messages per conversation
ZADD messages:conversation:456 <timestamp> <message_json>

# Get recent messages
ZREVRANGE messages:conversation:456 0 99 WITHSCORES
```

---

## 4. Core Workflows

### 4.1 Sending a Message (1-on-1 Chat)

**Flow:**

```
User A → WebSocket Server → Chat Service → Message Queue → Database
                                              ↓
                                      User B (if online)
```

**Implementation:**

```python
from flask_socketio import SocketIO, emit
import redis
import json
from datetime import datetime

socketio = SocketIO()
redis_client = redis.Redis()

@socketio.on('send_message')
def handle_send_message(data):
    """
    User sends a message
    data = {
        'conversation_id': 456,
        'sender_id': 123,
        'content': 'Hello!',
        'message_type': 'text'
    }
    """
    sender_id = data['sender_id']
    conversation_id = data['conversation_id']
    content = data['content']
    message_type = data.get('message_type', 'text')

    # 1. Save message to database
    message = db.execute("""
        INSERT INTO messages (conversation_id, sender_id, message_type, content, created_at)
        VALUES (?, ?, ?, ?, NOW())
        RETURNING id, created_at
    """, [conversation_id, sender_id, message_type, content])

    message_id = message['id']
    created_at = message['created_at']

    # 2. Cache message in Redis
    message_data = {
        'id': message_id,
        'conversation_id': conversation_id,
        'sender_id': sender_id,
        'content': content,
        'message_type': message_type,
        'created_at': created_at.isoformat()
    }

    redis_client.zadd(
        f'messages:conversation:{conversation_id}',
        {json.dumps(message_data): created_at.timestamp()}
    )

    # 3. Get conversation members (recipients)
    members = db.query("""
        SELECT user_id FROM conversation_members
        WHERE conversation_id = ? AND user_id != ?
    """, [conversation_id, sender_id])

    # 4. Send to online recipients via WebSocket
    for member in members:
        recipient_id = member['user_id']

        # Get recipient's WebSocket connection
        session = redis_client.hgetall(f'user:{recipient_id}:session')

        if session:
            # User is online, send via WebSocket
            socketio.emit('new_message', message_data, room=f'user_{recipient_id}')

            # Mark as delivered
            db.execute("""
                INSERT INTO message_status (message_id, user_id, status)
                VALUES (?, ?, 'delivered')
            """, [message_id, recipient_id])
        else:
            # User is offline, send push notification
            send_push_notification(recipient_id, {
                'title': f'New message from {get_user_name(sender_id)}',
                'body': content[:100],
                'conversation_id': conversation_id
            })

            # Mark as sent (not delivered yet)
            db.execute("""
                INSERT INTO message_status (message_id, user_id, status)
                VALUES (?, ?, 'sent')
            """, [message_id, recipient_id])

        # Increment unread count
        redis_client.hincrby(f'unread:user:{recipient_id}', f'conversation:{conversation_id}', 1)

    # 5. Acknowledge to sender
    emit('message_sent', {
        'message_id': message_id,
        'status': 'sent',
        'timestamp': created_at.isoformat()
    })

    # 6. Publish to Kafka for async processing (archival, search indexing)
    kafka_producer.send('chat_messages', {
        'message_id': message_id,
        'conversation_id': conversation_id,
        'sender_id': sender_id,
        'content': content,
        'created_at': created_at.isoformat()
    })
```

---

### 4.2 Message Delivery Guarantees

**Three-tier status system:**

1. **Sent** (✓) - Message stored in database
2. **Delivered** (✓✓) - Message delivered to recipient's device
3. **Read** (✓✓ blue) - Recipient opened and read the message

**Implementation:**

```python
# When recipient comes online and receives message
@socketio.on('message_delivered')
def handle_message_delivered(data):
    message_id = data['message_id']
    user_id = data['user_id']

    # Update status to delivered
    db.execute("""
        UPDATE message_status
        SET status = 'delivered', timestamp = NOW()
        WHERE message_id = ? AND user_id = ?
    """, [message_id, user_id])

    # Notify sender
    message = db.query("SELECT sender_id FROM messages WHERE id = ?", [message_id])
    socketio.emit('message_status_update', {
        'message_id': message_id,
        'status': 'delivered',
        'user_id': user_id
    }, room=f"user_{message['sender_id']}")


# When recipient reads message
@socketio.on('message_read')
def handle_message_read(data):
    message_id = data['message_id']
    user_id = data['user_id']

    # Update status to read
    db.execute("""
        UPDATE message_status
        SET status = 'read', timestamp = NOW()
        WHERE message_id = ? AND user_id = ?
    """, [message_id, user_id])

    # Update last_read_message_id for unread count
    conversation_id = data['conversation_id']
    db.execute("""
        UPDATE conversation_members
        SET last_read_message_id = ?
        WHERE conversation_id = ? AND user_id = ?
    """, [message_id, conversation_id, user_id])

    # Clear unread count in Redis
    redis_client.hdel(f'unread:user:{user_id}', f'conversation:{conversation_id}')

    # Notify sender
    message = db.query("SELECT sender_id FROM messages WHERE id = ?", [message_id])
    socketio.emit('message_status_update', {
        'message_id': message_id,
        'status': 'read',
        'user_id': user_id
    }, room=f"user_{message['sender_id']}")
```

---

### 4.3 Group Chat Message Fan-Out

**Challenge:** Send message to 1000 group members efficiently

**Approach:** Fan-out on read (lazy loading)

```python
@socketio.on('send_group_message')
def handle_group_message(data):
    conversation_id = data['conversation_id']
    sender_id = data['sender_id']
    content = data['content']

    # 1. Save message once
    message = db.execute("""
        INSERT INTO messages (conversation_id, sender_id, content, created_at)
        VALUES (?, ?, ?, NOW())
        RETURNING id, created_at
    """, [conversation_id, sender_id, content])

    message_id = message['id']

    # 2. Get online members only
    online_members = get_online_members(conversation_id, exclude=sender_id)

    # 3. Send to online members (max 100 at a time)
    for member_id in online_members[:100]:  # Limit initial fan-out
        socketio.emit('new_message', {
            'message_id': message_id,
            'conversation_id': conversation_id,
            'sender_id': sender_id,
            'content': content,
            'created_at': message['created_at'].isoformat()
        }, room=f'user_{member_id}')

    # 4. For offline members, send push notification summary (batched)
    offline_members = get_offline_members(conversation_id, exclude=sender_id)
    if offline_members:
        # Send one notification per user (not per message)
        send_batch_push_notifications(offline_members, {
            'title': f'{get_group_name(conversation_id)}',
            'body': f'{get_user_name(sender_id)}: {content[:50]}...',
            'conversation_id': conversation_id
        })

    # 5. Acknowledge to sender
    emit('message_sent', {'message_id': message_id, 'status': 'sent'})


def get_online_members(conversation_id, exclude=None):
    """Get members who are currently online"""
    members = db.query("""
        SELECT user_id FROM conversation_members
        WHERE conversation_id = ? AND user_id != ?
    """, [conversation_id, exclude])

    online = []
    for member in members:
        if redis_client.exists(f'user:{member["user_id"]}:session'):
            online.append(member['user_id'])

    return online
```

---

### 4.4 Online Presence & Typing Indicators

**Online/Offline Detection:**

```python
# User connects
@socketio.on('connect')
def handle_connect():
    user_id = get_authenticated_user_id()

    # Store session in Redis
    redis_client.hset(f'user:{user_id}:session', 'server', CURRENT_SERVER_ID)
    redis_client.hset(f'user:{user_id}:session', 'connected_at', int(time.time()))
    redis_client.expire(f'user:{user_id}:session', 300)  # 5 min expiry

    # Set online status
    redis_client.set(f'presence:user:{user_id}', 'online', ex=60)

    # Update database
    db.execute("UPDATE users SET is_online = true WHERE id = ?", [user_id])

    # Notify contacts
    notify_contacts_presence(user_id, 'online')


# Heartbeat to keep connection alive
@socketio.on('heartbeat')
def handle_heartbeat():
    user_id = get_authenticated_user_id()

    # Refresh expiry
    redis_client.expire(f'user:{user_id}:session', 300)
    redis_client.set(f'presence:user:{user_id}', 'online', ex=60)


# User disconnects
@socketio.on('disconnect')
def handle_disconnect():
    user_id = get_authenticated_user_id()

    # Remove session
    redis_client.delete(f'user:{user_id}:session')

    # Set last seen
    redis_client.set(f'presence:user:{user_id}', 'offline')
    db.execute("UPDATE users SET is_online = false, last_seen = NOW() WHERE id = ?", [user_id])

    # Notify contacts
    notify_contacts_presence(user_id, 'offline')


def notify_contacts_presence(user_id, status):
    """Notify all contacts of user's presence change"""
    # Get all conversations this user is part of
    conversations = db.query("""
        SELECT DISTINCT conversation_id FROM conversation_members
        WHERE user_id = ?
    """, [user_id])

    for conv in conversations:
        socketio.emit('presence_update', {
            'user_id': user_id,
            'status': status,
            'last_seen': datetime.now().isoformat() if status == 'offline' else None
        }, room=f'conversation_{conv["conversation_id"]}')
```

**Typing Indicators:**

```python
@socketio.on('typing_start')
def handle_typing_start(data):
    user_id = get_authenticated_user_id()
    conversation_id = data['conversation_id']

    # Set typing indicator (expires after 5 seconds)
    redis_client.set(
        f'typing:conversation:{conversation_id}:user:{user_id}',
        '1',
        ex=5
    )

    # Notify other members
    socketio.emit('user_typing', {
        'user_id': user_id,
        'conversation_id': conversation_id
    }, room=f'conversation_{conversation_id}', skip_sid=request.sid)


@socketio.on('typing_stop')
def handle_typing_stop(data):
    user_id = get_authenticated_user_id()
    conversation_id = data['conversation_id']

    # Remove typing indicator
    redis_client.delete(f'typing:conversation:{conversation_id}:user:{user_id}')

    # Notify other members
    socketio.emit('user_stopped_typing', {
        'user_id': user_id,
        'conversation_id': conversation_id
    }, room=f'conversation_{conversation_id}', skip_sid=request.sid)
```

---

### 4.5 Message History Pagination

**Load more messages:**

```python
@app.route('/api/v1/conversations/<conversation_id>/messages', methods=['GET'])
def get_messages(conversation_id):
    user_id = get_authenticated_user_id()

    # Verify user is member of conversation
    if not is_conversation_member(user_id, conversation_id):
        return {"error": "Unauthorized"}, 403

    # Pagination parameters
    before_timestamp = request.args.get('before')  # ISO timestamp
    limit = int(request.args.get('limit', 50))

    # Try Redis cache first (recent messages)
    if not before_timestamp:
        cached_messages = redis_client.zrevrange(
            f'messages:conversation:{conversation_id}',
            0, limit - 1,
            withscores=True
        )

        if cached_messages:
            messages = [json.loads(msg) for msg, score in cached_messages]
            return {"messages": messages, "source": "cache"}

    # Query PostgreSQL (last 30 days)
    if not before_timestamp or is_recent(before_timestamp, days=30):
        messages = db.query("""
            SELECT id, sender_id, message_type, content, media_url, created_at
            FROM messages
            WHERE conversation_id = ?
            AND (? IS NULL OR created_at < ?)
            AND is_deleted = false
            ORDER BY created_at DESC
            LIMIT ?
        """, [conversation_id, before_timestamp, before_timestamp, limit])

        if messages:
            return {"messages": messages, "source": "postgresql"}

    # Query Cassandra (older messages)
    messages = cassandra_session.execute("""
        SELECT message_id, sender_id, message_type, content, media_url, created_at
        FROM message_history
        WHERE conversation_id = ?
        AND created_at < ?
        ORDER BY created_at DESC
        LIMIT ?
    """, [conversation_id, before_timestamp, limit])

    return {"messages": list(messages), "source": "cassandra"}
```

---

## 5. Scaling Strategies

### 5.1 WebSocket Server Scaling

**Problem:** Each server can handle ~10,000 concurrent connections

**Solution:** Horizontal scaling with Redis Pub/Sub

```
User A → WS Server 1
User B → WS Server 2

User A sends message to User B:
WS Server 1 → Redis Pub/Sub → WS Server 2 → User B
```

**Implementation:**

```python
import redis

redis_pubsub = redis.Redis().pubsub()
redis_pubsub.subscribe('chat_messages')

# When user sends message
def send_message_across_servers(recipient_id, message_data):
    # Publish to Redis channel
    redis_client.publish('chat_messages', json.dumps({
        'recipient_id': recipient_id,
        'message': message_data
    }))

# All WebSocket servers listen
for message in redis_pubsub.listen():
    if message['type'] == 'message':
        data = json.loads(message['data'])
        recipient_id = data['recipient_id']

        # If this server has the recipient connection, deliver it
        if has_user_connection(recipient_id):
            socketio.emit('new_message', data['message'], room=f'user_{recipient_id}')
```

**Scaling numbers:**
- 1 WebSocket server: 10,000 connections
- 100 WebSocket servers: 1,000,000 connections
- Redis Pub/Sub: Handles millions of messages/sec

---

### 5.2 Database Sharding

**Shard by conversation_id:**

```python
def get_shard_for_conversation(conversation_id):
    return conversation_id % NUM_SHARDS

# Example: 4 shards
shard = get_shard_for_conversation(12345)  # 12345 % 4 = 1
db_connection = get_db_connection(f'shard_{shard}')
```

**Why conversation_id?**
- ✅ All messages in same conversation on same shard (no cross-shard queries)
- ✅ Even distribution
- ✅ Group chats stay together

---

### 5.3 Caching Strategy

**Multi-level caching:**

```
Level 1: Redis (recent 100 messages per conversation)
Level 2: PostgreSQL (last 30 days)
Level 3: Cassandra (full history)
```

**Cache warming:**

```python
def warm_cache_for_conversation(conversation_id):
    """Pre-load recent messages when user opens conversation"""
    # Get last 100 messages from PostgreSQL
    messages = db.query("""
        SELECT * FROM messages
        WHERE conversation_id = ?
        ORDER BY created_at DESC
        LIMIT 100
    """, [conversation_id])

    # Store in Redis sorted set
    for msg in messages:
        redis_client.zadd(
            f'messages:conversation:{conversation_id}',
            {json.dumps(msg.to_dict()): msg.created_at.timestamp()}
        )

    # Set expiry (1 hour)
    redis_client.expire(f'messages:conversation:{conversation_id}', 3600)
```

---

## 6. Media Handling

### Upload Flow

```
Client → API Gateway → Media Service → S3 → CDN
```

**Implementation:**

```python
@app.route('/api/v1/media/upload', methods=['POST'])
def upload_media():
    user_id = get_authenticated_user_id()
    file = request.files['file']
    media_type = request.form.get('type')  # 'image', 'video', 'document'

    # Validate file
    if not allowed_file(file.filename, media_type):
        return {"error": "Invalid file type"}, 400

    # Generate unique filename
    file_extension = file.filename.rsplit('.', 1)[1].lower()
    filename = f"{user_id}/{uuid.uuid4()}.{file_extension}"

    # Upload to S3
    s3_client.upload_fileobj(
        file,
        BUCKET_NAME,
        filename,
        ExtraArgs={'ContentType': file.content_type}
    )

    # Generate CDN URL
    cdn_url = f"https://cdn.example.com/{filename}"

    # Create thumbnail for images/videos
    if media_type in ['image', 'video']:
        thumbnail_url = generate_thumbnail(cdn_url)
    else:
        thumbnail_url = None

    return {
        "media_url": cdn_url,
        "thumbnail_url": thumbnail_url,
        "media_type": media_type
    }

# Send message with media
@socketio.on('send_media_message')
def handle_media_message(data):
    # Media already uploaded, just send message with URL
    handle_send_message({
        'conversation_id': data['conversation_id'],
        'sender_id': data['sender_id'],
        'message_type': data['media_type'],
        'content': data.get('caption', ''),
        'media_url': data['media_url']
    })
```

---

## 7. Search Functionality

**Elasticsearch for full-text search:**

```python
from elasticsearch import Elasticsearch

es = Elasticsearch(['http://localhost:9200'])

# Index message (async worker)
def index_message(message):
    es.index(index='messages', id=message['id'], body={
        'conversation_id': message['conversation_id'],
        'sender_id': message['sender_id'],
        'content': message['content'],
        'created_at': message['created_at']
    })

# Search messages
@app.route('/api/v1/search', methods=['GET'])
def search_messages():
    user_id = get_authenticated_user_id()
    query = request.args.get('q')
    conversation_id = request.args.get('conversation_id')

    # Get user's conversations
    user_conversations = get_user_conversations(user_id)

    # Build Elasticsearch query
    es_query = {
        "query": {
            "bool": {
                "must": [
                    {"match": {"content": query}}
                ],
                "filter": [
                    {"terms": {"conversation_id": user_conversations}}
                ]
            }
        },
        "sort": [{"created_at": "desc"}],
        "size": 50
    }

    # Add conversation filter if specified
    if conversation_id:
        es_query["query"]["bool"]["filter"].append(
            {"term": {"conversation_id": conversation_id}}
        )

    # Execute search
    results = es.search(index='messages', body=es_query)

    messages = [hit['_source'] for hit in results['hits']['hits']]
    return {"results": messages, "total": results['hits']['total']['value']}
```

---

## 8. Security Considerations

| Issue | Solution |
|-------|----------|
| **Unauthorized access** | JWT tokens, WebSocket authentication |
| **Message tampering** | HTTPS/WSS, message signatures |
| **Spam/abuse** | Rate limiting (10 msg/sec per user) |
| **Media injection** | File type validation, virus scanning |
| **Account takeover** | 2FA, device verification |
| **End-to-end encryption** | Signal Protocol (optional) |
| **Data privacy** | GDPR compliance, data retention policies |

**WebSocket Authentication:**

```python
from functools import wraps
import jwt

def authenticated_socket(f):
    @wraps(f)
    def wrapped(*args, **kwargs):
        token = request.args.get('token')

        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
            request.user_id = payload['user_id']
            return f(*args, **kwargs)
        except jwt.ExpiredSignatureError:
            disconnect()
        except jwt.InvalidTokenError:
            disconnect()

    return wrapped

@socketio.on('connect')
@authenticated_socket
def handle_connect():
    user_id = request.user_id
    # ... rest of connection logic
```

---

## 9. API Design

### REST Endpoints

```python
# Authentication
POST   /api/v1/auth/register         # Register new user
POST   /api/v1/auth/login            # Login
POST   /api/v1/auth/logout           # Logout

# Conversations
GET    /api/v1/conversations         # List user's conversations
POST   /api/v1/conversations         # Create new conversation (group)
GET    /api/v1/conversations/:id     # Get conversation details
DELETE /api/v1/conversations/:id     # Delete conversation
POST   /api/v1/conversations/:id/members  # Add member to group
DELETE /api/v1/conversations/:id/members/:user_id  # Remove member

# Messages
GET    /api/v1/conversations/:id/messages  # Get message history
GET    /api/v1/search                # Search messages

# Media
POST   /api/v1/media/upload          # Upload image/video/file

# Users
GET    /api/v1/users/:id             # Get user profile
PUT    /api/v1/users/:id             # Update profile
GET    /api/v1/users/search          # Search users
```

### WebSocket Events

```javascript
// Client → Server
socket.emit('send_message', {conversation_id, content, message_type})
socket.emit('typing_start', {conversation_id})
socket.emit('typing_stop', {conversation_id})
socket.emit('message_delivered', {message_id})
socket.emit('message_read', {message_id})
socket.emit('heartbeat', {})

// Server → Client
socket.on('new_message', (message) => { /* Display message */ })
socket.on('message_status_update', ({message_id, status}) => { /* Update UI */ })
socket.on('user_typing', ({user_id, conversation_id}) => { /* Show "typing..." */ })
socket.on('user_stopped_typing', ({user_id}) => { /* Hide "typing..." */ })
socket.on('presence_update', ({user_id, status}) => { /* Update online status */ })
```

---

## 10. Capacity Estimation & Performance

### Traffic Estimation

**Assumptions:**
- 500 million registered users
- 100 million daily active users (DAU)
- Average user sends 50 messages/day
- Average message size: 200 bytes

**Calculations:**

#### Messages Per Second
```
Total daily messages = 100M DAU × 50 messages = 5 billion messages/day
Messages/second (avg) = 5B ÷ 86,400 = ~58,000 messages/sec
Peak (3x avg) = 174,000 messages/sec
```

#### WebSocket Connections
```
Concurrent users (peak) = 100M × 20% = 20 million concurrent
Each WebSocket server handles 10,000 connections
Servers needed = 20M ÷ 10,000 = 2,000 WebSocket servers
```

#### Database Writes
```
Messages/sec = 58,000 writes/sec
With 4 shards = 14,500 writes/sec per shard
```

---

### Storage Estimation

#### Messages Table (PostgreSQL - 30 days)
```
Message size = 200 bytes (avg)
Daily messages = 5 billion
30-day storage = 5B × 30 × 200 bytes = 30 TB
```

#### Message History (Cassandra - Forever)
```
Annual storage = 5B messages/day × 365 days × 200 bytes
              = 365 TB/year
5-year storage = 1.8 PB
```

#### Media Storage (S3)
```
Assume 10% of messages have media
Media per day = 5B × 10% = 500M files
Average file size = 2 MB (images/videos)
Daily storage = 500M × 2 MB = 1 PB/day
Monthly storage = 30 PB/month

With compression + CDN = ~10 PB/month
```

---

### Redis Memory

```
User sessions: 20M users × 200 bytes = 4 GB
Presence data: 20M users × 50 bytes = 1 GB
Unread counts: 100M users × 1 KB = 100 GB
Message cache: 1M active conversations × 100 messages × 500 bytes = 50 GB

Total Redis: ~160 GB (can use Redis Cluster)
```

---

### Bandwidth

```
Outgoing (message delivery):
58,000 msg/sec × 200 bytes = 11.6 MB/sec = 93 Mbps (avg)
Peak: 280 Mbps

Incoming (uploads):
58,000 msg/sec × 200 bytes = 93 Mbps (avg)
Media uploads (10%): 5,800 files/sec × 2 MB = 11.6 GB/sec = 93 Gbps

Total bandwidth: ~100 Gbps (handled by CDN)
```

---

### Server Estimation

```
WebSocket servers: 2,000 (20M connections ÷ 10K per server)
API servers: 100 (for HTTP requests)
PostgreSQL shards: 16 shards × 3 replicas = 48 DB servers
Redis Cluster: 20 nodes (8 GB RAM each)
Cassandra: 50 nodes (distributed across regions)
Kafka: 10 brokers
```

---

## 11. Interview Questions & Answers

### Q1: How do you handle message ordering in group chats?

**Answer:** Use Lamport timestamps or vector clocks.

**Simple approach:** Server-assigned timestamps

```python
# Each message gets timestamp from database
message = db.execute("""
    INSERT INTO messages (conversation_id, sender_id, content, created_at)
    VALUES (?, ?, ?, NOW())
    RETURNING id, created_at
""", [conversation_id, sender_id, content])

# Clients sort by created_at timestamp
# Single source of truth (database server time)
```

**Problem:** Clock skew across database servers

**Better approach:** Hybrid Logical Clock (HLC)

```python
class HybridLogicalClock:
    def __init__(self):
        self.physical_time = 0
        self.logical_counter = 0

    def tick(self):
        current_time = int(time.time() * 1000)  # Milliseconds

        if current_time > self.physical_time:
            self.physical_time = current_time
            self.logical_counter = 0
        else:
            self.logical_counter += 1

        return f"{self.physical_time}-{self.logical_counter}"

# Usage
hlc = HybridLogicalClock()
message_timestamp = hlc.tick()  # "1699920000-0"

# Messages sorted by: physical_time first, then logical_counter
```

**Result:** Total ordering even with clock skew

---

### Q2: How do you implement "last seen" feature?

**Answer:** Update on disconnect + background job for cleanup

```python
# When user disconnects
@socketio.on('disconnect')
def handle_disconnect():
    user_id = get_authenticated_user_id()

    # Set last_seen immediately
    db.execute("UPDATE users SET last_seen = NOW() WHERE id = ?", [user_id])

    # Also store in Redis for fast access
    redis_client.set(f'last_seen:user:{user_id}', int(time.time()), ex=86400)

# Privacy settings: Show "last seen" based on user preference
def get_last_seen(user_id, requesting_user_id):
    # Check privacy setting
    privacy = get_user_privacy_setting(user_id, 'last_seen')

    if privacy == 'everyone':
        return get_last_seen_time(user_id)
    elif privacy == 'contacts':
        if is_contact(user_id, requesting_user_id):
            return get_last_seen_time(user_id)
    elif privacy == 'nobody':
        return None

    return None
```

---

### Q3: How do you handle offline message delivery?

**Answer:** Queue messages and deliver when user comes online

```python
# When message sent to offline user
def send_to_offline_user(user_id, message):
    # 1. Store in database (already done)
    # 2. Send push notification
    send_push_notification(user_id, {
        'title': f'New message from {message["sender_name"]}',
        'body': message['content'][:100],
        'data': {
            'conversation_id': message['conversation_id'],
            'message_id': message['id']
        }
    })

# When user comes online
@socketio.on('connect')
def handle_connect():
    user_id = get_authenticated_user_id()

    # Get pending messages (undelivered)
    pending = db.query("""
        SELECT m.* FROM messages m
        JOIN message_status ms ON m.id = ms.message_id
        WHERE ms.user_id = ? AND ms.status = 'sent'
        ORDER BY m.created_at ASC
    """, [user_id])

    # Deliver all pending messages
    for message in pending:
        emit('new_message', message)

        # Update status to delivered
        db.execute("""
            UPDATE message_status
            SET status = 'delivered', timestamp = NOW()
            WHERE message_id = ? AND user_id = ?
        """, [message['id'], user_id])
```

---

### Q4: How do you prevent duplicate messages?

**Answer:** Idempotency with client-generated message IDs

```python
@socketio.on('send_message')
def handle_send_message(data):
    client_message_id = data['client_message_id']  # UUID from client
    sender_id = get_authenticated_user_id()

    # Check if already processed
    existing = redis_client.get(f'msg_dedup:{client_message_id}')
    if existing:
        # Already processed, return existing message_id
        emit('message_sent', {
            'client_message_id': client_message_id,
            'message_id': int(existing),
            'status': 'duplicate'
        })
        return

    # Process message
    message = db.execute("""
        INSERT INTO messages (conversation_id, sender_id, content, created_at)
        VALUES (?, ?, ?, NOW())
        RETURNING id
    """, [data['conversation_id'], sender_id, data['content']])

    message_id = message['id']

    # Store for deduplication (24 hour TTL)
    redis_client.setex(f'msg_dedup:{client_message_id}', 86400, message_id)

    # Rest of message sending logic...
    emit('message_sent', {
        'client_message_id': client_message_id,
        'message_id': message_id,
        'status': 'sent'
    })
```

---

### Q5: How do you implement end-to-end encryption?

**Answer:** Use Signal Protocol (Double Ratchet Algorithm)

**High-level approach:**

```python
# 1. Key Exchange (X3DH - Extended Triple Diffie-Hellman)
class KeyExchange:
    def generate_keys(self):
        # Generate identity key pair (long-term)
        identity_key_private, identity_key_public = generate_curve25519_keypair()

        # Generate signed pre-key (medium-term, rotated weekly)
        signed_prekey_private, signed_prekey_public = generate_curve25519_keypair()
        signature = sign(signed_prekey_public, identity_key_private)

        # Generate one-time pre-keys (single-use)
        onetime_prekeys = [generate_curve25519_keypair() for _ in range(100)]

        # Upload to server
        upload_keys_to_server({
            'identity_key': identity_key_public,
            'signed_prekey': signed_prekey_public,
            'signature': signature,
            'onetime_prekeys': [k[1] for k in onetime_prekeys]  # Public keys
        })

# 2. Encrypt message
def encrypt_message(plaintext, shared_secret):
    # Derive encryption key from shared secret
    encryption_key = derive_key(shared_secret)

    # Encrypt with AES-256-CBC
    iv = os.urandom(16)
    cipher = AES.new(encryption_key, AES.MODE_CBC, iv)
    ciphertext = cipher.encrypt(pad(plaintext))

    return {
        'iv': base64.encode(iv),
        'ciphertext': base64.encode(ciphertext)
    }

# 3. Send encrypted message
@socketio.on('send_encrypted_message')
def handle_encrypted_message(data):
    # Server doesn't decrypt, just forwards
    recipient_id = data['recipient_id']
    encrypted_payload = data['encrypted_payload']  # From client

    # Forward to recipient
    socketio.emit('new_encrypted_message', {
        'sender_id': get_authenticated_user_id(),
        'encrypted_payload': encrypted_payload
    }, room=f'user_{recipient_id}')
```

**Note:** Full Signal Protocol implementation is complex. Most chat apps use TLS encryption (transport-level) rather than E2E encryption.

---

### Q6: How do you handle message edits and deletions?

**Answer:** Soft deletes + edit history

```python
# Edit message
@socketio.on('edit_message')
def handle_edit_message(data):
    message_id = data['message_id']
    new_content = data['new_content']
    user_id = get_authenticated_user_id()

    # Verify ownership
    message = db.query("SELECT * FROM messages WHERE id = ?", [message_id])
    if message['sender_id'] != user_id:
        return {"error": "Unauthorized"}, 403

    # Store edit history
    db.execute("""
        INSERT INTO message_edits (message_id, old_content, edited_at)
        VALUES (?, ?, NOW())
    """, [message_id, message['content']])

    # Update message
    db.execute("""
        UPDATE messages
        SET content = ?, is_edited = true
        WHERE id = ?
    """, [new_content, message_id])

    # Notify all conversation members
    socketio.emit('message_edited', {
        'message_id': message_id,
        'new_content': new_content,
        'edited_at': datetime.now().isoformat()
    }, room=f'conversation_{message["conversation_id"]}')


# Delete message
@socketio.on('delete_message')
def handle_delete_message(data):
    message_id = data['message_id']
    user_id = get_authenticated_user_id()
    delete_for = data.get('delete_for', 'me')  # 'me' or 'everyone'

    message = db.query("SELECT * FROM messages WHERE id = ?", [message_id])

    if delete_for == 'everyone':
        # Only sender can delete for everyone (within 1 hour)
        if message['sender_id'] != user_id:
            return {"error": "Unauthorized"}, 403

        time_since_sent = (datetime.now() - message['created_at']).seconds
        if time_since_sent > 3600:  # 1 hour limit
            return {"error": "Cannot delete messages older than 1 hour"}, 400

        # Soft delete
        db.execute("UPDATE messages SET is_deleted = true WHERE id = ?", [message_id])

        # Notify everyone
        socketio.emit('message_deleted', {
            'message_id': message_id
        }, room=f'conversation_{message["conversation_id"]}')

    else:  # delete_for = 'me'
        # Just hide for this user
        db.execute("""
            INSERT INTO hidden_messages (user_id, message_id)
            VALUES (?, ?)
        """, [user_id, message_id])
```

---

### Q7: How do you implement unread message count efficiently?

**Answer:** Denormalize with Redis + database

```python
# Increment unread count when message sent
def increment_unread_count(user_id, conversation_id):
    # Redis (fast)
    redis_client.hincrby(f'unread:user:{user_id}', f'conversation:{conversation_id}', 1)

    # Also update in database (eventual consistency)
    db.execute("""
        UPDATE conversation_members
        SET unread_count = unread_count + 1
        WHERE user_id = ? AND conversation_id = ?
    """, [user_id, conversation_id])


# Get total unread count (for badge)
def get_total_unread_count(user_id):
    # Try Redis first
    counts = redis_client.hgetall(f'unread:user:{user_id}')
    if counts:
        return sum(int(v) for v in counts.values())

    # Fallback to database
    result = db.query("""
        SELECT SUM(unread_count) as total
        FROM conversation_members
        WHERE user_id = ?
    """, [user_id])

    return result['total'] or 0


# Clear unread count when user reads messages
@socketio.on('mark_as_read')
def handle_mark_as_read(data):
    user_id = get_authenticated_user_id()
    conversation_id = data['conversation_id']
    last_read_message_id = data['last_read_message_id']

    # Clear in Redis
    redis_client.hdel(f'unread:user:{user_id}', f'conversation:{conversation_id}')

    # Update in database
    db.execute("""
        UPDATE conversation_members
        SET last_read_message_id = ?,
            unread_count = 0
        WHERE user_id = ? AND conversation_id = ?
    """, [last_read_message_id, user_id, conversation_id])

    # Update all message statuses to 'read'
    db.execute("""
        UPDATE message_status
        SET status = 'read', timestamp = NOW()
        WHERE user_id = ?
        AND message_id IN (
            SELECT id FROM messages
            WHERE conversation_id = ?
            AND id <= ?
        )
    """, [user_id, conversation_id, last_read_message_id])
```

---

### Q8: How would you implement voice/video calling?

**Answer:** Use WebRTC with signaling server

**Architecture:**

```
Caller → Signaling Server (WebSocket) → Callee
    ↓                                      ↓
    └──────── WebRTC (P2P) ────────────────┘
```

**Implementation:**

```python
# Signaling server (initiate call)
@socketio.on('call_user')
def handle_call(data):
    caller_id = get_authenticated_user_id()
    callee_id = data['callee_id']
    offer = data['offer']  # WebRTC SDP offer

    # Check if callee is online
    if not is_user_online(callee_id):
        emit('call_failed', {'reason': 'User offline'})
        return

    # Forward offer to callee
    socketio.emit('incoming_call', {
        'caller_id': caller_id,
        'offer': offer
    }, room=f'user_{callee_id}')


# Callee accepts call
@socketio.on('answer_call')
def handle_answer(data):
    caller_id = data['caller_id']
    answer = data['answer']  # WebRTC SDP answer

    # Forward answer back to caller
    socketio.emit('call_answered', {
        'answer': answer
    }, room=f'user_{caller_id}')


# ICE candidate exchange (for NAT traversal)
@socketio.on('ice_candidate')
def handle_ice_candidate(data):
    peer_id = data['peer_id']
    candidate = data['candidate']

    # Forward ICE candidate to peer
    socketio.emit('ice_candidate', {
        'candidate': candidate
    }, room=f'user_{peer_id}')
```

**For group calls:** Use SFU (Selective Forwarding Unit) like Janus or Mediasoup

---

### Q9: How do you handle rate limiting for spam prevention?

**Answer:** Multi-tier rate limiting

```python
from functools import wraps

def rate_limit(key_func, limit, window):
    def decorator(f):
        @wraps(f)
        def wrapped(*args, **kwargs):
            user_id = get_authenticated_user_id()
            key = key_func(user_id, *args, **kwargs)

            # Use sliding window counter
            current_time = int(time.time())
            window_start = current_time - window

            # Remove old entries
            redis_client.zremrangebyscore(key, 0, window_start)

            # Count requests in window
            count = redis_client.zcard(key)

            if count >= limit:
                return {"error": "Rate limit exceeded"}, 429

            # Add current request
            redis_client.zadd(key, {str(current_time): current_time})
            redis_client.expire(key, window)

            return f(*args, **kwargs)

        return wrapped
    return decorator

# Apply rate limits
@socketio.on('send_message')
@rate_limit(
    key_func=lambda user_id, data: f'rate:msg:user:{user_id}',
    limit=10,  # 10 messages
    window=1   # per second
)
def handle_send_message(data):
    # ... message sending logic


# Different limits for different tiers
def get_rate_limit_for_user(user_id):
    tier = get_user_tier(user_id)

    limits = {
        'free': {'messages_per_second': 5, 'group_create_per_day': 3},
        'premium': {'messages_per_second': 20, 'group_create_per_day': 50},
        'business': {'messages_per_second': 100, 'group_create_per_day': 1000}
    }

    return limits.get(tier, limits['free'])
```

---

### Q10: How do you archive old conversations for cost savings?

**Answer:** Tiered storage with automatic archival

```python
# Background job (runs daily)
def archive_old_conversations():
    # Find conversations with no activity in 30 days
    old_conversations = db.query("""
        SELECT id FROM conversations
        WHERE updated_at < NOW() - INTERVAL '30 days'
    """)

    for conv in old_conversations:
        conversation_id = conv['id']

        # 1. Move messages from PostgreSQL to Cassandra
        messages = db.query("""
            SELECT * FROM messages
            WHERE conversation_id = ?
        """, [conversation_id])

        for msg in messages:
            cassandra_session.execute("""
                INSERT INTO message_history
                (conversation_id, created_at, message_id, sender_id, content, media_url)
                VALUES (?, ?, ?, ?, ?, ?)
            """, [msg['conversation_id'], msg['created_at'], msg['id'],
                  msg['sender_id'], msg['content'], msg['media_url']])

        # 2. Delete from PostgreSQL
        db.execute("""
            DELETE FROM messages
            WHERE conversation_id = ?
        """, [conversation_id])

        # 3. Move media to cold storage (S3 Glacier)
        media_urls = [m['media_url'] for m in messages if m['media_url']]
        for media_url in media_urls:
            move_to_glacier(media_url)

        # 4. Clear from Redis cache
        redis_client.delete(f'messages:conversation:{conversation_id}')

    print(f"Archived {len(old_conversations)} conversations")


# When user reopens old conversation
def load_archived_conversation(conversation_id):
    # Query from Cassandra
    messages = cassandra_session.execute("""
        SELECT * FROM message_history
        WHERE conversation_id = ?
        ORDER BY created_at DESC
        LIMIT 50
    """, [conversation_id])

    # Warm cache
    for msg in messages:
        redis_client.zadd(
            f'messages:conversation:{conversation_id}',
            {json.dumps(msg): msg.created_at.timestamp()}
        )

    return list(messages)
```

---

## 12. Trade-offs & Alternatives

| Decision | Chosen Approach | Alternative | Trade-off |
|----------|----------------|-------------|-----------|
| **Real-time protocol** | WebSocket | Server-Sent Events (SSE) | WebSocket: Bidirectional, more complex. SSE: Simpler, unidirectional |
| **Message storage** | PostgreSQL + Cassandra | MongoDB | PostgreSQL: ACID, better for recent data. Cassandra: Better for write-heavy history |
| **Presence** | Redis | Database polling | Redis: Real-time, fast. DB: Persistent but slower |
| **Message ordering** | Server timestamps | Lamport clocks | Server timestamps: Simpler but clock skew. Lamport: Accurate but complex |
| **Group chat** | Fan-out on read | Fan-out on write | Read: Efficient for large groups. Write: Faster delivery for small groups |
| **Encryption** | Transport (TLS) | End-to-end (E2E) | TLS: Simpler, server can search. E2E: More secure, no server access |

---

## 13. Summary

✅ **WebSocket for real-time** - Bidirectional persistent connections

✅ **Multi-tier storage** - PostgreSQL (recent) + Cassandra (history)

✅ **Redis for caching** - Sessions, presence, unread counts, recent messages

✅ **Message delivery guarantees** - Sent ✓, Delivered ✓✓, Read ✓✓ (blue)

✅ **Horizontal scaling** - WebSocket servers with Redis Pub/Sub

✅ **Database sharding** - Shard by conversation_id

✅ **Push notifications** - For offline users

✅ **Media via CDN** - S3 + CloudFront for images/videos

✅ **Search with Elasticsearch** - Full-text message search

✅ **Rate limiting** - Prevent spam (10 msg/sec)

**Key Design Decisions:**
- WebSocket + Redis Pub/Sub for real-time at scale
- Tiered storage (PostgreSQL → Cassandra → Glacier)
- Conversation-based sharding for data locality
- Idempotency with client-generated message IDs
- Fan-out on read for large groups (efficient)
- Async processing with Kafka for non-critical operations

---

**Related Concepts:**
- [WebSockets & Real-Time](concepts/websockets-realtime.md) - WebSocket scaling, Redis Pub/Sub
- [Notification Systems](concepts/notification-systems.md) - Push notifications for offline users
- [Database Sharding](concepts/database-sharding.md) - Conversation-based sharding
- [Caching Strategies](concepts/caching-strategies.md) - Multi-level caching
- [Message Queues](concepts/message-queues.md) - Kafka for async processing
- [Search & Indexing](concepts/search-indexing.md) - Elasticsearch for message search
