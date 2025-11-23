# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a **System Design documentation repository** containing detailed technical documentation for designing scalable distributed systems. It serves as a knowledge base for understanding real-world system architectures, design patterns, and scalability considerations.

## Repository Structure

The repository is organized as follows:

- **Root-level markdown files**: Main system design documents (e.g., `url-shorterner.md`, `video-streaming-platform.md`)
- **Subdirectories**: Supporting documentation for complex systems (e.g., `video-streaming-platform/`)
- **Diagrams**: Architecture diagrams as PNG files (e.g., `url-shortener-diag.png`)
- **`/concepts` directory**: Cross-cutting technical concepts that appear across multiple system designs
- **TEMPLATE.md**: Standard template for creating new system design documents
- **CLAUDE.md**: This file - guidance for Claude Code when working in this repository

### Key System Designs

1. **URL Shortener** (`url-shorterner.md`) - Scalable URL shortening service like Bit.ly/TinyURL
   - Base62 encoding for short key generation
   - SHA-256 fallback for collision handling
   - Database sharding strategies
   - Redis caching implementation

2. **Video Streaming Platform** (`video-streaming-platform.md`) - Netflix/YouTube-like architecture
   - Main document with high-level architecture
   - Subdirectory contains detailed breakdowns:
     - `sql-vs-nosql.md` - Database choice rationale
     - `streaming-workflow.md` - HLS, MPEG-DASH, CMAF explained
     - `video-transcoding-pipeline.md` - FFmpeg-based transcoding
     - `DRM.md` - Digital Rights Management implementation
     - `CDN-Caching.md` - Content delivery optimization

## Documentation Standards

When working with system design documents in this repository:

### File Organization

- Main system design documents should be placed at the root level
- Complex systems requiring multiple supporting documents should have a dedicated subdirectory named after the main document (without `.md`)
- Subdirectory documents should be referenced from the main document in a "Miscellaneous" section

### Document Structure

Each system design document follows this standardized format:

1. **Requirements Analysis**
   - Functional Requirements (✅ bullet points)
   - Non-Functional Requirements (✅ bullet points)

2. **High-Level Architecture**
   - Key Components (numbered list with descriptions)

3. **Database Schema**
   - SQL schemas with CREATE TABLE statements
   - NoSQL document examples in JSON format

4. **Core Workflow/Logic**
   - Algorithm explanations
   - Pseudocode examples in Python
   - Step-by-step processes

5. **Caching Strategy** (if applicable)
   - Cache structure (Key-Value pairs)
   - TTL configurations
   - Cache miss handling

6. **Scaling Strategies**
   - Database scaling (sharding, replication, partitioning)
   - Load balancing
   - CDN integration

7. **Security Considerations**
   - Presented in table format with Issue → Solution mapping

8. **System Design Diagram**
   - Reference to architecture diagram PNG file

9. **Miscellaneous**
   - Detailed deep-dives on specific topics
   - Links to subdirectory documents for complex systems
   - Mathematical calculations and examples

### Writing Style

