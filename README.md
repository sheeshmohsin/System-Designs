# System Designs
Welcome to my **System Designs** repository! 🚀 This repository contains detailed system design documents covering various real-world applications and architectures.

## 📌 About This Repository

This repository serves as a **comprehensive learning resource** for understanding how to design and architect large-scale distributed systems. Each document provides:

- **Problem statement** - Real-world scenarios
- **System requirements** - Functional & non-functional
- **High-level architecture** - Component breakdown
- **Scalability considerations** - Horizontal & vertical scaling
- **Database design** - Schema design with SQL/NoSQL choices
- **API design** - RESTful endpoints and data flow
- **Caching, load balancing, and optimizations** - Performance tuning
- **Trade-offs and alternative approaches** - Why choose X over Y
- **Code examples** - Pseudocode and implementation snippets
- **Capacity estimations** - Real-world calculations

## 🎯 Learning Path

Follow this structured learning path to build your system design knowledge progressively:

### Beginner Level (Start Here!)
**Estimated Time: 2-3 hours per topic**

| Status | Topic | Key Concepts | Difficulty |
|--------|-------|--------------|------------|
| ✅ | **[URL Shortener](url-shorterner.md)** | Hashing, Base62 encoding, Redis caching, Database sharding | ⭐⭐ |

**What you'll learn:**
- Basic distributed system patterns
- Caching strategies with Redis
- Database sharding and partitioning
- Hash-based URL generation
- Handling high read traffic

---

### Intermediate Level
**Estimated Time: 4-6 hours per topic**

| Status | Topic | Key Concepts | Difficulty |
|--------|-------|--------------|------------|
| ✅ | **[Video Streaming Platform](video-streaming-platform.md)** | CDN, Video transcoding, HLS/DASH, Multi-tier caching | ⭐⭐⭐ |
| ✅ | **[Rate Limiter](rate-limiter.md)** | Token bucket, Sliding window, Redis, Distributed systems | ⭐⭐⭐ |

**What you'll learn:**
- Multi-service architecture
- Content delivery networks (CDN)
- Adaptive bitrate streaming
- Video processing pipelines
- DRM and security
- SQL vs NoSQL trade-offs

---

### Advanced Level (Coming Soon)
**Estimated Time: 6-8 hours per topic**

| Status | Topic | Key Concepts | Difficulty |
|--------|-------|--------------|------------|
| 📝 | **Ride-Sharing System (Uber)** | Geospatial indexing, WebSockets, Real-time matching | ⭐⭐⭐⭐ |
| 📝 | **Real-time Chat Application** | WebSockets, Message queues, Presence detection | ⭐⭐⭐⭐ |
| 📝 | **E-commerce System** | Inventory management, Payment processing, Order fulfillment | ⭐⭐⭐⭐ |
| 📝 | **Distributed Search Engine** | Inverted index, Ranking algorithms, Elasticsearch | ⭐⭐⭐⭐⭐ |

**Legend:**
- ✅ Complete with detailed documentation
- 🚧 In progress
- 📝 Planned

## 📖 Core Concepts Reference

Learn fundamental patterns that appear across multiple system designs:

| Concept | Where It's Used | Read More |
|---------|----------------|-----------|
| **Caching Strategies** | URL Shortener, Video Streaming | [concepts/caching-strategies.md](concepts/caching-strategies.md) |
| **Database Sharding** | URL Shortener, Chat Systems | [concepts/database-sharding.md](concepts/database-sharding.md) |
| **Load Balancing** | All Systems | [concepts/load-balancing.md](concepts/load-balancing.md) |
| **API Design Patterns** | All Systems | [concepts/api-design-patterns.md](concepts/api-design-patterns.md) |
| **Message Queues** | Video Streaming, Async Processing | [concepts/message-queues.md](concepts/message-queues.md) |
| **Consistency Patterns** | Distributed Systems | [concepts/consistency-patterns.md](concepts/consistency-patterns.md) |
| **WebSockets & Real-Time** | Chat, Live Updates | [concepts/websockets-realtime.md](concepts/websockets-realtime.md) |
| **Search & Indexing** | E-commerce, Content Platforms | [concepts/search-indexing.md](concepts/search-indexing.md) |
| **Distributed Transactions** | E-commerce, Financial Systems | [concepts/distributed-transactions.md](concepts/distributed-transactions.md) |
| **CDN Architecture** | Video Streaming, Static Content | [Video Streaming - CDN](video-streaming-platform/CDN-Caching.md) |

