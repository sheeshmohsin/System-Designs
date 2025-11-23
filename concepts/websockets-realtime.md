# WebSockets & Real-Time Communication

## Overview

**Real-time communication** enables instant, bidirectional data exchange between client and server. This is essential for chat applications, live notifications, collaborative editing, gaming, and live tracking systems.

## Why Real-Time Communication?

### Problems with Traditional HTTP

❌ **Request-Response only** - Client must initiate every interaction
❌ **High latency** - Need to constantly poll server for updates
❌ **Inefficient** - Each request has HTTP overhead (headers, handshake)
❌ **No server push** - Server can't notify client of changes

### Benefits of Real-Time Communication

✅ **Instant updates** - Sub-second latency

✅ **Bidirectional** - Both client and server can send messages

✅ **Efficient** - Single persistent connection

✅ **Lower overhead** - No repeated HTTP handshakes

---

## Real-Time Communication Patterns

### 1. WebSockets

**Full-duplex communication** over a single TCP connection.

#### How WebSockets Work

**1. HTTP Upgrade Handshake:**
```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**2. Server Response:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

**3. Persistent Connection Established:**
```
Client ↔ WebSocket Connection ↔ Server
  ↕                                ↕
Send/Receive messages         Send/Receive messages
```

#### WebSocket API (JavaScript)

**Client-Side:**
```javascript
// Connect to WebSocket server
const socket = new WebSocket('ws://example.com/chat');

// Connection opened
socket.onopen = (event) => {
  console.log('Connected to WebSocket');
  socket.send(JSON.stringify({
    type: 'join',
    user: 'Alice',
    room: 'general'
  }));
};

// Receive messages
socket.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
  displayMessage(message);
};

// Send messages
function sendMessage(text) {
  socket.send(JSON.stringify({
    type: 'message',
    text: text,
    timestamp: Date.now()
  }));
}

// Connection closed
socket.onclose = (event) => {
  console.log('Disconnected:', event.code, event.reason);
  // Attempt reconnection
  setTimeout(() => reconnect(), 3000);
};

// Error handling
socket.onerror = (error) => {
  console.error('WebSocket error:', error);
};
```

**Server-Side (Node.js with ws library):**
```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

// Store active connections
const clients = new Map();

wss.on('connection', (ws, req) => {
  const clientId = generateId();
  clients.set(clientId, ws);

  console.log(`Client ${clientId} connected`);

  // Handle incoming messages
  ws.on('message', (data) => {
    const message = JSON.parse(data);

    if (message.type === 'join') {
      ws.userId = message.user;
      ws.room = message.room;
      broadcast(message.room, {
        type: 'user_joined',
        user: message.user
      });
    }

    if (message.type === 'message') {
      broadcast(ws.room, {
        type: 'message',
        user: ws.userId,
        text: message.text,
        timestamp: message.timestamp
      });
    }
  });

  // Handle disconnection
  ws.on('close', () => {
    clients.delete(clientId);
    if (ws.room) {
      broadcast(ws.room, {
        type: 'user_left',
        user: ws.userId
      });
    }
  });

  // Send heartbeat
  const heartbeat = setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.ping();
    }
  }, 30000);

  ws.on('close', () => clearInterval(heartbeat));
});

// Broadcast to all clients in a room
function broadcast(room, message) {
  clients.forEach((client) => {
    if (client.room === room && client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(message));
    }
  });
}
```

#### WebSocket Message Formats

**Text Messages:**
```javascript
socket.send("Hello, World!");
```

**JSON Messages (Recommended):**
```javascript
socket.send(JSON.stringify({
  type: 'chat_message',
  payload: {
    text: 'Hello',
    user: 'Alice'
  }
}));
```

**Binary Messages:**
```javascript
const buffer = new ArrayBuffer(8);
socket.send(buffer);
```

#### Pros & Cons

**Pros:**
- ✅ True bidirectional communication
- ✅ Low latency (< 100ms)
- ✅ Efficient (no HTTP overhead after handshake)
- ✅ Works through firewalls (starts as HTTP)

**Cons:**
- ❌ Stateful (harder to scale)
- ❌ Requires sticky sessions with load balancers
- ❌ More complex than HTTP
- ❌ No automatic reconnection (must implement)

**Used in:** Slack, WhatsApp Web, Discord, Trading platforms

---

### 2. Server-Sent Events (SSE)

**One-way server push** over HTTP.

#### How SSE Works

**Client subscribes to event stream:**
```javascript
const eventSource = new EventSource('/events');

