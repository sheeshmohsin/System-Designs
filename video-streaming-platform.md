### **System Design: Video Streaming Platform (Netflix, YouTube, etc.)** 🎬🚀

A **video streaming platform** allows users to **upload, store, transcode, and stream videos** at different qualities with minimal latency.


## **1. Requirements Analysis**

### **🔹 Functional Requirements**
✅ **Video Upload** – Users can upload videos of various formats.  
✅ **Video Storage** – Store videos in an efficient and scalable way.  
✅ **Video Transcoding** – Convert videos into different resolutions (e.g., 1080p, 720p, 480p).  
✅ **Video Streaming** – Provide smooth playback (HLS, MPEG-DASH, CMAF).  
✅ **Content Delivery** – Low-latency video streaming across the globe using a **CDN**.  
✅ **User Authentication** – Support user-based content (public/private videos).  
✅ **Recommendation System** – Suggest videos based on user preferences.  

### **🔹 Non-Functional Requirements**
✅ **High Availability & Scalability** – Handle millions of concurrent users.  
✅ **Low Latency** – Videos should start instantly with minimal buffering.  
✅ **Efficient Storage** – Optimize costs while storing terabytes of data.  
✅ **Security** – Protect copyrighted content using DRM (Digital Rights Management).  


## **2. High-Level Architecture**

### **🔹 Key Components**
1. **Client (Web/App Player)**
   - Requests videos & streams them using HLS or DASH.

2. **API Gateway**
   - Handles user requests (upload, search, playback).
   - Routes API calls to the right service.

3. **Upload Service**
   - Accepts user video uploads.
   - Stores raw videos in **Object Storage** (AWS S3, GCS).

4. **Video Processing (Transcoding Service)**
   - Converts videos into **multiple formats (H.264, VP9, AV1)**.
   - Generates **HLS (.m3u8) or MPEG-DASH segments** for adaptive streaming.

5. **Storage System**
   - **Cold Storage (S3, Glacier)** for archived videos.
   - **Hot Storage (SSD, Block Storage)** for trending videos.

6. **Content Delivery Network (CDN)**
   - Caches video segments closer to users for **low-latency streaming**.

7. **Streaming Service**
   - Serves videos dynamically based on user device & network conditions.

8. **Metadata & Search Service**
   - Stores video metadata (title, duration, description, tags).
   - Supports **fast search & recommendations**.

9. **Authentication & DRM**
   - Controls access to premium videos.
   - Encrypts videos using **AES-128 or Widevine DRM**.

10. **Analytics & Monitoring**
   - Tracks **user engagement, watch time, and errors**.
   - Monitors **buffering rates & server health**.


## **3. Database Schema**

We will use a **combination of SQL & NoSQL**.

### **🔹 SQL (PostgreSQL / MySQL) for Metadata**
```sql
CREATE TABLE videos (
    video_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(255),
    description TEXT,
    upload_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('processing', 'ready', 'failed'),
    duration INT,
    views BIGINT DEFAULT 0
);
```

### **🔹 NoSQL (MongoDB / Cassandra) for Video Segments**
```json
{
  "video_id": "12345",
  "resolutions": {
    "1080p": "s3://bucket/video_1080p.m3u8",
    "720p": "s3://bucket/video_720p.m3u8",
    "480p": "s3://bucket/video_480p.m3u8"
  },
  "cdn_urls": {
    "us-east": "https://cdn.us/video_1080p.m3u8",
    "eu-west": "https://cdn.eu/video_1080p.m3u8"
  }
}
```


## **4. Video Transcoding Pipeline**

### **🔹 Why is Transcoding Required?**
- Videos are uploaded in **different formats & resolutions**.
- Streaming platforms need **multiple bitrates (1080p, 720p, 480p, etc.)**.
- Adaptive bitrate streaming (**HLS, MPEG-DASH**) adjusts quality dynamically.

### **🔹 Transcoding Process**
1. **User uploads video** → Stored in **Raw Storage (S3, GCS)**.
2. **Extract metadata** (FFmpeg reads codec, bitrate, resolution).
3. **Generate multiple resolutions** using **FFmpeg**.
4. **Split into chunks (HLS, MPEG-DASH segments)**.
5. **Store processed videos in CDN** for fast delivery.


## **5. Streaming Workflow**

### **🔹 Adaptive Bitrate Streaming**
- **HLS (HTTP Live Streaming)** – `.m3u8` playlist + `.ts` video chunks.
- **MPEG-DASH** – `.mpd` playlist + `.mp4` video chunks.
- **CMAF (Common Media Application Format)** – Used by **Netflix, YouTube**.

### **🔹 How Does Streaming Work?**
1. **User requests a video.**
2. The **video player requests the `.m3u8` or `.mpd` file**.
3. The player fetches **small video chunks (4s each)**.
4. If network speed drops → **Player switches to a lower quality (adaptive streaming).**


## **6. Scaling Strategies**

### **🔹 Storage Scaling**
✅ **Use Object Storage (S3, GCS)** for raw & processed videos.  
✅ **Cold Storage (Glacier)** for old, rarely accessed videos.  
✅ **CDN for caching** to reduce storage read load.  

### **🔹 Database Scaling**
✅ **Metadata DB: Partition by user_id.**  
✅ **NoSQL for fast lookups.**  

### **🔹 Traffic Scaling**
✅ **Load Balancer (Nginx, AWS ALB)** to distribute traffic.  
✅ **CDN Caching** to serve popular videos faster.  


## **7. Security Considerations**
| **Security Issue** | **Solution** |
|------------------|-------------|
| Unauthorized downloads | **DRM (AES-128, Widevine, FairPlay)** |
| DDoS attacks | **Rate limiting, WAF (Web Application Firewall)** |
| URL tampering | **Signed URLs (expiring tokens)** |


## **8. System Design Diagram**

Here is the **detailed system architecture** diagram for the **video streaming platform**. It illustrates how different components interact to handle video uploads, storage, transcoding, streaming, and CDN delivery. 

## **9. Miscellaneous**

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

