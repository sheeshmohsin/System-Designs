# Geospatial Systems

## Overview

**Geospatial systems** enable location-based queries like "find nearby restaurants" or "match closest driver". These are essential for ride-sharing apps, restaurant delivery, real estate search, and location-based services.

## Why Geospatial Indexing?

### Problems Without Geospatial Indexing

❌ **Slow proximity searches** - Calculate distance to every point (O(n))
❌ **Inefficient queries** - "Find restaurants within 5km" requires full table scan
❌ **Poor scalability** - Can't handle millions of locations
❌ **High latency** - Users wait seconds for results

### Benefits

✅ **Fast proximity queries** - O(log n) or better
✅ **Range searches** - Find all points in area
✅ **Real-time matching** - Match riders with nearby drivers instantly
✅ **Scalable** - Handle billions of locations

---

## Core Concepts

### 1. Latitude & Longitude

**Coordinates** represent locations on Earth.

```
Latitude: -90° to +90° (South to North)
Longitude: -180° to +180° (West to East)

Example: San Francisco
Latitude: 37.7749° N
Longitude: -122.4194° W
```

**Representation:**
```python
location = {
    "lat": 37.7749,
    "lon": -122.4194
}
```

---

### 2. Distance Calculation

#### Haversine Formula

**Calculate distance between two points** on Earth's surface.

**Formula:**
```
a = sin²(Δlat/2) + cos(lat1) × cos(lat2) × sin²(Δlon/2)
c = 2 × atan2(√a, √(1-a))
distance = R × c

Where:
R = Earth's radius (6,371 km)
Δlat = lat2 - lat1
Δlon = lon2 - lon1
```

**Implementation:**
```python
import math

def haversine_distance(lat1, lon1, lat2, lon2):
    """
    Calculate distance between two points in kilometers
    """
    R = 6371  # Earth's radius in kilometers

    # Convert to radians
    lat1_rad = math.radians(lat1)
    lat2_rad = math.radians(lat2)
    delta_lat = math.radians(lat2 - lat1)
    delta_lon = math.radians(lon2 - lon1)

    # Haversine formula
    a = (math.sin(delta_lat / 2) ** 2 +
         math.cos(lat1_rad) * math.cos(lat2_rad) *
         math.sin(delta_lon / 2) ** 2)

    c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

    distance = R * c
    return distance

# Example
sf_lat, sf_lon = 37.7749, -122.4194
la_lat, la_lon = 34.0522, -118.2437

distance = haversine_distance(sf_lat, sf_lon, la_lat, la_lon)
print(f"Distance: {distance:.2f} km")  # ~559 km
```

**Note:** Haversine is accurate for most applications. For very precise calculations over short distances, use Vincenty formula.

---

## Geospatial Indexing Techniques

### 1. Geohashing

**Encode lat/lon into a string** that represents a geographic area.

#### How Geohashing Works

1. Divide world into grid
2. Recursively subdivide into smaller grids
3. Encode path as string (base32)

**Example:**
```
Geohash: "9q8yy"

Precision:
1 char: ±2500 km
2 chars: ±630 km
3 chars: ±78 km
4 chars: ±20 km
5 chars: ±2.4 km
6 chars: ±0.61 km
7 chars: ±0.076 km
8 chars: ±0.019 km
```

#### Geohash Properties

**1. Proximity:** Nearby locations share prefixes
```
San Francisco: 9q8yy9mf2
Oakland: 9q9p1dcp3
New York: dr5ru7p

SF and Oakland both start with "9q" (close)
NYC starts with "dr" (far)
```

**2. Hierarchical:**
```
9        → Large region (Western USA)
9q       → California
9q8      → Bay Area
9q8y     → San Francisco
9q8yy    → Downtown SF
9q8yy9   → Specific block
```

#### Geohash Implementation

```python
import geohash2

# Encode location to geohash
lat, lon = 37.7749, -122.4194
hash = geohash2.encode(lat, lon, precision=7)
print(hash)  # "9q8yy9m"

# Decode geohash to location
decoded_lat, decoded_lon = geohash2.decode(hash)
print(f"Lat: {decoded_lat}, Lon: {decoded_lon}")

# Find neighbors (8 surrounding geohashes)
neighbors = geohash2.neighbors(hash)
print(neighbors)
# ['9q8yy9q', '9q8yy9w', '9q8yy9t', '9q8yy9k', ...]
```

#### Database Schema with Geohash

```sql
CREATE TABLE restaurants (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    lat DECIMAL(9, 6),
    lon DECIMAL(9, 6),
    geohash VARCHAR(12),
    INDEX idx_geohash (geohash)
);
```

**Insert:**
```sql
INSERT INTO restaurants (name, lat, lon, geohash)
VALUES ('Pizza Palace', 37.7749, -122.4194, '9q8yy9mf2rp');
```

