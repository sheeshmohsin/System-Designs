# Search & Indexing

## Overview

**Search** is one of the most critical features in modern applications. Whether it's searching products on Amazon, finding tweets on Twitter, or looking up documents in Google Docs, efficient search requires sophisticated indexing strategies.

## Why Search & Indexing?

### Problems Without Indexing

❌ **Slow queries** - Full table scans are O(n)
❌ **Poor user experience** - Users wait seconds for results
❌ **Limited functionality** - Can't do fuzzy search, typos, synonyms
❌ **High database load** - Every search hits the database hard

### Benefits of Search Indexing

✅ **Fast lookups** - O(log n) or better

✅ **Relevance ranking** - Best results first

✅ **Typo tolerance** - "ipone" finds "iphone"

✅ **Advanced features** - Filters, facets, autocomplete

---

## Search Fundamentals

### 1. Inverted Index

**The core data structure** for full-text search.

#### How It Works

Instead of mapping **document → words**, we map **word → documents**.

**Example Documents:**
```
Doc 1: "The quick brown fox"
Doc 2: "The lazy brown dog"
Doc 3: "Quick brown squirrels"
```

**Inverted Index:**
```
"the"     → [Doc 1, Doc 2]
"quick"   → [Doc 1, Doc 3]
"brown"   → [Doc 1, Doc 2, Doc 3]
"fox"     → [Doc 1]
"lazy"    → [Doc 2]
"dog"     → [Doc 2]
"squirrels" → [Doc 3]
```

**Search Query: "quick brown"**
```
"quick"  → [Doc 1, Doc 3]
"brown"  → [Doc 1, Doc 2, Doc 3]

Intersection: [Doc 1, Doc 3]  ← Results
```

#### Inverted Index Structure

```python
class InvertedIndex:
    def __init__(self):
        self.index = {}  # word → {doc_id: [positions]}

    def add_document(self, doc_id, text):
        words = tokenize(text.lower())

        for position, word in enumerate(words):
            if word not in self.index:
                self.index[word] = {}

            if doc_id not in self.index[word]:
                self.index[word][doc_id] = []

            self.index[word][doc_id].append(position)

    def search(self, query):
        words = tokenize(query.lower())
        results = None

        for word in words:
            if word in self.index:
                docs = set(self.index[word].keys())

                if results is None:
                    results = docs
                else:
                    results = results.intersection(docs)  # AND operation
            else:
                return []  # Word not found

        return list(results)

def tokenize(text):
    # Simple tokenization (real systems use stemming, lemmatization)
    return text.split()
```

---

### 2. Text Processing Pipeline

**Steps to prepare text for indexing:**

#### Step 1: Tokenization
Split text into words/tokens.

```python
text = "Hello, World! How are you?"
tokens = text.split()
# ["Hello,", "World!", "How", "are", "you?"]
```

#### Step 2: Lowercasing
Normalize case for matching.

```python
tokens = [t.lower() for t in tokens]
# ["hello,", "world!", "how", "are", "you?"]
```

#### Step 3: Punctuation Removal
```python
import string
tokens = [t.strip(string.punctuation) for t in tokens]
# ["hello", "world", "how", "are", "you"]
```

#### Step 4: Stop Word Removal
Remove common words with little meaning.

```python
STOP_WORDS = {'the', 'a', 'an', 'and', 'or', 'but', 'is', 'are', 'was', 'were'}

tokens = [t for t in tokens if t not in STOP_WORDS]
# ["hello", "world", "how", "you"]
```

#### Step 5: Stemming / Lemmatization
Reduce words to root form.

**Stemming (faster, less accurate):**
```python
"running" → "run"
"ran"     → "ran"
"runner"  → "run"
```

**Lemmatization (slower, more accurate):**
```python
"running" → "run"
"ran"     → "run"
"better"  → "good"
```

**Example:**
```python
from nltk.stem import PorterStemmer

stemmer = PorterStemmer()
tokens = [stemmer.stem(t) for t in tokens]
# ["hello", "world", "how", "you"]
```

---

### 3. TF-IDF (Term Frequency-Inverse Document Frequency)

**Ranking algorithm** to determine document relevance.

#### Formula

```
TF-IDF(term, doc) = TF(term, doc) × IDF(term)

TF(term, doc) = (Number of times term appears in doc) / (Total terms in doc)

IDF(term) = log(Total documents / Documents containing term)
```

#### Example Calculation

**Documents:**
```
Doc 1: "cat dog cat"  (3 words)
Doc 2: "dog bird"     (2 words)
Doc 3: "cat bird"     (2 words)
```

**Query: "cat"**

**TF("cat", Doc 1):**
```
= 2 / 3 = 0.67
```

**IDF("cat"):**
```
= log(3 / 2) = log(1.5) ≈ 0.18
```