## 🎓 How to Use This Repository

### For Interview Preparation
1. Start with **Beginner Level** to understand core patterns
2. Practice explaining each system's architecture out loud
3. Focus on trade-offs and alternative approaches
4. Review the "Miscellaneous" sections for deep technical details

### For Learning System Design
1. Read documents in the suggested order
2. Try to design the system yourself before reading the solution
3. Compare your design with the documented approach
4. Understand the "why" behind each decision

### For Reference
1. Use the **Core Concepts** section to understand specific patterns
2. Check architecture diagrams for visual understanding
3. Review code examples for implementation guidance

## 📚 System Designs Catalog

### Completed Designs

#### 1. [URL Shortener (Bit.ly, TinyURL)](url-shorterner.md)
Design a scalable URL shortening service that handles millions of requests per day.

**Key Features:**
- Generate short URLs from long URLs
- Fast redirection (sub-10ms)
- Custom short URLs
- Click analytics and tracking
- URL expiration

**Technologies:** Redis, PostgreSQL, Base62 encoding, Consistent hashing

---

#### 2. [Video Streaming Platform (Netflix, YouTube)](video-streaming-platform.md)
Architect a distributed video streaming platform with adaptive bitrate streaming.

**Key Features:**
- Video upload and storage
- Video transcoding pipeline
- Adaptive bitrate streaming (HLS, MPEG-DASH, CMAF)
- CDN-based content delivery
- DRM and content protection

**Technologies:** AWS S3, FFmpeg, CDN (CloudFront/Akamai), MongoDB, PostgreSQL, Widevine/FairPlay DRM

**Deep Dive Topics:**
- [SQL vs NoSQL Database Choices](video-streaming-platform/sql-vs-nosql.md)
- [Video Transcoding Pipeline](video-streaming-platform/video-transcoding-pipeline.md)
- [Streaming Workflow (HLS/DASH/CMAF)](video-streaming-platform/streaming-workflow.md)
- [Digital Rights Management (DRM)](video-streaming-platform/DRM.md)
- [CDN Caching Strategies](video-streaming-platform/CDN-Caching.md)

---

#### 3. [Rate Limiter (API Throttling)](rate-limiter.md)
Design a distributed rate limiting system to prevent API abuse and ensure fair usage.

**Key Features:**
- Multiple rate limiting algorithms (Token Bucket, Leaky Bucket, Sliding Window)
- Distributed rate limiting with Redis
- Multi-tier limits (free, premium, enterprise)
- Per-IP, per-user, per-endpoint limiting
- Graceful degradation strategies

**Technologies:** Redis, Lua scripts, Token Bucket algorithm, Sliding Window Counter

**Key Concepts:**
- 5 rate limiting algorithms compared
- Distributed systems with Redis Cluster
- Atomic operations with Lua scripts
- Multi-tier and composite rate limiting

---

### Planned Designs

- **Ride-Sharing System (Uber/Lyft)**
- **Real-time Chat Application (WhatsApp/Slack)**
- **E-commerce Platform (Amazon)**
- **Social Media Feed (Twitter/Instagram)**
- **Distributed Cache (Redis Cluster)**
- **Message Queue System (Kafka/RabbitMQ)**
- **Logging and Monitoring System**
- **Notification Service**

## 🛠️ Prerequisites

To get the most out of this repository, you should have basic knowledge of:

- **Databases:** SQL (PostgreSQL/MySQL) and NoSQL (MongoDB/Redis)
- **Networking:** HTTP/HTTPS, DNS, Load Balancing
- **Programming:** Basic understanding of Python, SQL, or any programming language
- **Data Structures:** Arrays, Hash Maps, Trees
- **System Concepts:** Caching, Queues, REST APIs

Don't worry if you're not an expert in all of these - the documents explain concepts as they're used!

## 🤝 Contributing

Found a typo? Have a suggestion? Want to add a new system design?

Feel free to open an issue or submit a pull request. All contributions are welcome!

## 📬 Connect

If you found this repository helpful, please consider giving it a star ⭐

Happy Learning! 🚀
