# **CDN Caching for Video Streaming** 🚀🎬  

## **1. Why Use a CDN for Video Streaming?**
A **Content Delivery Network (CDN)** caches video content **closer to users** to:

✅ **Reduce latency** → Faster video loading.  

✅ **Lower bandwidth costs** → Offload traffic from the origin server.  

✅ **Improve scalability** → Handle millions of concurrent users.  

✅ **Reduce storage read load** → Minimize origin server requests.  

💡 **Without a CDN:** Every user request **fetches video from the origin server** (slow + costly).  
💡 **With a CDN:** Videos are cached in **edge locations** near users for **instant playback**.


## **2. How Does a CDN Work in Video Streaming?**
🔥 **Step-by-Step Workflow:**  

1️⃣ **User requests a video (e.g., `https://cdn.example.com/videos/12345/1080p/master.m3u8`)**.  
2️⃣ The request **goes to the nearest CDN Edge Server** (instead of the origin).  
3️⃣ **If the file is cached** → The CDN **delivers it instantly** (Cache Hit ✅).  
4️⃣ **If the file is not cached** → The CDN **fetches it from the origin**, caches it, and then serves it (Cache Miss ❌).  
5️⃣ The next user **gets the cached version** (no need to fetch from origin again).  

📌 **Final Result:** Users get **fast playback** with **no buffering**! 🚀  


## **3. Types of CDN Caching for Video Streaming**
CDNs cache different parts of a video streaming pipeline:

| **Type of CDN Caching** | **Used For** | **Example CDN** |
|-----------------|----------------|--------------|
| **Static Content Caching** | Cache `.m3u8`, `.mpd`, `.ts`, `.mp4` video chunks | CloudFront, Akamai |
| **Edge Caching** | Store frequently watched videos near users | Fastly, Cloudflare |
| **Origin Shielding** | Reduce requests to main storage (AWS S3, GCS) | CloudFront Shield |
| **Tokenized URLs** | Secure access to private videos | Akamai, Limelight |

🔥 **CDN caching can store:**
- **HLS `.m3u8` playlists**
- **MPEG-DASH `.mpd` manifests**
- **Video chunks (`.ts`, `.mp4`)**
- **Thumbnails & metadata**


## **4. CDN Cache Hierarchy: How Videos Are Stored**
CDNs use a **multi-level caching strategy** to serve videos efficiently.

### **📌 4-Tier CDN Cache Strategy**
1️⃣ **User’s Local Device Cache (L1)**  
   - Some browsers **cache video chunks** (short-term).  
   - Helps with **seek/backward playback**.  

2️⃣ **CDN Edge Cache (L2 - Nearest PoP)**  
   - The **closest CDN server** caches videos for **fast delivery**.  
   - **Popular videos stay cached longer**.  

3️⃣ **Regional CDN Cache (L3 - Mid-Tier Cache)**  
   - If the Edge Cache doesn’t have the file, it **requests from a regional cache**.  
   - Reduces load on **Origin Storage**.  

4️⃣ **Origin Storage (L4 - S3, GCS, Blob Storage)**  
   - The **final source of truth** for all videos.  
   - Only accessed when the CDN doesn’t have the file.  

📌 **Example: Netflix Video Request Flow**
```
User → Edge Cache (L2) → Regional Cache (L3) → Origin Storage (L4 - AWS S3)
```
🚀 **Result:** The CDN handles most requests, keeping the **origin storage free**!



## **5. How CDN Reduces Storage Read Load**
💡 **Before CDN:** Every video request hits **AWS S3 / GCS storage** → **Slow & expensive**!  
💡 **With CDN:** Only the **first request goes to storage**, all future requests **serve from cache**!  

### **📌 Example: Bandwidth Savings**
| **Scenario** | **Without CDN** | **With CDN** |
|-------------|---------------|------------|
| **1,000 users request a 1GB video** | **1,000GB bandwidth from origin** | **Only 1GB from origin (CDN caches it)** |
| **Cache Hit Ratio** | 0% | 90-99% |
| **Storage Load** | Very High | Very Low |

📌 **Conclusion:**  
✅ **CDN reduces storage reads by 90-99%**.  

✅ **Lower egress costs (AWS S3 to the internet is expensive!).**  



## **6. Optimizing CDN Caching for Video Streaming**
To maximize **cache efficiency**, we use the following optimizations:

### **🔹 Cache Key Optimization**
- Ensure the **same video resolution & format maps to the same cache key**.
- Example Cache Key:
  ```
  /videos/12345/1080p/segment1.ts
  /videos/12345/720p/segment1.ts
  ```

### **🔹 Cache TTL (Time-to-Live) Settings**
- Popular videos should **stay cached longer**.
- Example:
  - **Trending videos:** `TTL = 7 days`
  - **Archived videos:** `TTL = 1 hour`

### **🔹 Cache Invalidation (Removing Old Files)**
- If a video **is updated**, the CDN must **remove the old cache**.
- **Methods:**
  - **Manual Purge:** `aws cloudfront create-invalidation`
  - **Cache Versioning:** Append `?v=2` to the URL (`/video.mp4?v=2`).

### **🔹 Signed URLs (Prevent Unauthorized Access)**
- **Signed URLs** ensure **only authorized users** can access premium content.
- Example:
  ```
  https://cdn.example.com/videos/1080p/master.m3u8?token=abc123
  ```


## **7. CDN Caching Strategies for Large-Scale Streaming**
| **Scenario** | **Best CDN Strategy** |
|-------------|---------------------|
| **Live Streaming (Twitch, YouTube Live)** | Low-latency caching with **Segment Prefetching** |
| **VOD (Netflix, Disney+)** | Multi-tier caching with **Long TTLs for popular videos** |
| **User-Generated Content (YouTube, TikTok)** | Edge caching + **Fast cache expiration for new uploads** |
| **Secure Streaming (Paywall Content)** | **Signed URLs + DRM protection** |


## **8. Summary: Why Use a CDN for Video Streaming?**
✅ **CDNs serve videos from cache → Reduce load on origin storage (AWS S3, GCS).**  

✅ **Reduce bandwidth costs → Fewer storage read operations.**  

✅ **Improve video startup time → No buffering!**  

✅ **Enhance scalability → Serve millions of users instantly.**  

🔥 **Without a CDN → Users experience buffering, and storage costs skyrocket!**  
🔥 **With a CDN → Instant playback, cost savings, and global scalability!**  