**Query (Find nearby restaurants):**
```sql
-- Find restaurants within same geohash prefix (approx 2.4km)
SELECT * FROM restaurants
WHERE geohash LIKE '9q8yy%'
ORDER BY geohash;
```

#### Geohash Limitations

**Edge Case Problem:**
```
Location A: geohash = 9q8yzz (edge of cell)
Location B: geohash = 9q9p00 (edge of adjacent cell)

These are VERY CLOSE but have different prefixes!
```

**Solution:** Search neighboring geohashes too
```python
def find_nearby(lat, lon, precision=5):
    center_hash = geohash2.encode(lat, lon, precision)
    neighbors = geohash2.neighbors(center_hash)

    # Search center + 8 neighbors (9 total cells)
    search_hashes = [center_hash] + neighbors

    results = []
    for hash_prefix in search_hashes:
        results.extend(
            db.query("SELECT * FROM restaurants WHERE geohash LIKE ?",
                    [hash_prefix + '%'])
        )

    return results
```

---

### 2. QuadTree

**Hierarchical spatial index** that divides space into 4 quadrants recursively.

#### How QuadTree Works

```
Start with entire map:
┌─────────────┐
│             │
│             │
│             │
└─────────────┘

Divide into 4 quadrants:
┌──────┬──────┐
│  NW  │  NE  │
├──────┼──────┤
│  SW  │  SE  │
└──────┴──────┘

Recursively divide dense quadrants:
┌───┬──┬──────┐
│NW │NE│      │
├───┼──┤  NE  │
│SW │SE│      │
└───┴──┴──────┘
```

#### QuadTree Node Structure

```python
class QuadTreeNode:
    def __init__(self, boundary, capacity=4):
        self.boundary = boundary  # (min_lat, max_lat, min_lon, max_lon)
        self.capacity = capacity  # Max points before subdividing
        self.points = []
        self.divided = False

        # Children (NW, NE, SW, SE)
        self.nw = None
        self.ne = None
        self.sw = None
        self.se = None

    def insert(self, point):
        # If point is outside boundary, reject
        if not self.contains(point):
            return False

        # If capacity not reached, add point
        if len(self.points) < self.capacity and not self.divided:
            self.points.append(point)
            return True

        # Otherwise, subdivide and insert
        if not self.divided:
            self.subdivide()

        # Try to insert into children
        if self.nw.insert(point): return True
        if self.ne.insert(point): return True
        if self.sw.insert(point): return True
        if self.se.insert(point): return True

        return False

    def subdivide(self):
        min_lat, max_lat, min_lon, max_lon = self.boundary
        mid_lat = (min_lat + max_lat) / 2
        mid_lon = (min_lon + max_lon) / 2

        # Create 4 children
        self.nw = QuadTreeNode((mid_lat, max_lat, min_lon, mid_lon))
        self.ne = QuadTreeNode((mid_lat, max_lat, mid_lon, max_lon))
        self.sw = QuadTreeNode((min_lat, mid_lat, min_lon, mid_lon))
        self.se = QuadTreeNode((min_lat, mid_lat, mid_lon, max_lon))

        self.divided = True

        # Re-insert existing points into children
        for point in self.points:
            self.insert(point)

        self.points = []

    def query_range(self, range_boundary):
        """Find all points within range"""
        results = []

        # If range doesn't intersect boundary, return empty
        if not self.intersects(range_boundary):
            return results

        # Check points in this node
        for point in self.points:
            if self.point_in_range(point, range_boundary):
                results.append(point)

        # Recursively search children
        if self.divided:
            results.extend(self.nw.query_range(range_boundary))
            results.extend(self.ne.query_range(range_boundary))
            results.extend(self.sw.query_range(range_boundary))
            results.extend(self.se.query_range(range_boundary))

        return results
```

#### QuadTree Query (Proximity Search)

```python
# Create QuadTree covering entire world
quadtree = QuadTreeNode((-90, 90, -180, 180))

# Insert restaurants
restaurants = [
    {"name": "Pizza Palace", "lat": 37.7749, "lon": -122.4194},
    {"name": "Burger Barn", "lat": 37.7849, "lon": -122.4094},
    # ... more restaurants
]

for restaurant in restaurants:
    quadtree.insert(restaurant)

# Find restaurants within 5km of user location
user_lat, user_lon = 37.7800, -122.4100

# Convert 5km to degrees (approx)
km_to_deg = 5 / 111.0  # 1 degree ≈ 111 km

range_boundary = (
    user_lat - km_to_deg,
    user_lat + km_to_deg,
    user_lon - km_to_deg,
    user_lon + km_to_deg
)

nearby = quadtree.query_range(range_boundary)
print(f"Found {len(nearby)} restaurants within 5km")
```

