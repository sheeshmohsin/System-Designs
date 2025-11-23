# API Design Patterns

## Overview

**API (Application Programming Interface)** design is critical for building scalable, maintainable systems. This document covers the most common API architectural patterns and best practices.

## Why Good API Design Matters

✅ **Developer experience** - Easy to understand and use
✅ **Scalability** - Handle growing traffic efficiently
✅ **Maintainability** - Easy to update without breaking clients
✅ **Performance** - Minimize latency and bandwidth
✅ **Security** - Protect against unauthorized access

---

## API Architectural Styles

### 1. REST (Representational State Transfer)

**Most common API style** using HTTP methods and resources.

#### Core Principles

1. **Resource-based** - Everything is a resource (users, videos, orders)
2. **HTTP methods** - GET, POST, PUT, PATCH, DELETE
3. **Stateless** - Each request contains all necessary information
4. **JSON format** - Standard data exchange format

#### REST API Example

**Endpoints:**
```
GET    /api/v1/users           # List all users
GET    /api/v1/users/123       # Get user by ID
POST   /api/v1/users           # Create new user
PUT    /api/v1/users/123       # Update entire user
PATCH  /api/v1/users/123       # Partially update user
DELETE /api/v1/users/123       # Delete user
```

**Request:**
```http
POST /api/v1/users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

**Response:**
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15T10:30:00Z"
}
```

#### REST Best Practices

✅ **Use nouns for resources** - `/users`, not `/getUsers`
✅ **Use plural names** - `/users`, not `/user`
✅ **Use HTTP status codes properly**
  - 200 OK - Success
  - 201 Created - Resource created
  - 400 Bad Request - Invalid input
  - 401 Unauthorized - Not authenticated
  - 403 Forbidden - Authenticated but not authorized
  - 404 Not Found - Resource doesn't exist
  - 500 Internal Server Error - Server error

✅ **Versioning** - `/api/v1/users`
✅ **Pagination** - `GET /users?page=2&limit=20`
✅ **Filtering** - `GET /users?status=active&role=admin`
✅ **Sorting** - `GET /users?sort=created_at&order=desc`

**Pros:**
- ✅ Simple and widely understood
- ✅ Cacheable (HTTP caching)
- ✅ Stateless (easy to scale)

**Cons:**
- ❌ Over-fetching (get more data than needed)
- ❌ Under-fetching (need multiple requests)
- ❌ Not ideal for real-time updates

**Used in:** URL Shortener, Video Streaming Platform, most web APIs

---

### 2. GraphQL

**Query language** that lets clients request exactly the data they need.

#### How GraphQL Works

**Schema Definition:**
```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users: [User!]!
}
```

**Client Query:**
```graphql
query {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

**Response:**
```json
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [
        {"title": "First Post"},
        {"title": "Second Post"}
      ]
    }
  }
}
```

#### GraphQL Operations

**Query (Read):**
```graphql
query GetUser {
  user(id: "123") {
    name
    email
  }
}
```

**Mutation (Write):**
```graphql
mutation CreateUser {
  createUser(name: "Jane", email: "jane@example.com") {
    id
    name
  }
}
```

**Subscription (Real-time):**
```graphql
subscription OnPostAdded {
  postAdded {
    title
    author {
      name
    }
  }
}
```

**Pros:**
- ✅ No over-fetching or under-fetching
- ✅ Single endpoint for all queries
- ✅ Strong typing with schema
- ✅ Real-time updates with subscriptions

**Cons:**
- ❌ More complex than REST
- ❌ Caching is harder
- ❌ Can have performance issues with complex queries

**Used in:** Facebook, GitHub, Shopify

---

### 3. gRPC (Google Remote Procedure Call)

**High-performance RPC framework** using Protocol Buffers (protobuf).

#### Protocol Buffers Definition

```protobuf
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser (CreateUserRequest) returns (User);
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  int64 id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 limit = 2;
}

message ListUsersResponse {
  repeated User users = 1;
}
```

#### gRPC Communication Types

**1. Unary RPC (Request-Response):**
```
Client → Request → Server
Client ← Response ← Server
```

**2. Server Streaming:**
```
Client → Request → Server
Client ← Stream of responses ← Server
```

**3. Client Streaming:**
```
Client → Stream of requests → Server
Client ← Single response ← Server
```

**4. Bidirectional Streaming:**
```
Client ↔ Stream of requests/responses ↔ Server
```

**Pros:**
- ✅ Very fast (binary format, HTTP/2)
- ✅ Strong typing with protobuf
- ✅ Supports streaming
- ✅ Language-agnostic

**Cons:**
- ❌ Not human-readable (binary)
- ❌ Limited browser support
- ❌ Steeper learning curve

**Used in:** Microservices communication, high-performance systems

---

### 4. WebSockets

**Full-duplex communication** for real-time bidirectional data flow.

#### WebSocket Flow

**Handshake (HTTP Upgrade):**
```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
```

**Server Response:**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

**Communication:**
```javascript
// Client
const socket = new WebSocket('ws://example.com/chat');

socket.onopen = () => {
  socket.send(JSON.stringify({message: "Hello"}));
};

