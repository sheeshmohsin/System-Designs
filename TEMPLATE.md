### **System Design: [System Name] ([Real-World Examples])**

[Brief 1-2 sentence introduction explaining what this system does]

## **1. Requirements Analysis**

### **Functional Requirements**
✅ [Requirement 1]

✅ [Requirement 2]

✅ [Requirement 3]

✅ [Requirement 4]

✅ [Requirement 5]

### **Non-Functional Requirements**
✅ [Performance requirement - e.g., Low latency, High throughput]

✅ [Scalability requirement - e.g., Handle millions of users]

✅ [Reliability requirement - e.g., 99.9% uptime]

✅ [Other requirements - e.g., Security, Consistency]

## **2. High-Level Architecture**

### **Key Components**
1. **[Component Name 1]**
   - [Brief description of what it does]
   - [Key responsibility]

2. **[Component Name 2]**
   - [Brief description]
   - [Key responsibility]

3. **[Component Name 3]**
   - [Brief description]
   - [Key responsibility]

[Continue with all major components...]

## **3. Database Schema**

### **SQL Database (PostgreSQL/MySQL) - For [What Data]**
```sql
CREATE TABLE [table_name] (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    [column_name] [DATA_TYPE] [CONSTRAINTS],
    [column_name] [DATA_TYPE] [CONSTRAINTS],
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### **NoSQL Database (MongoDB/Cassandra) - For [What Data]**
```json
{
  "[field_name]": "[value]",
  "[field_name]": {
    "[nested_field]": "[value]"
  },
  "[array_field]": []
}
```

**Why SQL vs NoSQL for this use case?**
- [Explain the reasoning]

## **4. [Core Logic/Workflow Name]**

### **How [Main Feature] Works**

#### **Step 1: [First Step]**
[Detailed explanation]

#### **Step 2: [Second Step]**
[Detailed explanation]

#### **Step 3: [Third Step]**
[Detailed explanation]

### **Algorithm/Implementation**

```python
# Pseudocode for [feature]
def [function_name]([parameters]):
    # Step 1: [What this does]
    [code_line]

    # Step 2: [What this does]
    [code_line]

    # Step 3: [What this does]
    return [result]

# Example Usage
[example_call]
```

## **5. Caching Strategy**

### **Why Cache?**
- [Reason 1]
- [Reason 2]

### **What to Cache?**
- **Key:** `[cache_key_pattern]`
- **Value:** `[cached_value]`
- **TTL:** [time] (e.g., 24 hours, 1 week)

### **Cache Implementation**

```python
# Pseudocode for cache logic
def [function_name]([key]):
    # Check cache first
    value = cache.get(key)
    if value:
        return value

    # Cache miss - fetch from database
    value = database.query([query])

    # Store in cache
    cache.setex(key, TTL, value)

    return value
```

## **6. Scaling Strategies**

### **Database Scaling**
- **[Strategy 1]:** [Explanation - e.g., Sharding, Replication]
- **[Strategy 2]:** [Explanation - e.g., Partitioning, Indexing]
- **[Strategy 3]:** [Explanation]

### **Application Scaling**
- **[Strategy 1]:** [Explanation - e.g., Load Balancing, Horizontal Scaling]
- **[Strategy 2]:** [Explanation - e.g., Microservices, Async Processing]

### **Storage Scaling**
- **[Strategy 1]:** [Explanation - e.g., CDN, Object Storage]
- **[Strategy 2]:** [Explanation - e.g., Cold/Hot Storage Tiers]

### **Network Scaling**
- **[Strategy 1]:** [Explanation - e.g., CDN, Edge Caching]

## **7. Security Considerations**

| **Security Issue** | **Solution** |
|-------------------|-------------|
| [Issue 1] | [Solution approach] |
| [Issue 2] | [Solution approach] |
| [Issue 3] | [Solution approach] |
| [Issue 4] | [Solution approach] |

## **8. System Design Diagram**

Here is the **detailed system architecture** diagram:

![System Design Diagram]([filename].png)

[Optional: Brief explanation of the diagram flow]

## **9. Capacity Estimation & Performance**

### **Traffic Estimation**
- **Daily Active Users (DAU):** [number]
- **Requests per Second (RPS):** [calculation]
- **Peak Traffic:** [calculation]

### **Storage Estimation**
- **Data per [unit]:** [size]
- **Total storage needed:** [calculation]
- **Storage growth rate:** [rate]

### **Bandwidth Estimation**
- **Average request size:** [size]
- **Average response size:** [size]
- **Total bandwidth:** [calculation]

## **10. Trade-offs & Alternatives**

### **Decision 1: [Technology/Approach Choice]**
**Chosen:** [What was chosen]
**Alternative:** [What else could have been used]
**Why?** [Reasoning]

### **Decision 2: [Another Key Decision]**
**Chosen:** [What was chosen]
**Alternative:** [What else could have been used]
**Why?** [Reasoning]

## **11. Potential Bottlenecks & Solutions**

| **Bottleneck** | **Impact** | **Solution** |
|---------------|-----------|-------------|
| [Bottleneck 1] | [How it affects system] | [How to mitigate] |
| [Bottleneck 2] | [How it affects system] | [How to mitigate] |

## **12. Monitoring & Observability**

### **Key Metrics to Track**
- **[Metric 1]:** [What to measure and why]
- **[Metric 2]:** [What to measure and why]
- **[Metric 3]:** [What to measure and why]

### **Alerting**
- Alert if [condition]
- Alert if [condition]

## **13. Miscellaneous Deep Dives**

For complex systems with multiple sub-topics, create a subdirectory named `[system-name]/` and link to detailed documents:

### **[Topic 1]**
[Brief description and link to detailed document if needed]

### **[Topic 2]**
[Brief description and link to detailed document if needed]

---

## **Interview Questions & Exercises**

### **Common Follow-up Questions**
1. **Q:** [Question about scaling]
   **A:** [Brief answer with key points]

2. **Q:** [Question about failure handling]
   **A:** [Brief answer with key points]

3. **Q:** [Question about trade-offs]
   **A:** [Brief answer with key points]

### **Hands-on Exercise**
Try implementing [specific component] using [technology]. Focus on:
- [Learning objective 1]
- [Learning objective 2]

---

## **Additional Resources**

- [Link to related concepts]
- [Link to technology documentation]
- [Link to similar real-world implementations]

---

## **Summary: Key Takeaways**

✅ **[Takeaway 1]**

✅ **[Takeaway 2]**

✅ **[Takeaway 3]**

✅ **[Takeaway 4]**

🚀 **This is how [Real Company 1], [Real Company 2] implement [system type]!**