**TF-IDF("cat", Doc 1):**
```
= 0.67 × 0.18 = 0.12
```

**Higher TF-IDF = More relevant**

#### Implementation

```python
import math
from collections import Counter

class TFIDF:
    def __init__(self):
        self.documents = []
        self.idf = {}

    def add_document(self, doc_id, text):
        self.documents.append((doc_id, text.split()))

    def calculate_idf(self):
        # Count documents containing each term
        df = Counter()
        for doc_id, words in self.documents:
            unique_words = set(words)
            for word in unique_words:
                df[word] += 1

        # Calculate IDF
        total_docs = len(self.documents)
        for word, count in df.items():
            self.idf[word] = math.log(total_docs / count)

    def search(self, query):
        query_words = query.split()
        scores = {}

        for doc_id, words in self.documents:
            score = 0
            word_counts = Counter(words)

            for query_word in query_words:
                if query_word in words:
                    # TF
                    tf = word_counts[query_word] / len(words)
                    # IDF
                    idf = self.idf.get(query_word, 0)
                    # TF-IDF
                    score += tf * idf

            scores[doc_id] = score

        # Sort by score (descending)
        return sorted(scores.items(), key=lambda x: x[1], reverse=True)
```

---

## Elasticsearch Basics

**Most popular search engine** built on Apache Lucene.

### Core Concepts

**Index** = Database (collection of documents)
**Type** = Table (deprecated in ES 7+)
**Document** = Row (JSON object)
**Field** = Column

### Creating an Index

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "standard"
      },
      "description": {
        "type": "text"
      },
      "price": {
        "type": "float"
      },
      "category": {
        "type": "keyword"
      },
      "tags": {
        "type": "keyword"
      },
      "created_at": {
        "type": "date"
      }
    }
  }
}
```

### Indexing Documents

```json
POST /products/_doc/1
{
  "name": "iPhone 14 Pro",
  "description": "Latest Apple smartphone with A16 chip",
  "price": 999.99,
  "category": "Electronics",
  "tags": ["phone", "apple", "smartphone"],
  "created_at": "2024-01-15T10:00:00Z"
}
```

### Basic Search

**Match Query (Full-Text Search):**
```json
GET /products/_search
{
  "query": {
    "match": {
      "name": "iphone pro"
    }
  }
}
```

**Multi-Match (Search Multiple Fields):**
```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "apple smartphone",
      "fields": ["name", "description"]
    }
  }
}
```

**Boolean Query (AND/OR/NOT):**
```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "category": "Electronics" } }
      ],
      "should": [
        { "match": { "name": "iphone" } }
      ],
      "must_not": [
        { "match": { "name": "refurbished" } }
      ],
      "filter": [
        { "range": { "price": { "gte": 500, "lte": 1500 } } }
      ]
    }
  }
}
```

### Aggregations (Facets)

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": { "field": "category" }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 100 },
          { "from": 100, "to": 500 },
          { "from": 500 }
        ]
      }
    }
  }
}
```

**Response:**
```json
{
  "aggregations": {
    "categories": {
      "buckets": [
        { "key": "Electronics", "doc_count": 150 },
        { "key": "Books", "doc_count": 80 }
      ]
    },
    "price_ranges": {
      "buckets": [
        { "key": "*-100.0", "doc_count": 200 },
        { "key": "100.0-500.0", "doc_count": 300 },
        { "key": "500.0-*", "doc_count": 100 }
      ]
    }
  }
}
```

---

## Advanced Search Features

### 1. Fuzzy Search (Typo Tolerance)

**Levenshtein distance** - allows character edits.

```json
GET /products/_search
{
  "query": {
    "match": {
      "name": {
        "query": "ipone",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

**Results:** "iPhone", "iPhone Pro" (1 character different)

**Fuzziness Levels:**
- `AUTO` - 1 edit for words 3-5 chars, 2 edits for 6+ chars
- `0` - No fuzziness
- `1` - 1 character edit
- `2` - 2 character edits

---

### 2. Autocomplete (Typeahead)

**N-grams** for prefix matching.

**Index Mapping:**
```json
PUT /autocomplete
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "autocomplete": {
            "type": "text",
            "analyzer": "autocomplete_analyzer"
          }
        }
      }
    }
  },
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "autocomplete_filter"]
        }
      },
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10
        }
      }
    }
  }
}
```

**Example:**
```
Input: "macbook"

Edge N-grams (2-10):
"ma", "mac", "macb", "macbo", "macboo", "macbook"