- Use **clear section headers** with emoji indicators (🔹, ✅, ⚠️, 🚀)
- Include **code examples** in SQL, Python, JSON, or shell script format
- Provide **comparative tables** for technology choices
- Use **mathematical formulas** when explaining capacity calculations
- Include **real-world examples** (e.g., "Netflix uses...", "YouTube implements...")
- Use **step-by-step numbered workflows** for complex processes
- **IMPORTANT**: Never include AI/assistant traces in any documentation files (README.md, system designs, concepts, etc.)
  - No mentions of "Claude", "AI-generated", "assistant", "GPT", or similar terms
  - CLAUDE.md is the only exception (it's specifically for AI guidance)
  - All content should appear as human-written technical documentation

### Technical Details

- Database schemas should use appropriate SQL dialects (PostgreSQL/MySQL)
- Include specific technology choices (Redis, Nginx, AWS services, etc.)
- Provide pseudocode for algorithms and logic flows
- Show example data structures and API responses
- Include performance metrics and capacity calculations (e.g., "Base62 6-char = 56.8 billion unique URLs")

## Core Concepts Directory

The `/concepts` directory contains detailed explanations of fundamental patterns that appear across multiple system designs. When working on system designs, reference these concept documents rather than re-explaining common patterns.

### Available Concepts

1. **[Caching Strategies](concepts/caching-strategies.md)**
   - Cache-Aside, Write-Through, Write-Behind, Read-Through patterns
   - Cache eviction policies (LRU, LFU, FIFO, TTL)
   - Multi-level caching hierarchies
   - Cache key design and invalidation strategies
   - Used in: URL Shortener, Video Streaming Platform

2. **[Database Sharding](concepts/database-sharding.md)**
   - Hash-based, Range-based, Geographic, Directory-based sharding
   - Consistent hashing for minimal data movement
   - Shard key selection criteria
   - Cross-shard query challenges
   - Sharding vs Partitioning
   - Used in: URL Shortener, Multi-tenant systems

3. **[Load Balancing](concepts/load-balancing.md)**
   - Load balancing algorithms (Round Robin, Least Connections, IP Hash)
   - Layer 4 vs Layer 7 load balancing
   - Health checks, sticky sessions, SSL termination
   - Global load balancing with DNS
   - Used in: All distributed systems

### When to Create New Concept Documents

Create a new concept document when:
- A technical pattern appears in 2+ system designs
- The pattern requires detailed explanation (>500 words)
- The pattern has multiple implementation approaches
- Understanding the pattern helps with multiple interview questions

Keep concept documents:
- **Technology-agnostic** where possible
- **Example-rich** with pseudocode and real-world usage
- **Cross-referenced** to system designs that use them

## Cross-References

When creating new system designs that share components with existing ones:

- **Caching**: Link to [concepts/caching-strategies.md](concepts/caching-strategies.md) instead of re-explaining
- **Sharding**: Link to [concepts/database-sharding.md](concepts/database-sharding.md) with specific example
- **Load Balancing**: Link to [concepts/load-balancing.md](concepts/load-balancing.md) for general patterns
- **CDN**: Reference [video-streaming-platform/CDN-Caching.md](video-streaming-platform/CDN-Caching.md) for video-specific caching
- **DRM/Security**: Reference security patterns from appropriate documents

## Common Patterns

### Database Decisions
- Use **SQL** (PostgreSQL/MySQL) for structured, relational metadata requiring ACID compliance
- Use **NoSQL** (MongoDB/Cassandra) for unstructured data requiring horizontal scaling
- Use **Object Storage** (S3/GCS) for large binary files (videos, images)

### Caching Strategy
- Use **Redis** for hot data with TTL-based expiration
- Implement cache-aside pattern (check cache → DB fallback → populate cache)
- Consider CDN for static and frequently accessed content

### Scaling Approaches
- **Database**: Sharding (hash-based mod N), read replicas, time-based partitioning
- **Application**: Load balancing, horizontal scaling, microservices
- **Storage**: CDN distribution, cold/hot storage tiers
- **Processing**: Async job queues, distributed processing

## Creating New System Designs

When adding a new system design to this repository:

1. **Use TEMPLATE.md** as a starting point - it contains all required sections
2. **Follow the learning path structure**:
   - Beginner (⭐⭐): Simple, focused systems (1-2 main components)
   - Intermediate (⭐⭐⭐): Multi-component systems with caching/scaling
   - Advanced (⭐⭐⭐⭐+): Complex distributed systems with multiple data stores
3. **Create subdirectory** if the system has 3+ deep-dive topics
4. **Link to concept documents** instead of re-explaining common patterns
5. **Update README.md** to add the new design to the appropriate learning level
6. **Include real-world examples** (which companies use this pattern)
7. **Add interview questions** section if relevant

## Git Workflow

This repository uses:
- **Main branch**: `main`
- Commits follow descriptive messages (e.g., "Update video-streaming-platform.md", "Create CDN-Caching.md")
- Documents are tagged in README.md with status: ✅ (Complete), 🚧 (In Progress), 📝 (Planned)
