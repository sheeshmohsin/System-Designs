### **9.1. Why Use SQL for Metadata & NoSQL for Video Segments?**  

A **video streaming platform** deals with **structured metadata** (titles, descriptions, upload times, etc.) and **large unstructured video segments** (HLS, MPEG-DASH chunks).  

To optimize **storage, retrieval, and scalability**, we use **SQL for metadata** and **NoSQL for video segments**. Let’s break it down:

#### **1. SQL for Video Metadata (PostgreSQL/MySQL)**  

##### **🔹 What is Video Metadata?**
Metadata contains structured **descriptive information** about each video, such as:
- Video **title, description, category**.
- **Upload date, user ID, privacy settings**.
- **Video status** (processing, ready, failed).
- **Number of views, likes, comments**.

##### **🔹 Why Use SQL for Metadata?**
✅ **Structured & Relational Data:**  
   - Metadata is well-structured and benefits from **relations between tables** (e.g., users, playlists, subscriptions).  

✅ **Transactions & Consistency:**  
   - SQL ensures **ACID (Atomicity, Consistency, Isolation, Durability)** compliance, which is important for operations like **views counting, tracking subscriptions, and recommendations**.  

✅ **Indexing & Fast Lookups:**  
   - Queries like **"Get all videos uploaded by user X"** or **"Sort by trending videos"** are **fast with indexes**.  

✅ **Joins with Other Data:**  
   - Metadata needs frequent **joins** (e.g., "Show all comments on a video").  

##### **🔹 Example Metadata Schema (PostgreSQL)**
```sql
CREATE TABLE videos (
    video_id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(255),
    description TEXT,
    upload_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('processing', 'ready', 'failed'),
    duration INT,
    views BIGINT DEFAULT 0,
    likes BIGINT DEFAULT 0
);
```

#### **2. NoSQL for Video Segments (MongoDB/Cassandra/S3)**  

##### **🔹 What are Video Segments?**
Videos are **split into small chunks (4s segments)** for adaptive bitrate streaming (**HLS, MPEG-DASH, CMAF**).

**Example Video File Storage:**
```
/videos/12345/1080p/segment1.ts
/videos/12345/1080p/segment2.ts
/videos/12345/720p/segment1.ts
/videos/12345/720p/segment2.ts
```

##### **🔹 Why Use NoSQL for Video Segments?**
✅ **Scalability (Sharding & Replication)**  
   - **Billions of videos & trillions of segments** → **NoSQL scales horizontally** across distributed servers.  
   - **Cassandra/MongoDB automatically shards data**.

✅ **Unstructured & Semi-Structured Data**  
   - Video storage paths, resolution mappings, and CDNs don’t fit into a strict relational model.

✅ **Fast Reads & Writes**  
   - Streaming requires **fast, key-value lookups** → NoSQL databases excel at this.

✅ **Distributed Storage (Eventual Consistency OK)**  
   - Video segments can be **eventually consistent** across nodes (not strict ACID like SQL).  
   - If a video is **buffered**, a small delay **won't break the system**.

##### **🔹 Example Video Segments Storage (MongoDB)**
```json
{
  "video_id": "12345",
  "resolutions": {
    "1080p": "s3://videos/12345/1080p/master.m3u8",
    "720p": "s3://videos/12345/720p/master.m3u8",
    "480p": "s3://videos/12345/480p/master.m3u8"
  },
  "cdn_urls": {
    "us-east": "https://cdn.us/video_12345/1080p.m3u8",
    "eu-west": "https://cdn.eu/video_12345/1080p.m3u8"
  }
}
```

#### **3. Summary: SQL vs. NoSQL in Video Streaming**  

| Feature            | **SQL (Metadata - PostgreSQL)** | **NoSQL (Video Segments - MongoDB, Cassandra, S3)** |
|--------------------|--------------------------------|------------------------------------------------------|
| **Data Type**       | Structured & Relational       | Semi-Structured & Unstructured |
| **Consistency**     | **Strong (ACID Compliance)**  | **Eventual Consistency (BASE)** |
| **Scaling**         | Vertical (Scaling Up)        | Horizontal (Sharding & Replication) |
| **Read/Write Speed**| Optimized for fast lookups   | Optimized for fast reads/writes |
| **Query Type**      | SQL Queries, Joins, Indexing | Simple key-value, No Joins |
| **Use Case**        | Metadata (titles, views, likes, comments) | Storing & retrieving video chunks |


#### **Final Thoughts**  
✅ **Use SQL (PostgreSQL/MySQL) for structured metadata.**  
✅ **Use NoSQL (MongoDB/Cassandra) for scalable, fast retrieval of video segments.**  
✅ **Use Object Storage (AWS S3, GCS) for actual video files.**  

🚀 **This hybrid approach balances performance, scalability, and consistency!**  