**Pros:**
- ✅ Dynamic (handles moving objects)
- ✅ Efficient for unevenly distributed data
- ✅ Good for in-memory indexing

**Cons:**
- ❌ Unbalanced trees possible (if points clustered)
- ❌ Not ideal for databases (hard to persist)

**Used in:** In-memory geospatial caches, game engines

---

### 3. R-Tree

**Similar to B-Tree but for spatial data** - optimized for databases.

#### How R-Tree Works

Stores **bounding boxes** (minimum rectangles) that contain points.

```
Root Node: [Contains all data]
    ↓
Level 1: [Region A] [Region B] [Region C]
    ↓
Level 2: [Points in A1] [Points in A2] [Points in B1] ...
```

**Bounding Box:**
```
Rectangle containing multiple points:
(min_lat, min_lon, max_lat, max_lon)
```

#### R-Tree in PostgreSQL (PostGIS)

**Enable PostGIS extension:**
```sql
CREATE EXTENSION postgis;
```

**Create table with geospatial column:**
```sql
CREATE TABLE drivers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    location GEOMETRY(Point, 4326)  -- WGS84 coordinate system
);

-- Create spatial index (R-Tree)
CREATE INDEX idx_drivers_location ON drivers USING GIST(location);
```

**Insert drivers:**
```sql
INSERT INTO drivers (id, name, location)
VALUES (1, 'Driver A', ST_SetSRID(ST_MakePoint(-122.4194, 37.7749), 4326));
```

**Query (Find drivers within 5km):**
```sql
SELECT id, name,
       ST_Distance(
           location::geography,
           ST_SetSRID(ST_MakePoint(-122.4100, 37.7800), 4326)::geography
       ) AS distance_meters
FROM drivers
WHERE ST_DWithin(
    location::geography,
    ST_SetSRID(ST_MakePoint(-122.4100, 37.7800), 4326)::geography,
    5000  -- 5km in meters
)
ORDER BY distance_meters;
```

**Query (Find drivers in bounding box):**
```sql
SELECT * FROM drivers
WHERE ST_Contains(
    ST_MakeEnvelope(-122.5, 37.7, -122.3, 37.9, 4326),
    location
);
```

**Pros:**
- ✅ Database-native (persisted)
- ✅ Optimized for range queries
- ✅ Production-ready (PostGIS, MySQL Spatial)

**Cons:**
- ❌ More complex than geohashing
- ❌ Requires spatial database support

**Used in:** PostgreSQL (PostGIS), MySQL (Spatial Extensions), MongoDB (2dsphere)

---

### 4. S2 Geometry (Google)

**Cell-based indexing** used by Google Maps, Uber, etc.

#### How S2 Works

1. Map Earth to cube (6 faces)
2. Project cube faces to sphere
3. Divide into hierarchical cells

**S2 Cell Properties:**
- Better coverage than geohashing (no edge issues)
- Variable cell sizes
- Efficient proximity searches

**S2 Cell IDs:**
```
Level 0: 6 cells (6 cube faces)
Level 1: 24 cells
Level 2: 96 cells
...
Level 30: ~6.5 trillion cells
```

#### S2 in Python

```python
from s2sphere import CellId, LatLng, Cell

# Create S2 cell from location
latlng = LatLng.from_degrees(37.7749, -122.4194)
cell_id = CellId.from_lat_lng(latlng)

# Get cell at specific level (higher = smaller cell)
cell_id_level_15 = cell_id.parent(15)

print(f"S2 Cell ID: {cell_id_level_15.id()}")

# Get neighboring cells
neighbors = cell_id_level_15.get_all_neighbors(15)

# Get bounding lat/lng
cell = Cell(cell_id_level_15)
rect = cell.get_rect_bound()
print(f"Min Lat: {rect.lat_lo().degrees}")
print(f"Max Lat: {rect.lat_hi().degrees}")
```

**Database Schema:**
```sql
CREATE TABLE locations (
    id BIGINT PRIMARY KEY,
    lat DECIMAL(9, 6),
    lon DECIMAL(9, 6),
    s2_cell_id BIGINT,  -- S2 cell at level 15
    INDEX idx_s2_cell (s2_cell_id)
);
```

**Query:**
```sql
-- Find locations in same or neighboring cells
SELECT * FROM locations
WHERE s2_cell_id IN (?, ?, ?, ?, ?, ?, ?, ?, ?)  -- 9 cells (center + 8 neighbors)
```

**Pros:**
- ✅ Better than geohashing (no edge issues)
- ✅ Used in production by Google, Uber
- ✅ Efficient covering algorithms

**Cons:**
- ❌ More complex than geohashing
- ❌ Less intuitive

**Used in:** Google Maps, Uber, Lyft

---

## Comparison Table