eventSource.onmessage = (event) => {
  console.log('New message:', event.data);
  updateUI(JSON.parse(event.data));
};

eventSource.addEventListener('notification', (event) => {
  console.log('Notification:', event.data);
});

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
};
```

**Server sends events:**
```javascript
// Node.js with Express
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  // Send events
  const sendEvent = (data) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // Named events
  const sendNotification = (data) => {
    res.write(`event: notification\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };

  // Send initial event
  sendEvent({ message: 'Connected to event stream' });

  // Keep connection alive with heartbeat
  const heartbeat = setInterval(() => {
    res.write(': heartbeat\n\n');
  }, 30000);

  // Cleanup on disconnect
  req.on('close', () => {
    clearInterval(heartbeat);
  });
});
```

**SSE Message Format:**
```
data: {"message": "Hello"}

event: notification
data: {"type": "alert", "text": "New message"}
id: 123
retry: 5000

data: Multi-line
data: message
data: example

```

**Pros:**
- ✅ Simple (built on HTTP)
- ✅ Automatic reconnection
- ✅ Event IDs for resuming
- ✅ Works with existing HTTP infrastructure

**Cons:**
- ❌ One-way only (server → client)
- ❌ Text-based only (no binary)
- ❌ Limited browser support (not IE)
- ❌ HTTP/1.1 connection limit (6 per domain)

**Used in:** Live notifications, stock tickers, news feeds

---

### 3. Long Polling

**Simulated real-time** using extended HTTP requests.

#### How Long Polling Works

```
1. Client sends request
2. Server holds request open until data available
3. Server responds with data
4. Client immediately sends new request
```

**Client Implementation:**
```javascript
async function longPoll() {
  try {
    const response = await fetch('/poll?last_id=' + lastMessageId, {
      method: 'GET',
      headers: { 'Content-Type': 'application/json' }
    });

    if (response.ok) {
      const data = await response.json();
      handleMessages(data.messages);
      lastMessageId = data.last_id;
    }
  } catch (error) {
    console.error('Polling error:', error);
    await sleep(5000); // Wait before retry
  }

  // Immediately poll again
  longPoll();
}

// Start polling
longPoll();
```

**Server Implementation:**
```javascript
app.get('/poll', async (req, res) => {
  const lastId = req.query.last_id;
  const timeout = 30000; // 30 seconds

  // Wait for new messages or timeout
  const messages = await waitForMessages(lastId, timeout);

  res.json({
    messages: messages,
    last_id: messages.length > 0 ? messages[messages.length - 1].id : lastId
  });
});

async function waitForMessages(lastId, timeout) {
  const startTime = Date.now();

  while (Date.now() - startTime < timeout) {
    const messages = await getNewMessages(lastId);
    if (messages.length > 0) {
      return messages;
    }
    await sleep(100); // Check every 100ms
  }

  return []; // Timeout - return empty
}
```

**Pros:**
- ✅ Works everywhere (just HTTP)
- ✅ No special server requirements
- ✅ Firewall-friendly

**Cons:**
- ❌ Higher latency than WebSockets
- ❌ More server load (constant connections)
- ❌ More bandwidth (HTTP headers on each request)

**Used in:** Older chat systems, simple notifications

---

### 4. Short Polling (Not Recommended)

**Regular HTTP requests** at fixed intervals.

```javascript
setInterval(async () => {
  const response = await fetch('/messages?since=' + lastCheck);
  const data = await response.json();
  handleMessages(data.messages);
  lastCheck = Date.now();
}, 5000); // Poll every 5 seconds
```

**Pros:**
- ✅ Very simple
- ✅ Works everywhere

**Cons:**
- ❌ High latency (up to polling interval)
- ❌ Wastes bandwidth (many empty responses)
- ❌ High server load

**Only use for:** Low-frequency updates (every 30+ seconds)

---

## Comparison Table

| Feature | **WebSockets** | **SSE** | **Long Polling** | **Short Polling** |
|---------|---------------|---------|------------------|------------------|
| **Direction** | Bidirectional | Server → Client | Bidirectional | Bidirectional |
| **Protocol** | WebSocket | HTTP | HTTP | HTTP |
| **Latency** | Very Low (<100ms) | Low (<500ms) | Medium (1-2s) | High (5-30s) |
| **Efficiency** | High | Medium | Low | Very Low |
| **Browser Support** | Excellent | Good (no IE) | Universal | Universal |
| **Reconnection** | Manual | Automatic | Manual | N/A |
| **Best For** | Chat, Gaming | Notifications | Simple real-time | Infrequent updates |

---

## Scaling WebSocket Connections

### Challenge: Stateful Connections

WebSockets are **stateful** - each connection is tied to a specific server.

**Problem:**
```
User A connects to Server 1
User B connects to Server 2
User A sends message to User B → Must route between servers
```

### Solution 1: Sticky Sessions

**Load balancer routes user to same server:**

```
Load Balancer (sticky sessions by user_id)
    ↓
Server 1: [User A, User C]
Server 2: [User B, User D]
```

**Pros:** Simple
**Cons:** Uneven load, server failure loses all connections

---

### Solution 2: Message Broker (Pub/Sub)

**Servers communicate via Redis Pub/Sub:**

```
Server 1 ←→ Redis Pub/Sub ←→ Server 2
   ↓                            ↓
User A                       User B
```

**Implementation:**
```javascript
const redis = require('redis');
const subscriber = redis.createClient();
const publisher = redis.createClient();

// Subscribe to messages for this server
subscriber.subscribe('chat_messages');

subscriber.on('message', (channel, message) => {
  const data = JSON.parse(message);

  // Send to connected clients
  clients.forEach((client) => {
    if (client.readyState === WebSocket.OPEN) {
      client.send(JSON.stringify(data));
    }
  });
});

// When receiving message from client
ws.on('message', (data) => {
  const message = JSON.parse(data);

  // Publish to Redis (all servers receive)
  publisher.publish('chat_messages', JSON.stringify(message));
});
```

**Pros:** Scales horizontally
**Cons:** Redis becomes single point of failure

---

### Solution 3: Shared State in Database

**Store message history in database:**

```
Server 1 ←→ Database ←→ Server 2
   ↓                        ↓
User A                   User B
```

**When user connects:**
```javascript
ws.on('open', async () => {
  // Load recent messages
  const messages = await db.query(
    'SELECT * FROM messages WHERE room_id = ? ORDER BY created_at DESC LIMIT 50',
    [roomId]
  );

  ws.send(JSON.stringify({ type: 'history', messages }));
});
```

---

## Heartbeat & Connection Management

### Keep-Alive (Ping/Pong)

**Server sends periodic pings:**
```javascript
const heartbeat = setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.ping();
  } else {
    clearInterval(heartbeat);
  }
}, 30000); // Every 30 seconds

ws.on('pong', () => {
  ws.isAlive = true;
});
```

**Detect dead connections:**
```javascript
setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) {
      ws.terminate(); // Close dead connection
      return;
    }
    ws.isAlive = false;
    ws.ping();
  });
}, 30000);
```

---

### Automatic Reconnection

**Client-side reconnection logic:**
```javascript
class WebSocketClient {
  constructor(url) {
    this.url = url;
    this.reconnectDelay = 1000;
    this.maxReconnectDelay = 30000;
    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectDelay = 1000; // Reset delay
    };

    this.ws.onclose = () => {
      console.log('Disconnected, reconnecting...');
      setTimeout(() => {
        this.reconnectDelay = Math.min(
          this.reconnectDelay * 2,
          this.maxReconnectDelay
        );
        this.connect();
      }, this.reconnectDelay);
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.ws.close();
    };
  }

  send(data) {
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(data);
    } else {
      console.warn('WebSocket not connected, queuing message');
      // Queue messages for retry
    }
  }
}
```

**Exponential backoff:**
- 1st retry: 1 second
- 2nd retry: 2 seconds
- 3rd retry: 4 seconds
- ...
- Max: 30 seconds

---

## Real-World Use Cases

### 1. Chat Application

**Features:**
- One-to-one messaging
- Group chats
- Typing indicators
- Read receipts
- Online status

**Technology:** WebSockets + Redis Pub/Sub

---

### 2. Live Notifications

**Features:**
- Push notifications to browser
- Unread count updates
- Real-time alerts

**Technology:** Server-Sent Events (SSE)

---

### 3. Collaborative Editing (Google Docs)

**Features:**
- Real-time text updates
- Cursor positions
- Conflict resolution

**Technology:** WebSockets + Operational Transformation (OT)

---

### 4. Live Dashboard

**Features:**
- Real-time metrics
- Live charts
- Server monitoring

**Technology:** SSE or WebSockets

---

### 5. Multiplayer Gaming

**Features:**
- Player positions
- Game state sync
- Low latency critical

**Technology:** WebSockets (or WebRTC for peer-to-peer)

---

## Security Considerations

### 1. Authentication

**Authenticate before upgrade:**
```javascript
wss.on('connection', (ws, req) => {
  const token = req.headers.authorization;

  if (!validateToken(token)) {
    ws.close(1008, 'Unauthorized');
    return;
  }

  ws.userId = getUserIdFromToken(token);
});
```

---

### 2. Rate Limiting

**Limit messages per connection:**
```javascript
const rateLimiter = new Map();

ws.on('message', (data) => {
  const userId = ws.userId;
  const now = Date.now();

  if (!rateLimiter.has(userId)) {
    rateLimiter.set(userId, []);
  }

  const timestamps = rateLimiter.get(userId);
  const recentMessages = timestamps.filter(t => now - t < 60000); // Last minute

  if (recentMessages.length >= 60) {
    ws.send(JSON.stringify({ error: 'Rate limit exceeded' }));
    return;
  }

  recentMessages.push(now);
  rateLimiter.set(userId, recentMessages);

  processMessage(data);
});
```

---

### 3. Input Validation

**Always validate message content:**
```javascript
ws.on('message', (data) => {
  let message;
  try {
    message = JSON.parse(data);
  } catch (e) {
    ws.send(JSON.stringify({ error: 'Invalid JSON' }));
    return;
  }

  if (!message.type || !message.payload) {
    ws.send(JSON.stringify({ error: 'Invalid message format' }));
    return;
  }

  // Sanitize text content
  if (message.type === 'chat') {
    message.payload.text = sanitizeHtml(message.payload.text);
  }

  processMessage(message);
});
```

---

## Monitoring & Observability

**Key Metrics:**
- Active WebSocket connections
- Messages per second
- Average message latency
- Connection duration
- Reconnection rate
- Error rate

**Example Monitoring:**
```javascript
let activeConnections = 0;
let totalMessages = 0;

wss.on('connection', (ws) => {
  activeConnections++;
  metrics.gauge('websocket.connections', activeConnections);

  ws.on('message', () => {
    totalMessages++;
    metrics.increment('websocket.messages');
  });

  ws.on('close', () => {
    activeConnections--;
    metrics.gauge('websocket.connections', activeConnections);
  });
});
```

---

## Summary

✅ **WebSockets** - Best for true bidirectional real-time (chat, gaming)

✅ **SSE** - Best for server → client updates (notifications, dashboards)

✅ **Long Polling** - Fallback when WebSockets unavailable

✅ **Redis Pub/Sub** - Scale WebSockets across multiple servers

✅ **Heartbeat** - Keep connections alive and detect failures

✅ **Exponential backoff** - Reliable reconnection strategy

✅ **Rate limiting** - Prevent abuse on WebSocket connections

**Golden Rule:** Use WebSockets for interactive real-time, SSE for one-way updates!