socket.onmessage = (event) => {
  console.log('Received:', event.data);
};
```

**Pros:**
- ✅ Real-time bidirectional communication
- ✅ Low latency
- ✅ Persistent connection

**Cons:**
- ❌ Harder to scale (stateful connections)
- ❌ No automatic reconnection
- ❌ Load balancing complexity

**Used in:** Chat applications, live notifications, gaming, collaborative editing

---

## API Design Patterns

### 1. Pagination

**Problem:** Returning all results is inefficient.

**Solutions:**

#### Offset-based Pagination
```http
GET /users?page=2&limit=20
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 500,
    "total_pages": 25
  }
}
```

**Cons:** Inefficient for large datasets (skips records)

#### Cursor-based Pagination
```http
GET /users?cursor=eyJpZCI6MTIzfQ&limit=20
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTQzfQ",
    "has_more": true
  }
}
```

**Pros:** Consistent results even with data changes

---

### 2. Rate Limiting

**Problem:** Prevent API abuse.

**HTTP Headers:**
```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

**Response when rate limited:**
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 3600

{
  "error": "Rate limit exceeded"
}
```

---

### 3. Filtering & Searching

**Simple Filtering:**
```http
GET /users?status=active&role=admin
```

**Advanced Search:**
```http
GET /users?q=john&fields=name,email
```

**Range Queries:**
```http
GET /orders?created_after=2024-01-01&price_min=100
```

---

### 4. Bulk Operations

**Problem:** Need to create/update multiple resources.

**Batch Create:**
```http
POST /users/batch
Content-Type: application/json

{
  "users": [
    {"name": "User1", "email": "user1@example.com"},
    {"name": "User2", "email": "user2@example.com"}
  ]
}
```

**Response:**
```json
{
  "created": [
    {"id": 1, "name": "User1"},
    {"id": 2, "name": "User2"}
  ],
  "errors": []
}
```

---

### 5. HATEOAS (Hypermedia as the Engine of Application State)

**Include links** to related resources.

**Response:**
```json
{
  "id": 123,
  "name": "John Doe",
  "links": {
    "self": "/users/123",
    "posts": "/users/123/posts",
    "delete": "/users/123"
  }
}
```

---

### 6. API Versioning

**URL Versioning:**
```
/api/v1/users
/api/v2/users
```

**Header Versioning:**
```http
Accept: application/vnd.myapi.v1+json
```

**Query Parameter:**
```
/api/users?version=1
```

**Best Practice:** URL versioning (most visible and clear)

---

### 7. Error Handling

**Standardized Error Format:**
```json
{
  "error": {
    "code": "INVALID_EMAIL",
    "message": "Email format is invalid",
    "field": "email",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

**Common Error Codes:**
- `VALIDATION_ERROR` - Input validation failed
- `RESOURCE_NOT_FOUND` - Requested resource doesn't exist
- `UNAUTHORIZED` - Authentication required
- `RATE_LIMIT_EXCEEDED` - Too many requests

---

## API Security Patterns

### 1. Authentication

#### API Keys
```http
GET /users
X-API-Key: abc123xyz
```

#### JWT (JSON Web Tokens)
```http
GET /users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### OAuth 2.0
```
1. User authorizes app
2. App receives access token
3. App uses token for API requests
```

---

### 2. HTTPS Only

Always use HTTPS for API endpoints to encrypt data in transit.

---

### 3. Input Validation

Validate all inputs to prevent injection attacks.

```python
def create_user(data):
    if not is_valid_email(data['email']):
        return {"error": "Invalid email"}, 400

    if len(data['name']) > 100:
        return {"error": "Name too long"}, 400
```

---

## Performance Optimization

### 1. Caching Headers

```http
Cache-Control: public, max-age=3600
ETag: "abc123"
```

### 2. Compression

```http
Accept-Encoding: gzip, deflate
Content-Encoding: gzip
```

### 3. Field Selection

```http
GET /users?fields=id,name,email
```

---

## API Documentation

**Use OpenAPI/Swagger** for automatic documentation.

**Example:**
```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
paths:
  /users:
    get:
      summary: List all users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
```

---

## REST vs GraphQL vs gRPC Comparison

| Feature | **REST** | **GraphQL** | **gRPC** |
|---------|---------|------------|---------|
| **Protocol** | HTTP | HTTP | HTTP/2 |
| **Data Format** | JSON | JSON | Protobuf (binary) |
| **Performance** | Medium | Medium | High |
| **Caching** | Easy | Hard | Medium |
| **Real-time** | No (need SSE/WebSockets) | Yes (subscriptions) | Yes (streaming) |
| **Browser Support** | Excellent | Excellent | Limited |
| **Learning Curve** | Easy | Medium | Hard |
| **Best For** | Public APIs, CRUD operations | Complex queries, mobile apps | Microservices, high performance |

---

## Real-World Examples

### URL Shortener API

**REST endpoints:**
```
POST   /api/v1/shorten     # Create short URL
GET    /api/v1/urls/:id    # Get URL details
GET    /:short_code        # Redirect to original URL
DELETE /api/v1/urls/:id    # Delete URL
```

See: [URL Shortener](../url-shorterner.md)

---

### Video Streaming Platform API

**REST + Streaming:**
```
POST   /api/v1/videos          # Upload video
GET    /api/v1/videos/:id      # Get video metadata
GET    /stream/:id/master.m3u8 # Get HLS playlist
POST   /api/v1/videos/:id/like # Like video
```

See: [Video Streaming Platform](../video-streaming-platform.md)

---

## Summary

✅ **REST** - Best for most web APIs (simple, cacheable)
✅ **GraphQL** - Best when clients need flexible queries
✅ **gRPC** - Best for internal microservices communication
✅ **WebSockets** - Best for real-time bidirectional communication
✅ **Use versioning** - Maintain backward compatibility
✅ **Implement rate limiting** - Prevent abuse
✅ **Document your API** - Use OpenAPI/Swagger

**Golden Rule:** Choose the API style that best fits your use case - don't over-engineer!