| Technique | **Query Speed** | **Precision** | **DB Support** | **Complexity** | **Best For** |
|-----------|----------------|---------------|---------------|---------------|-------------|
| **Geohashing** | Fast | Good | Excellent | Low | Simple proximity, easy to implement |
| **QuadTree** | Very Fast | Excellent | Poor | Medium | In-memory, dynamic data |
| **R-Tree** | Fast | Excellent | Excellent | Medium | Database persistence |
| **S2** | Very Fast | Excellent | Medium | High | Production systems, high scale |

---

## Real-World Use Cases

### 1. Ride-Sharing (Uber/Lyft)

**Problem:** Match rider with nearest available driver.

**Solution:**
```python
def find_nearby_drivers(rider_lat, rider_lon, radius_km=5):
    # Using S2
    rider_latlng = LatLng.from_degrees(rider_lat, rider_lon)
    rider_cell = CellId.from_lat_lng(rider_latlng).parent(15)

    # Get center + neighbor cells
    cells_to_search = [rider_cell] + rider_cell.get_all_neighbors(15)
    cell_ids = [c.id() for c in cells_to_search]

    # Query database
    drivers = db.query("""
        SELECT * FROM drivers
        WHERE s2_cell_id IN (?)
        AND status = 'available'
    """, [cell_ids])

    # Filter by exact distance
    nearby_drivers = []
    for driver in drivers:
        distance = haversine_distance(
            rider_lat, rider_lon,
            driver['lat'], driver['lon']
        )
        if distance <= radius_km:
            nearby_drivers.append({
                "driver_id": driver['id'],
                "distance_km": distance
            })

    # Sort by distance
    nearby_drivers.sort(key=lambda d: d['distance_km'])

    return nearby_drivers[:10]  # Top 10 nearest
```

---

### 2. Restaurant Discovery

**Problem:** "Find Italian restaurants within 2km"

**Solution:**
```python
def find_restaurants(user_lat, user_lon, cuisine, radius_km=2):
    # Using Geohash
    geohash = geohash2.encode(user_lat, user_lon, precision=6)  # ~0.61km
    neighbors = geohash2.neighbors(geohash)

    search_prefixes = [geohash] + neighbors

    results = []
    for prefix in search_prefixes:
        restaurants = db.query("""
            SELECT * FROM restaurants
            WHERE geohash LIKE ?
            AND cuisine = ?
        """, [prefix + '%', cuisine])

        for r in restaurants:
            distance = haversine_distance(user_lat, user_lon, r['lat'], r['lon'])
            if distance <= radius_km:
                results.append({
                    "name": r['name'],
                    "distance_km": distance
                })

    results.sort(key=lambda r: r['distance_km'])
    return results
```

---

### 3. Real Estate Search

**Problem:** "Show homes for sale in this map view"

**Solution (Bounding Box Query):**
```sql
-- Using PostGIS
SELECT * FROM properties
WHERE ST_Contains(
    ST_MakeEnvelope(
        -122.5, 37.7,  -- min_lon, min_lat (SW corner)
        -122.3, 37.9,  -- max_lon, max_lat (NE corner)
        4326
    ),
    location
)
AND status = 'for_sale';
```

---

## Performance Optimization

### 1. Spatial Indexing

**Always create spatial indexes:**
```sql
-- PostGIS (R-Tree)
CREATE INDEX idx_location ON drivers USING GIST(location);

-- Geohash
CREATE INDEX idx_geohash ON drivers(geohash);

-- S2
CREATE INDEX idx_s2_cell ON drivers(s2_cell_id);
```

---

### 2. Caching

**Cache hot areas:**
```python
# Redis with geospatial commands
import redis

r = redis.Redis()

# Add locations
r.geoadd('drivers', -122.4194, 37.7749, 'driver1')
r.geoadd('drivers', -122.4094, 37.7849, 'driver2')

# Find nearby (within 5km)
nearby = r.georadius('drivers', -122.4100, 37.7800, 5, unit='km')
print(nearby)  # ['driver1', 'driver2']

# Get distance between two locations
distance = r.geodist('drivers', 'driver1', 'driver2', unit='km')
print(f"Distance: {distance} km")
```

---

### 3. Sharding by Location

**Shard database by geographic region:**
```
Shard 1: US West Coast
Shard 2: US East Coast
Shard 3: Europe
Shard 4: Asia
```

Reduces cross-shard queries for location-based searches.

---

## Summary

✅ **Geohashing** - Simple, string-based, great for basic proximity
✅ **QuadTree** - Dynamic, in-memory, good for games/real-time
✅ **R-Tree** - Database-native, production-ready (PostGIS)
✅ **S2 Geometry** - Most advanced, used by Google/Uber
✅ **Haversine** - Accurate distance calculation
✅ **Redis GEO** - Fast geospatial caching
✅ **Spatial indexes** - Critical for performance

**Golden Rule:** Use Geohashing for simplicity, S2 for scale, PostGIS for databases!
