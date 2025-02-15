### **9.2. Video Transcoding Pipeline - Detailed Explanation** 🎬⚡

A **video transcoding pipeline** converts raw uploaded videos into multiple formats and resolutions for **efficient streaming**.

## **1. Why is Transcoding Required?**
When users upload videos, they can be in **various formats, resolutions, and bitrates** (e.g., 4K, 1080p, AVI, MOV, MP4). However, **direct streaming is inefficient** because:
- **Not all devices support all video formats** (e.g., iPhone supports H.264 but not VP9).
- **High-resolution videos consume too much bandwidth** (e.g., 4K on slow internet = buffering).
- **Different internet speeds require adaptive quality switching** (e.g., Netflix adjusts quality dynamically).

💡 **Solution** → Convert videos into **multiple resolutions** using **Adaptive Bitrate Streaming (ABR).**  
💡 **Output Formats:** HLS (`.m3u8`), MPEG-DASH (`.mpd`), CMAF.  


## **2. Video Transcoding Process** 🚀
### **Step 1: User Uploads Video**
📌 **Raw videos are stored in an object storage** like **AWS S3, GCS, Azure Blob Storage**.

| **Input** | **Example** |
|-----------|------------|
| File Format | `.mp4`, `.avi`, `.mov`, `.mkv` |
| Resolution | `4K (2160p)`, `1080p`, `720p` |
| Codec | `H.264, VP9, AV1` |
| Bitrate | `5 Mbps, 2 Mbps, 1 Mbps` |

📌 **Storage Path Example:**
```
/raw_videos/user123/myvideo.mp4
```

### **Step 2: Extract Metadata**
- We **analyze the uploaded file** to extract **codec, resolution, bitrate, duration**.
- This helps **determine if transcoding is needed** (if it's already in the correct format, we may skip transcoding).

💡 **Tool:** `ffprobe` (from FFmpeg) extracts metadata.

```bash
ffprobe -v quiet -print_format json -show_format -show_streams myvideo.mp4
```

💡 **Example Output (Metadata Extraction)**
```json
{
  "format": {
    "duration": "300.55",
    "size": "500MB",
    "bit_rate": "4500000",
    "codec_name": "h264",
    "width": 1920,
    "height": 1080
  }
}
```

---

### **Step 3: Generate Multiple Resolutions**
📌 **Convert videos into multiple formats (H.264, VP9, AV1) and resolutions.**  
📌 **Each resolution has a different bitrate to match internet speeds.**  

| **Resolution** | **Bitrate** |
|--------------|------------|
| 1080p (Full HD) | 5 Mbps |
| 720p (HD) | 2.5 Mbps |
| 480p (SD) | 1.2 Mbps |
| 360p (Low) | 700 Kbps |

💡 **Tool:** `ffmpeg` (most popular open-source transcoder)

```bash
ffmpeg -i myvideo.mp4 -preset fast -c:v libx264 -b:v 2500k -s 1280x720 -c:a aac -b:a 128k output_720p.mp4
```

💡 **Example Transcoding Pipeline**
```bash
ffmpeg -i myvideo.mp4 \
  -preset fast -c:v libx264 -b:v 5000k -s 1920x1080 -c:a aac -b:a 128k output_1080p.mp4 \
  -preset fast -c:v libx264 -b:v 2500k -s 1280x720 -c:a aac -b:a 128k output_720p.mp4 \
  -preset fast -c:v libx264 -b:v 1200k -s 854x480 -c:a aac -b:a 96k output_480p.mp4
```

🔥 **Result:** We now have **multiple resolutions** of the same video.


### **Step 4: Split into Chunks (HLS, MPEG-DASH)**
Instead of storing a **single large file**, we **split videos into small 4-second chunks** for adaptive bitrate streaming.

📌 **Why?**
- A player can **switch quality dynamically** (e.g., from 1080p to 720p if internet slows).
- Faster **seeking and playback start time**.

📌 **HLS Format (`.m3u8`) Example**
```bash
ffmpeg -i myvideo.mp4 -c:v libx264 -b:v 2500k -s 1280x720 -hls_time 4 -hls_playlist_type vod -f hls output_720p.m3u8
```

💡 **Output Folder Structure (HLS)**
```
/videos/12345/1080p/master.m3u8
/videos/12345/1080p/segment1.ts
/videos/12345/1080p/segment2.ts
/videos/12345/720p/master.m3u8
/videos/12345/720p/segment1.ts
/videos/12345/720p/segment2.ts
```

📌 **MPEG-DASH Format (`.mpd`) Example**
```bash
ffmpeg -i myvideo.mp4 -c:v libx264 -b:v 2500k -s 1280x720 -dash 1 -f dash output.mpd
```

🔥 **Now, we have:**
- `.m3u8` (for HLS streaming)
- `.mpd` (for MPEG-DASH streaming)


### **Step 5: Store Processed Videos in CDN**
🔥 **Once transcoded, the video files are uploaded to a CDN (CloudFront, Akamai, Fastly, etc.)** for low-latency playback.

📌 **Final Storage Paths**
```
s3://processed_videos/12345/1080p/master.m3u8
s3://processed_videos/12345/720p/master.m3u8
```

📌 **CDN URLs**
```
https://cdn.example.com/videos/12345/1080p/master.m3u8
https://cdn.example.com/videos/12345/720p/master.m3u8
```

🔥 **User’s Video Player fetches these URLs** and plays them dynamically based on **network speed**.


## **5. Summary: The Complete Transcoding Pipeline**
1️⃣ **User uploads video** → Stored in **Raw Storage (S3, GCS)**.  
2️⃣ **Extract metadata** using `ffprobe` to check codec, bitrate, resolution.  
3️⃣ **Generate multiple resolutions** using **FFmpeg (1080p, 720p, 480p, etc.)**.  
4️⃣ **Split into chunks** (`.ts` for HLS, `.mp4` for MPEG-DASH).  
5️⃣ **Store in CDN** for low-latency playback.  
6️⃣ **Serve to users via streaming protocols (HLS, MPEG-DASH, CMAF).**  


## **6. Final Thought: Why This Pipeline Works?**
✅ **Scalability** – Handles millions of concurrent video streams.  
✅ **Fast Playback** – Chunks allow instant buffering & dynamic quality switching.  
✅ **Efficient Storage** – Object storage + CDN caching reduces costs.  
✅ **Adaptive Streaming** – Supports all devices and internet speeds.  

🚀 **This is how Netflix, YouTube, and Twitch process videos!**  
