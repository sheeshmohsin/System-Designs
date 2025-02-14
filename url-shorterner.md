### **Detailed System Design: URL Shortener (Bit.ly, TinyURL)**  

## **1. Requirements Analysis**  

### **Functional Requirements**  
✅ Shorten a long URL to a unique short URL.  
✅ Redirect users to the original URL when accessing the short URL.  
✅ Track statistics (clicks, location, device, etc.).  
✅ Support custom short URLs.  
✅ Support URL expiry.  

### **Non-Functional Requirements**  
✅ High availability and low latency.  
✅ Scalable to handle millions of requests.  
✅ Prevent URL collisions.  
✅ Fast redirections (sub-10ms).  

## **2. High-Level Architecture**  

### **Key Components**
1. **Client (Web/App API)**
   - Users submit long URLs to shorten.
   - Users access short URLs to be redirected.

2. **API Gateway**
   - Handles HTTP requests.
   - Routes API calls to the correct service.
   - Load balances requests across backend servers.

3. **Shortening Service (Backend)**
   - Generates and stores short URLs.
   - Ensures uniqueness via **Base62 encoding**.
   - Checks for **custom URLs** and **expiry**.

4. **Database**
   - Stores **(short_key → original_url, metadata)**.
   - Supports **sharding** and **replication**.

5. **Caching (Redis)**
   - Stores **frequently accessed short URLs**.
   - Reduces database queries.

6. **Redirection Service**
   - Fetches the original URL (cache or DB).
   - Redirects user.

7. **Analytics & Logging**
   - Tracks URL usage (clicks, device, region).


## **3. Database Schema**  

Using **SQL (PostgreSQL/MySQL)** for structured data:  

```sql
CREATE TABLE urls (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_key VARCHAR(8) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NULL,
    user_id BIGINT NULL
);
```

For analytics tracking:

```sql
CREATE TABLE url_clicks (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    short_key VARCHAR(8),
    clicked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    user_agent TEXT
);
```

## **4. URL Shortening Logic**  

### **1. Generate a Unique Short Key**
#### **Base62 Encoding**
- Converts the URL ID into a **6-8 character** alphanumeric string.
- Uses **A-Z, a-z, 0-9** for compact representation.
- Example: `123456` → `dX9A3B`

#### **Hashing Algorithm (Fallback)**
- If Base62 leads to a **collision**, apply **SHA-256 + truncate**.
- Store hash collisions in **separate retry logic**.

### **2. Store in Database**
- Check if **URL exists** to avoid duplicates.
- Insert `(short_key → original_url, created_at, expires_at)`.

### **3. Redirect Users**
- On access, **check Redis** for short_key:
  - If **found**, redirect instantly.
  - If **not found**, fetch from **DB**, store in cache, redirect.


## **5. Caching Strategy (Redis)**
### **Why?**
- Reduce DB lookups.
- Handle frequent redirects efficiently.

### **What to Cache?**
- **Key:** `short_key`
- **Value:** `original_url`
- **TTL:** 24 hours (for popular links)

### **Cache Miss Handling**
- If not in cache, query DB, store result in Redis.

```python
# Pseudocode for redirect logic
def get_original_url(short_key):
    url = redis.get(short_key)
    if url:
        return url
    else:
        url = db.query("SELECT original_url FROM urls WHERE short_key = ?", short_key)
        redis.setex(short_key, 86400, url)  # Cache for 24 hours
        return url
```

---

## **6. Scaling Strategies**  

### **Database Scaling**
- **Sharding:** Short_key hash **mod(N servers)** to distribute data.
- **Read Replicas:** For high read traffic.
- **Partitioning:** Time-based partitioning for expiry cleanup.

### **Load Balancing**
- **Nginx / AWS ALB** to distribute traffic across API servers.

### **CDN & Edge Caching**
- **Cloudflare, Akamai** for global caching of static URLs.
- Edge caching improves **latency**.

### **Background Jobs (Async Processing)**
- **URL expiry cleanup** (delete expired entries).
- **Batch analytics processing** (click logs).

## **7. Security Considerations**  

| **Issue** | **Solution** |
|-----------|-------------|
| URL enumeration (predictable URLs) | Use random Base62 encoding. |
| Phishing prevention | Blacklist known spam domains. |
| Rate limiting | Use **token bucket** or **Redis Leaky Bucket**. |
| Data consistency | Use **strong consistency** for writes. |


## **8. System Design Diagram**  

Here is a **detailed system design diagram** for the URL shortener, showcasing how the components interact.

## **9. Miscellaneous**
### **9.1 Hashing Algorithm Fallback: SHA-256 + Truncate**
When using **Base62 encoding**, we may encounter **collisions** (i.e., the same short key being generated for different URLs). To handle this, we use a **fallback hashing method** such as **SHA-256** and then truncate it to obtain a unique, deterministic short key.


#### **1. Steps for Generating a Short URL with SHA-256 + Truncate**
1. **Hash the URL using SHA-256.**
2. **Convert the hash to Base62 encoding.**
3. **Truncate the hash to a fixed length (e.g., 6-8 characters).**
4. **Check for collisions in the database.**
5. If a collision exists, **append a counter** and rehash.


#### **2. Example of SHA-256 Hashing + Truncation**
##### **Example Input**
Original long URL:  
`https://www.example.com/my-very-long-url-that-needs-shortening`

##### **Step 1: Generate SHA-256 Hash**
SHA-256 of the URL:

```
3f8b31cf32a1e2d72e8adbd30f63e22c23d20a2eaf9a6a44f312f8f9f1f6236b
```

##### **Step 2: Convert to Base62**
Convert this **hexadecimal SHA-256 hash** to Base62:

```
"Gh3F2aB7kLp"
```

##### **Step 3: Truncate the Key**
Take the **first 6 characters** of the Base62 string:

```
"Gh3F2a"
```

##### **Step 4: Check for Collision**
- If `"Gh3F2a"` **already exists** in the database, append a counter (`1`, `2`, etc.), rehash, and retry.


#### **3. Pseudocode for SHA-256 + Truncate**
```python
import hashlib
import base64

def generate_short_key(url, length=6):
    # Step 1: Create SHA-256 hash of the URL
    sha256_hash = hashlib.sha256(url.encode()).hexdigest()
    
    # Step 2: Convert hex to integer
    hash_int = int(sha256_hash, 16)
    
    # Step 3: Convert integer to Base62
    base62_chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    base62_string = ""
    
    while hash_int > 0:
        base62_string = base62_chars[hash_int % 62] + base62_string
        hash_int //= 62

    # Step 4: Truncate and return
    short_key = base62_string[:length]
    return short_key

# Example Usage
long_url = "https://www.example.com/my-very-long-url-that-needs-shortening"
short_key = generate_short_key(long_url)
print("Short URL Key:", short_key)
```


#### **4. Handling Collisions**
- If a collision is detected (i.e., the short key **already exists** in the database), we:
  1. Append a **unique counter** to the original URL before hashing.
  2. Retry the hashing process.

**Example:**
1. `"https://example.com"` → `"Gh3F2a"`
2. **Collision detected** in DB.
3. Retry with `"https://example.com1"` → `"Xb7A3D"`


#### **5. Advantages of SHA-256 + Truncate**
✅ **Ensures uniqueness** (SHA-256 is cryptographically strong).  
✅ **Avoids collisions** when combined with retries.  
✅ **Fast and deterministic** (same input always gives the same short key).  


This method ensures that **short URLs remain unique and scalable**, even when handling **millions of requests**. Let me know if you need a deeper explanation!