Query: "macb" → Matches "macbook"
```

**Search:**
```json
GET /autocomplete/_search
{
  "query": {
    "match": {
      "title.autocomplete": "macb"
    }
  }
}
```

---

### 3. Synonym Search

**Handle different terms with same meaning.**

**Synonym Configuration:**
```json
PUT /products
{
  "settings": {
    "analysis": {
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "phone, smartphone, mobile",
            "laptop, notebook, computer",
            "tv, television"
          ]
        }
      },
      "analyzer": {
        "synonym_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "synonym_filter"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "synonym_analyzer"
      }
    }
  }
}
```

**Query "phone"** → Matches "smartphone", "mobile"

---

### 4. Geospatial Search

**Search by location proximity.**

**Mapping:**
```json
PUT /restaurants
{
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "location": { "type": "geo_point" }
    }
  }
}
```

**Index Document:**
```json
POST /restaurants/_doc/1
{
  "name": "Pizza Palace",
  "location": {
    "lat": 40.7128,
    "lon": -74.0060
  }
}
```

**Search (Within Radius):**
```json
GET /restaurants/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": {
            "lat": 40.7128,
            "lon": -74.0060
          }
        }
      }
    }
  }
}
```

---

## Search Performance Optimization

### 1. Sharding

**Distribute index across multiple nodes.**

```json
PUT /products
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
```

**Benefits:**
- Parallel query execution
- Larger dataset capacity

---

### 2. Caching

**Elasticsearch caches:**
- **Query cache** - Cached query results
- **Request cache** - Cached aggregation results
- **Field data cache** - Cached field values

```json
GET /products/_search
{
  "query": { "match": { "category": "Electronics" } },
  "size": 0,
  "request_cache": true
}
```

---

### 3. Filter vs Query

**Filter:**
- Yes/No matching (binary)
- Cacheable
- Faster
- No relevance scoring

**Query:**
- Relevance scoring
- Slower
- Not cached

**Best Practice:** Use filter for exact matches, query for full-text.

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": {
        "match": { "description": "smartphone" }  // Query (scored)
      },
      "filter": {
        "term": { "category": "Electronics" }     // Filter (cached)
      }
    }
  }
}
```

---

## Database vs Search Engine

| Feature | **Database (PostgreSQL)** | **Search Engine (Elasticsearch)** |
|---------|--------------------------|-----------------------------------|
| **Purpose** | Store data | Find data |
| **Queries** | Exact matches, joins | Full-text, fuzzy, ranking |
| **Performance** | Optimized for writes | Optimized for reads |
| **Indexing** | B-tree, hash | Inverted index |
| **Consistency** | ACID (strong) | Eventually consistent |
| **Scaling** | Vertical (primary) | Horizontal (sharding) |

**Best Practice:** Use **both**
- PostgreSQL: Source of truth, transactions
- Elasticsearch: Fast search, analytics

---

## Syncing Database with Search Index

### Pattern 1: Application-Level Sync

```python
def create_product(data):
    # Write to database
    product = db.create(data)

    # Index in Elasticsearch
    es.index(index='products', id=product.id, body={
        'name': product.name,
        'description': product.description,
        'price': product.price
    })

    return product
```

**Cons:** Can get out of sync if ES fails

---

### Pattern 2: Change Data Capture (CDC)

**Stream database changes to Elasticsearch.**

```
PostgreSQL → Debezium (CDC) → Kafka → Elasticsearch Connector
```

**Pros:**
- Eventual consistency
- Database is source of truth
- Automatic sync

**Tools:** Debezium, Kafka Connect, Logstash

---

### Pattern 3: Dual Write with Reconciliation

```python
def create_product(data):
    try:
        # Write to both
        product = db.create(data)
        es.index(index='products', id=product.id, body=data)
    except Exception as e:
        # Queue for retry
        retry_queue.add(product.id)

# Background job reconciles
def reconcile():
    for product_id in retry_queue:
        product = db.get(product_id)
        es.index(index='products', id=product_id, body=product.to_dict())
```

---

## Real-World Examples

### Amazon Product Search

**Features:**
- Autocomplete
- Filters (price, category, brand)
- Faceted search
- Relevance ranking
- Personalization

**Technologies:** Elasticsearch, custom ranking algorithms

---

### Twitter Search

**Features:**
- Real-time indexing
- Hashtag search
- User mentions
- Time-based sorting

**Technologies:** Custom search infrastructure (Early Bird)

---

### Google Docs Search

**Features:**
- Full-text search across documents
- Search within document
- Recent documents ranking

**Technologies:** Proprietary search engine

---

## Summary

✅ **Inverted index** - Core data structure for search

✅ **TF-IDF** - Ranking algorithm for relevance

✅ **Elasticsearch** - Industry-standard search engine

✅ **Text processing** - Tokenization, stemming, stop words

✅ **Fuzzy search** - Handle typos with Levenshtein distance

✅ **Autocomplete** - Edge n-grams for prefix matching

✅ **Filter vs Query** - Use filter for exact, query for full-text

✅ **Sync pattern** - CDC for keeping DB and search in sync

**Golden Rule:** Use Elasticsearch for search, PostgreSQL for data!
