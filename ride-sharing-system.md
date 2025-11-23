# Ride-Sharing System (Uber/Lyft)

Design a scalable ride-sharing platform that matches riders with drivers in real-time, handles millions of concurrent users, and provides live location tracking.

---

## 1. Requirements Analysis

### Functional Requirements

✅ **Rider features:**

- Request a ride (pickup location, destination, ride type)
- View nearby drivers in real-time
- Track driver location during pickup and ride
- View estimated time of arrival (ETA) and fare
- Rate driver after ride completion
- Payment processing

✅ **Driver features:**

- Go online/offline
- Accept/reject ride requests
- Navigate to pickup location and destination
- Update location in real-time
- View earnings and ride history
- Rate rider after ride completion

✅ **Admin features:**

- Monitor active rides
- Handle disputes
- Manage surge pricing
- View analytics and metrics

### Non-Functional Requirements

✅ **Scalability** - Handle millions of concurrent users globally

✅ **Low latency** - Driver matching within 2-3 seconds, location updates < 1 second

✅ **High availability** - 99.99% uptime (critical for safety)

✅ **Consistency** - No double-booking of drivers

✅ **Real-time** - Live location tracking and updates

✅ **Accuracy** - Precise geospatial matching (within 500m radius)

✅ **Security** - Payment security, user data protection

✅ **Reliability** - No missed ride requests or failed payments

---

## 2. High-Level Architecture

### Key Components

1. **Rider App (Mobile)** - iOS/Android app for requesting rides
2. **Driver App (Mobile)** - iOS/Android app for accepting rides and navigation
3. **API Gateway** - Routes requests, handles authentication, rate limiting
4. **Ride Matching Service** - Matches riders with nearby available drivers
5. **Location Service** - Tracks and stores real-time driver/rider locations
6. **WebSocket Server** - Real-time communication for location updates and notifications
7. **Trip Service** - Manages ride lifecycle (requested, accepted, started, completed)
8. **Payment Service** - Handles payments, refunds, driver payouts
9. **Pricing Service** - Calculates fare estimates and surge pricing
10. **Notification Service** - Push notifications for ride events
11. **User Service** - User profiles, authentication, ratings
12. **Databases** - PostgreSQL (trips, users), Redis (locations, sessions), Cassandra (ride history)
13. **Message Queue** - Kafka for async processing and event streaming

### Architecture Diagram

```
┌─────────────┐          ┌─────────────┐
│  Rider App  │          │  Driver App │
└──────┬──────┘          └──────┬──────┘
       │                        │
       │   HTTPS/WebSocket      │
       └────────┬───────────────┘
                │
         ┌──────▼────────┐
         │  API Gateway  │
         │  (Nginx/Kong) │
         └──────┬────────┘
                │
    ┌───────────┼───────────────┬──────────────┐
    │           │               │              │
┌───▼────┐  ┌──▼─────┐  ┌──────▼──────┐  ┌───▼──────┐
│Location│  │  Ride  │  │   Payment   │  │   User   │
│Service │  │Matching│  │   Service   │  │ Service  │
└───┬────┘  │Service │  └──────┬──────┘  └────────┘
    │       └───┬────┘         │
    │           │              │
┌───▼──────────────────────────▼─────┐
│      WebSocket Server Cluster      │
│    (Real-time location updates)    │
└────────────────────────────────────┘
    │           │              │
┌───▼────┐  ┌──▼──────┐  ┌───▼──────┐
│ Redis  │  │Postgres │  │ Cassandra│
│(Cache) │  │(Trips)  │  │ (History)│
└────────┘  └─────────┘  └──────────┘
```

---

## 3. Database Schema

### PostgreSQL (Transactional Data)

#### Users Table

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255) NOT NULL,
    user_type VARCHAR(20) NOT NULL, -- 'rider' or 'driver'
    profile_photo_url TEXT,
    rating DECIMAL(3,2) DEFAULT 5.0,
    total_rides INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_phone (phone),
    INDEX idx_user_type (user_type)
);
```

#### Drivers Table

```sql
CREATE TABLE drivers (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    license_number VARCHAR(50) UNIQUE NOT NULL,
    vehicle_type VARCHAR(20) NOT NULL, -- 'economy', 'premium', 'suv'
    vehicle_model VARCHAR(100),
    vehicle_plate VARCHAR(20) NOT NULL,
    status VARCHAR(20) DEFAULT 'offline', -- 'online', 'offline', 'on_trip'
    current_lat DECIMAL(10,8),
    current_lng DECIMAL(11,8),
    last_location_update TIMESTAMP,
    earnings_total DECIMAL(10,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
);
```

#### Trips Table

```sql
CREATE TABLE trips (
    id BIGSERIAL PRIMARY KEY,
    rider_id BIGINT REFERENCES users(id),
    driver_id BIGINT REFERENCES drivers(id),

    -- Locations
    pickup_lat DECIMAL(10,8) NOT NULL,
    pickup_lng DECIMAL(11,8) NOT NULL,
    pickup_address TEXT,
    dropoff_lat DECIMAL(10,8) NOT NULL,
    dropoff_lng DECIMAL(11,8) NOT NULL,
    dropoff_address TEXT,

    -- Trip details
    status VARCHAR(20) NOT NULL, -- 'requested', 'accepted', 'arriving', 'started', 'completed', 'cancelled'
    ride_type VARCHAR(20) NOT NULL, -- 'economy', 'premium', 'suv'

    -- Timestamps
    requested_at TIMESTAMP DEFAULT NOW(),
    accepted_at TIMESTAMP,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,

    -- Pricing
    estimated_fare DECIMAL(10,2),
    actual_fare DECIMAL(10,2),
    surge_multiplier DECIMAL(3,2) DEFAULT 1.0,
    distance_km DECIMAL(8,2),
    duration_minutes INT,

    -- Ratings
    rider_rating SMALLINT, -- 1-5
    driver_rating SMALLINT, -- 1-5

    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_rider_id (rider_id),
    INDEX idx_driver_id (driver_id),
    INDEX idx_status (status),
    INDEX idx_requested_at (requested_at)
);
```

#### Payments Table

```sql
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    trip_id BIGINT REFERENCES trips(id),
    rider_id BIGINT REFERENCES users(id),
    amount DECIMAL(10,2) NOT NULL,
    payment_method VARCHAR(20) NOT NULL, -- 'card', 'wallet', 'cash'
    payment_status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'completed', 'failed', 'refunded'
    transaction_id VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_trip_id (trip_id),
    INDEX idx_payment_status (payment_status)
);
```

---

### Redis (Real-Time Data)

#### Driver Locations (Geospatial)

```redis
# Store driver locations using Redis GEO
GEOADD drivers:online <longitude> <latitude> <driver_id>

# Example
GEOADD drivers:online -122.4194 37.7749 driver123

# Find drivers within 5km radius
GEORADIUS drivers:online -122.4100 37.7800 5 km WITHDIST WITHCOORD

# Returns: ["driver123", "2.3km", [-122.4194, 37.7749]]
```

#### Driver Status

```redis
# Driver availability status
HSET driver:123 status "online"
HSET driver:123 vehicle_type "economy"
HSET driver:123 last_update 1699920000

# Trip assignment (prevent double-booking)
SET trip:456:driver driver123 EX 300  # 5 min expiry
```

#### Active Trips

```redis
# Track active trips
HSET trip:456 rider_id rider789
HSET trip:456 driver_id driver123
HSET trip:456 status "started"
HSET trip:456 pickup_lat 37.7749
HSET trip:456 pickup_lng -122.4194
```

---

### Cassandra (Historical Data)

#### Trip History (Time-Series)

```sql
CREATE TABLE trip_history (
    rider_id BIGINT,
    trip_id BIGINT,
    trip_date DATE,
    driver_id BIGINT,
    pickup_location TEXT,
    dropoff_location TEXT,
    fare DECIMAL,
    distance_km DECIMAL,
    duration_minutes INT,
    created_at TIMESTAMP,
    PRIMARY KEY ((rider_id), trip_date, trip_id)
) WITH CLUSTERING ORDER BY (trip_date DESC, trip_id DESC);

-- Query: Get rider's last 10 trips
SELECT * FROM trip_history
WHERE rider_id = 789
LIMIT 10;
```

#### Location Tracking History

```sql
CREATE TABLE location_tracking (
    trip_id BIGINT,
    timestamp TIMESTAMP,
    driver_id BIGINT,
    lat DECIMAL,
    lng DECIMAL,
    speed DECIMAL,
    heading INT,
    PRIMARY KEY ((trip_id), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Query: Get trip route
SELECT lat, lng, timestamp FROM location_tracking
WHERE trip_id = 456
ORDER BY timestamp ASC;
```

---

## 4. Core Algorithms

### 4.1 Driver Matching Algorithm

**Goal:** Find nearest available driver within 5km radius in < 2 seconds.

**Algorithm:**

```python
import redis
from typing import List, Dict

redis_client = redis.Redis(host='localhost', port=6379)

def find_nearby_drivers(pickup_lat: float, pickup_lng: float,
                       ride_type: str, radius_km: int = 5) -> List[Dict]:
    """
    Find available drivers near pickup location
    Returns: List of drivers sorted by distance
    """

    # 1. Query Redis GEO for drivers within radius
    nearby_drivers = redis_client.georadius(
        name='drivers:online',
        longitude=pickup_lng,
        latitude=pickup_lat,
        radius=radius_km,
        unit='km',
        withdist=True,
        withcoord=True,
        sort='ASC'  # Sort by distance (nearest first)
    )

    # 2. Filter by vehicle type and availability
    available_drivers = []
    for driver_data in nearby_drivers:
        driver_id = driver_data[0].decode('utf-8')
        distance_km = float(driver_data[1])
        coordinates = driver_data[2]

        # Check if driver is truly available (not on another trip)
        driver_info = redis_client.hgetall(f'driver:{driver_id}')

        if not driver_info:
            continue

        status = driver_info.get(b'status', b'').decode('utf-8')
        vehicle_type = driver_info.get(b'vehicle_type', b'').decode('utf-8')

        # Filter conditions
        if (status == 'online' and
            vehicle_type == ride_type and
            not redis_client.exists(f'driver:{driver_id}:assigned')):

            available_drivers.append({
                'driver_id': driver_id,
                'distance_km': distance_km,
                'lat': coordinates[1],
                'lng': coordinates[0],
                'vehicle_type': vehicle_type
            })

    # 3. Return top 5 nearest drivers
    return available_drivers[:5]


def assign_driver_to_trip(trip_id: int, driver_id: str, timeout: int = 30) -> bool:
    """
    Assign driver to trip using distributed lock to prevent double-booking
    Returns: True if assignment successful, False if driver already assigned
    """

    # Atomic operation: Try to acquire lock
    lock_key = f'driver:{driver_id}:assigned'
    lock_acquired = redis_client.set(
        lock_key,
        trip_id,
        nx=True,  # Only set if key doesn't exist
        ex=timeout  # Expire after 30 seconds if not confirmed
    )

    if lock_acquired:
        # Update driver status
        redis_client.hset(f'driver:{driver_id}', 'status', 'assigned')

        # Store trip assignment
        redis_client.hset(f'trip:{trip_id}', 'driver_id', driver_id)
        redis_client.hset(f'trip:{trip_id}', 'status', 'assigned')

        return True
    else:
        # Driver already assigned to another trip
        return False
```

---

### 4.2 ETA Calculation

**Calculate estimated time of arrival using distance and traffic.**

```python
import math

def haversine_distance(lat1: float, lng1: float, lat2: float, lng2: float) -> float:
    """
    Calculate distance between two lat/lng points in kilometers
    """
    R = 6371  # Earth's radius in km

    lat1_rad = math.radians(lat1)
    lat2_rad = math.radians(lat2)
    delta_lat = math.radians(lat2 - lat1)
    delta_lng = math.radians(lng2 - lng1)

    a = (math.sin(delta_lat / 2) ** 2 +
         math.cos(lat1_rad) * math.cos(lat2_rad) *
         math.sin(delta_lng / 2) ** 2)

    c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

    return R * c


def calculate_eta(driver_lat: float, driver_lng: float,
                 pickup_lat: float, pickup_lng: float,
                 time_of_day: str = 'normal') -> Dict:
    """
    Calculate ETA from driver to pickup location
    """

    # 1. Calculate straight-line distance
    distance_km = haversine_distance(driver_lat, driver_lng, pickup_lat, pickup_lng)

    # 2. Apply route factor (roads aren't straight)
    route_factor = 1.4  # Actual route is ~40% longer than straight line
    actual_distance_km = distance_km * route_factor

    # 3. Estimate speed based on time of day
    speed_kmh = {
        'peak_hour': 20,    # Heavy traffic
        'normal': 40,       # Normal traffic
        'late_night': 60    # Light traffic
    }.get(time_of_day, 40)

    # 4. Calculate ETA
    eta_minutes = (actual_distance_km / speed_kmh) * 60

    return {
        'distance_km': round(actual_distance_km, 2),
        'eta_minutes': round(eta_minutes, 1),
        'eta_text': f"{int(eta_minutes)} min"
    }


def calculate_fare(distance_km: float, duration_minutes: int,
                  ride_type: str, surge_multiplier: float = 1.0) -> Dict:
    """
    Calculate fare based on distance, duration, ride type, and surge pricing
    """

    # Base pricing per ride type
    pricing = {
        'economy': {
            'base_fare': 2.50,
            'per_km': 1.50,
            'per_minute': 0.30,
            'minimum_fare': 5.00
        },
        'premium': {
            'base_fare': 5.00,
            'per_km': 2.50,
            'per_minute': 0.50,
            'minimum_fare': 10.00
        },
        'suv': {
            'base_fare': 7.00,
            'per_km': 3.00,
            'per_minute': 0.60,
            'minimum_fare': 15.00
        }
    }

    config = pricing.get(ride_type, pricing['economy'])

    # Calculate fare components
    base_fare = config['base_fare']
    distance_fare = distance_km * config['per_km']
    time_fare = duration_minutes * config['per_minute']

    # Subtotal
    subtotal = base_fare + distance_fare + time_fare

    # Apply minimum fare
    subtotal = max(subtotal, config['minimum_fare'])

    # Apply surge pricing
    total_fare = subtotal * surge_multiplier

    return {
        'base_fare': round(base_fare, 2),
        'distance_fare': round(distance_fare, 2),
        'time_fare': round(time_fare, 2),
        'subtotal': round(subtotal, 2),
        'surge_multiplier': surge_multiplier,
        'total_fare': round(total_fare, 2)
    }
```

---

### 4.3 Surge Pricing Algorithm

**Dynamic pricing based on supply and demand.**

```python
import time

def calculate_surge_multiplier(pickup_lat: float, pickup_lng: float,
                               radius_km: int = 2) -> float:
    """
    Calculate surge multiplier based on supply/demand ratio
    Returns: Surge multiplier (1.0 = no surge, 2.0 = 2x pricing)
    """

    # 1. Count demand (active ride requests in area)
    demand_count = redis_client.zcount(
        f'ride_requests:{get_geohash(pickup_lat, pickup_lng, precision=5)}',
        '-inf', '+inf'
    )

    # 2. Count supply (available drivers in area)
    available_drivers = redis_client.georadius(
        name='drivers:online',
        longitude=pickup_lng,
        latitude=pickup_lat,
        radius=radius_km,
        unit='km',
        count=50
    )
    supply_count = len(available_drivers)

    # 3. Calculate demand/supply ratio
    if supply_count == 0:
        demand_supply_ratio = float('inf')
    else:
        demand_supply_ratio = demand_count / supply_count

    # 4. Determine surge multiplier
    if demand_supply_ratio < 0.5:
        return 1.0  # No surge
    elif demand_supply_ratio < 1.0:
        return 1.2  # Low surge
    elif demand_supply_ratio < 2.0:
        return 1.5  # Medium surge
    elif demand_supply_ratio < 4.0:
        return 2.0  # High surge
    else:
        return 2.5  # Very high surge (cap at 2.5x)


def get_geohash(lat: float, lng: float, precision: int = 7) -> str:
    """
    Convert lat/lng to geohash for grid-based demand tracking
    """
    import geohash2
    return geohash2.encode(lat, lng, precision=precision)
```

---

## 5. Real-Time Communication

### WebSocket Architecture

**Bidirectional real-time updates between app and server.**

```python
from flask import Flask
from flask_socketio import SocketIO, emit, join_room, leave_room
import redis

app = Flask(__name__)
socketio = SocketIO(app, cors_allowed_origins="*")
redis_client = redis.Redis(host='localhost', port=6379)

# Driver connects and goes online
@socketio.on('driver_online')
def handle_driver_online(data):
    driver_id = data['driver_id']
    lat = data['lat']
    lng = data['lng']
    vehicle_type = data['vehicle_type']

    # Add to room for this driver
    join_room(f'driver_{driver_id}')

    # Update Redis GEO
    redis_client.geoadd('drivers:online', lng, lat, driver_id)

    # Update driver status
    redis_client.hset(f'driver:{driver_id}', 'status', 'online')
    redis_client.hset(f'driver:{driver_id}', 'vehicle_type', vehicle_type)

    emit('status_updated', {'status': 'online'})


# Driver updates location (every 5 seconds)
@socketio.on('location_update')
def handle_location_update(data):
    driver_id = data['driver_id']
    lat = data['lat']
    lng = data['lng']

    # Update location in Redis
    redis_client.geoadd('drivers:online', lng, lat, driver_id)
    redis_client.hset(f'driver:{driver_id}', 'last_update', int(time.time()))

    # If driver is on a trip, broadcast location to rider
    trip_id = redis_client.hget(f'driver:{driver_id}:trip', 'trip_id')
    if trip_id:
        trip_id = trip_id.decode('utf-8')
        rider_id = redis_client.hget(f'trip:{trip_id}', 'rider_id').decode('utf-8')

        # Send location update to rider
        socketio.emit('driver_location', {
            'lat': lat,
            'lng': lng,
            'trip_id': trip_id
        }, room=f'rider_{rider_id}')


# Rider requests ride
@socketio.on('request_ride')
def handle_ride_request(data):
    rider_id = data['rider_id']
    pickup_lat = data['pickup_lat']
    pickup_lng = data['pickup_lng']
    dropoff_lat = data['dropoff_lat']
    dropoff_lng = data['dropoff_lng']
    ride_type = data['ride_type']

    # Join rider room
    join_room(f'rider_{rider_id}')

    # Find nearby drivers
    nearby_drivers = find_nearby_drivers(pickup_lat, pickup_lng, ride_type)

    if not nearby_drivers:
        emit('no_drivers_available', {'message': 'No drivers available nearby'})
        return

    # Create trip in database
    trip_id = create_trip(rider_id, pickup_lat, pickup_lng,
                         dropoff_lat, dropoff_lng, ride_type)

    # Send ride request to nearest drivers (top 3)
    for driver in nearby_drivers[:3]:
        driver_id = driver['driver_id']

        # Send to driver
        socketio.emit('ride_request', {
            'trip_id': trip_id,
            'rider_name': 'John Doe',
            'pickup_lat': pickup_lat,
            'pickup_lng': pickup_lng,
            'dropoff_lat': dropoff_lat,
            'dropoff_lng': dropoff_lng,
            'estimated_fare': calculate_fare(driver['distance_km'], 10, ride_type)
        }, room=f'driver_{driver_id}')

    emit('searching_driver', {'trip_id': trip_id})


# Driver accepts ride
@socketio.on('accept_ride')
def handle_accept_ride(data):
    driver_id = data['driver_id']
    trip_id = data['trip_id']

    # Try to assign driver (atomic operation)
    assigned = assign_driver_to_trip(trip_id, driver_id)

    if assigned:
        # Get trip details
        trip = db.get_trip(trip_id)
        rider_id = trip['rider_id']

        # Notify rider
        socketio.emit('driver_assigned', {
            'trip_id': trip_id,
            'driver_name': 'Jane Smith',
            'driver_photo': 'https://...',
            'vehicle_model': 'Toyota Camry',
            'vehicle_plate': 'ABC-123',
            'driver_rating': 4.8,
            'driver_lat': data['lat'],
            'driver_lng': data['lng'],
            'eta_minutes': 5
        }, room=f'rider_{rider_id}')

        # Confirm to driver
        emit('ride_accepted', {'trip_id': trip_id, 'status': 'accepted'})

        # Cancel requests to other drivers
        cancel_other_ride_requests(trip_id, driver_id)
    else:
        # Driver was already assigned or trip taken
        emit('ride_already_taken', {'message': 'This ride has been taken'})


# Driver arrives at pickup
@socketio.on('driver_arrived')
def handle_driver_arrived(data):
    trip_id = data['trip_id']

    # Update trip status
    db.update_trip_status(trip_id, 'arrived')

    # Notify rider
    trip = db.get_trip(trip_id)
    socketio.emit('driver_arrived', {
        'trip_id': trip_id,
        'message': 'Your driver has arrived'
    }, room=f'rider_{trip["rider_id"]}')


# Trip started
@socketio.on('start_trip')
def handle_start_trip(data):
    trip_id = data['trip_id']

    # Update trip status and timestamp
    db.update_trip_status(trip_id, 'started')
    db.execute("UPDATE trips SET started_at = NOW() WHERE id = ?", [trip_id])

    # Notify rider
    trip = db.get_trip(trip_id)
    socketio.emit('trip_started', {
        'trip_id': trip_id,
        'message': 'Your trip has started'
    }, room=f'rider_{trip["rider_id"]}')


# Trip completed
@socketio.on('complete_trip')
def handle_complete_trip(data):
    trip_id = data['trip_id']
    final_lat = data['lat']
    final_lng = data['lng']

    # Get trip details
    trip = db.get_trip(trip_id)

    # Calculate actual distance and duration
    distance_km = haversine_distance(
        trip['pickup_lat'], trip['pickup_lng'],
        final_lat, final_lng
    )

    duration_minutes = (time.time() - trip['started_at'].timestamp()) / 60

    # Calculate final fare
    fare_details = calculate_fare(
        distance_km,
        duration_minutes,
        trip['ride_type'],
        trip['surge_multiplier']
    )

    # Update trip
    db.execute("""
        UPDATE trips
        SET status = 'completed',
            completed_at = NOW(),
            actual_fare = ?,
            distance_km = ?,
            duration_minutes = ?
        WHERE id = ?
    """, [fare_details['total_fare'], distance_km, duration_minutes, trip_id])

    # Process payment
    process_payment(trip_id, trip['rider_id'], fare_details['total_fare'])

    # Remove driver from online pool temporarily
    redis_client.zrem('drivers:online', trip['driver_id'])

    # Notify both rider and driver
    socketio.emit('trip_completed', {
        'trip_id': trip_id,
        'fare': fare_details['total_fare'],
        'distance_km': distance_km,
        'duration_minutes': int(duration_minutes)
    }, room=f'rider_{trip["rider_id"]}')

    socketio.emit('trip_completed', {
        'trip_id': trip_id,
        'earnings': fare_details['total_fare'] * 0.75  # Driver gets 75%
    }, room=f'driver_{trip["driver_id"]}')


if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=5000)
```

---

## 6. Scaling Strategies

### 6.1 Database Sharding

**Shard trips by geographic region (city).**

```python
def get_shard_for_location(lat: float, lng: float) -> str:
    """
    Determine database shard based on geohash
    """
    geohash = get_geohash(lat, lng, precision=4)  # City-level precision

    # Map geohash prefixes to shards
    shard_mapping = {
        '9q8y': 'shard_sf',      # San Francisco
        'dr5r': 'shard_ny',      # New York
        'w21z': 'shard_paris',   # Paris
        # ... more cities
    }

    return shard_mapping.get(geohash[:4], 'shard_default')


# Example: Route trip write to correct shard
shard = get_shard_for_location(trip['pickup_lat'], trip['pickup_lng'])
db_connection = get_database_connection(shard)
db_connection.execute("INSERT INTO trips (...) VALUES (...)", trip_data)
```

---

### 6.2 WebSocket Scaling

**Use Redis Pub/Sub for multi-server WebSocket synchronization.**

```python
import redis
from flask_socketio import SocketIO

# Redis pub/sub for cross-server communication
redis_client = redis.Redis(host='localhost', port=6379)
socketio = SocketIO(app, message_queue='redis://localhost:6379')

# When location updates happen on Server A, broadcast to all servers
@socketio.on('location_update')
def handle_location_update(data):
    driver_id = data['driver_id']

    # Publish to Redis (all WebSocket servers subscribed)
    redis_client.publish('location_updates', json.dumps({
        'driver_id': driver_id,
        'lat': data['lat'],
        'lng': data['lng']
    }))

# All servers receive and broadcast to their connected clients
pubsub = redis_client.pubsub()
pubsub.subscribe('location_updates')

for message in pubsub.listen():
    if message['type'] == 'message':
        data = json.loads(message['data'])
        socketio.emit('driver_location', data, room=f'rider_{rider_id}')
```

---

### 6.3 Caching Strategy

**Redis caching for hot data.**

```python
# Cache driver locations (expires after 10 seconds if not updated)
redis_client.setex(
    f'driver:{driver_id}:location',
    10,  # TTL
    json.dumps({'lat': lat, 'lng': lng, 'timestamp': time.time()})
)

# Cache user profiles (1 hour TTL)
redis_client.setex(
    f'user:{user_id}:profile',
    3600,
    json.dumps(user_data)
)

# Cache fare estimates (5 minutes TTL)
cache_key = f'fare:{pickup_hash}:{dropoff_hash}:{ride_type}'
cached_fare = redis_client.get(cache_key)
if cached_fare:
    return json.loads(cached_fare)
else:
    fare = calculate_fare(...)
    redis_client.setex(cache_key, 300, json.dumps(fare))
    return fare
```

---

## 7. Security Considerations

| Issue | Solution |
|-------|----------|
| **Driver location privacy** | Only share precise location during active trip, fuzzy location otherwise |
| **Payment fraud** | Use tokenized payment (Stripe), PCI-DSS compliance, 3D Secure |
| **GPS spoofing** | Validate location changes (speed limits), cross-verify with cell towers |
| **Account takeover** | Multi-factor authentication, device fingerprinting |
| **Fake drivers** | Background checks, license verification, real-time photo verification |
| **Rider safety** | Share trip details with emergency contacts, in-app SOS button, driver ratings |
| **API abuse** | Rate limiting (100 requests/min per user), API key rotation |
| **Data encryption** | TLS 1.3 for transit, AES-256 for data at rest |
| **SQL injection** | Parameterized queries, prepared statements |
| **Session hijacking** | Short-lived JWT tokens (15 min), refresh tokens with rotation |

---

## 8. API Design

### REST Endpoints

```python
# Rider APIs
POST   /api/v1/rides/request          # Request a ride
GET    /api/v1/rides/{trip_id}        # Get ride details
POST   /api/v1/rides/{trip_id}/cancel # Cancel ride
POST   /api/v1/rides/{trip_id}/rate   # Rate driver
GET    /api/v1/rides/history           # Get ride history

# Driver APIs
POST   /api/v1/drivers/online         # Go online
POST   /api/v1/drivers/offline        # Go offline
POST   /api/v1/rides/{trip_id}/accept # Accept ride
POST   /api/v1/rides/{trip_id}/start  # Start trip
POST   /api/v1/rides/{trip_id}/complete # Complete trip
GET    /api/v1/drivers/earnings       # Get earnings

# Pricing APIs
POST   /api/v1/pricing/estimate       # Get fare estimate
GET    /api/v1/pricing/surge          # Get current surge multiplier

# Payment APIs
POST   /api/v1/payments/process       # Process payment
GET    /api/v1/payments/{payment_id}  # Get payment status
```

**Example: Request Ride**

```http
POST /api/v1/rides/request
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "pickup_lat": 37.7749,
  "pickup_lng": -122.4194,
  "pickup_address": "123 Market St, San Francisco",
  "dropoff_lat": 37.7849,
  "dropoff_lng": -122.4094,
  "dropoff_address": "456 Mission St, San Francisco",
  "ride_type": "economy",
  "payment_method": "card"
}
```

**Response:**

```json
{
  "trip_id": 12345,
  "status": "searching",
  "estimated_fare": {
    "base_fare": 2.50,
    "distance_fare": 3.00,
    "time_fare": 1.50,
    "subtotal": 7.00,
    "surge_multiplier": 1.5,
    "total_fare": 10.50
  },
  "estimated_arrival": "5 min"
}
```

---

## 9. Capacity Estimation & Performance

### Traffic Estimation

**Assumptions:**
- 100 million total users (80% riders, 20% drivers)
- 10 million daily active riders
- 5% of active riders request a ride simultaneously during peak hours
- Average ride duration: 20 minutes

**Calculations:**

#### Peak Concurrent Rides
```
Peak concurrent requests = 10M * 5% = 500,000 concurrent ride requests
Drivers needed = 500,000 (1:1 ratio during peak)
```

#### Requests Per Second
```
Location updates (drivers send every 5 seconds):
- Active drivers: 500,000
- Updates/sec = 500,000 / 5 = 100,000 location updates/sec

WebSocket connections:
- Active riders: 500,000
- Active drivers: 500,000
- Total: 1,000,000 concurrent WebSocket connections
```

#### Database Load
```
Trip writes:
- New trips/hour (peak) = 500,000 trips (each lasts ~20 min)
- Trips/sec = 500,000 / 3600 = ~140 writes/sec

Trip reads:
- Status checks, updates: ~10x writes = 1,400 reads/sec

Location writes:
- 100,000 updates/sec to Redis GEO
```

---

### Storage Estimation

#### Trips Table
```
Average trip record size: 500 bytes
Daily trips: 10M * 1 trip/user = 10M trips
Daily storage: 10M * 500 bytes = 5 GB/day
Annual storage: 5 GB * 365 = 1.825 TB/year
5-year storage: ~9 TB
```

#### Location Tracking History
```
Location update size: 50 bytes (lat, lng, timestamp, speed)
Updates per trip: 20 min / 5 sec = 240 updates
Storage per trip: 240 * 50 bytes = 12 KB
Daily storage: 10M trips * 12 KB = 120 GB/day
Annual storage: 120 GB * 365 = 43.8 TB/year (use Cassandra for time-series)
```

#### Redis (Hot Data)
```
Active driver locations: 500,000 * 100 bytes = 50 MB
Active trips: 500,000 * 500 bytes = 250 MB
User sessions: 1M * 200 bytes = 200 MB
Total Redis: ~500 MB (easily fits in memory)
```

---

### Server Estimation

#### API Servers
```
Requests/sec: 1,400 (reads) + 140 (writes) = 1,540 QPS
Assuming 1 server handles 100 QPS:
Servers needed: 1,540 / 100 = 16 servers
With 3x redundancy: 48 API servers
```

#### WebSocket Servers
```
Concurrent connections: 1,000,000
Assuming 1 server handles 10,000 connections:
Servers needed: 1,000,000 / 10,000 = 100 servers
With 2x redundancy: 200 WebSocket servers
```

#### Database Servers
```
PostgreSQL (trips):
- Sharded by city (top 50 cities)
- 50 shards * 3 replicas each = 150 database servers

Redis (cache + locations):
- Cluster with 10 master nodes
- 3 replicas each = 30 Redis servers

Cassandra (history):
- 20 nodes for distributed storage
```

---

### Bandwidth

```
Outgoing (location updates to riders):
- 1M active riders * 1 update/5sec * 100 bytes = 20 MB/sec = 160 Mbps

Incoming (driver location updates):
- 500K drivers * 1 update/5sec * 100 bytes = 10 MB/sec = 80 Mbps

Total bandwidth: ~250 Mbps (easily handled by modern infrastructure)
```

---

## 10. Interview Questions & Answers

### Q1: How do you prevent double-booking of drivers?

**Answer:** Use Redis distributed locks with atomic operations.

```python
# Try to acquire lock on driver
lock_acquired = redis_client.set(
    f'driver:{driver_id}:assigned',
    trip_id,
    nx=True,  # Only set if key doesn't exist (atomic)
    ex=30     # Expire after 30 seconds
)

if lock_acquired:
    # Driver is now locked to this trip
    assign_driver_to_trip(trip_id, driver_id)
else:
    # Driver already assigned, try next driver
    try_next_driver()
```

The `NX` flag ensures only one trip can acquire the lock atomically, preventing race conditions.

---

### Q2: How do you handle driver location updates at scale (100K updates/sec)?

**Answer:** Use Redis GEO with batching and async processing.

**Optimizations:**
1. **Client-side batching:** Driver app batches 3 location points and sends every 15 seconds instead of every 5 seconds (reduces load by 3x)
2. **Redis GEO:** O(log N) insertion performance, handles millions of points
3. **Async processing:** WebSocket server writes to Redis asynchronously, doesn't block response
4. **Sharding:** Shard Redis by geographic region (SF, NY, LA clusters)

```python
# Async location update
@socketio.on('location_update')
def handle_location_update(data):
    # Non-blocking: Queue for async processing
    location_queue.put(data)
    emit('ack')  # Immediate acknowledgment

# Background worker processes queue
def process_location_updates():
    while True:
        batch = []
        for _ in range(100):  # Process 100 at a time
            if not location_queue.empty():
                batch.append(location_queue.get())

        if batch:
            # Batch write to Redis (pipeline)
            pipe = redis_client.pipeline()
            for update in batch:
                pipe.geoadd('drivers:online',
                           update['lng'], update['lat'], update['driver_id'])
            pipe.execute()
```

---

### Q3: How does surge pricing work in real-time?

**Answer:** Calculate demand/supply ratio using geohash-based grid system.

**Algorithm:**
1. Divide city into grid using geohash (precision 5 = ~5km grid)
2. Track ride requests per grid cell
3. Track available drivers per grid cell
4. Calculate ratio every 1 minute
5. Apply surge multiplier based on ratio

```python
# Grid-based demand tracking
def track_ride_request(lat, lng):
    grid_id = get_geohash(lat, lng, precision=5)
    redis_client.zincrby(f'demand:{grid_id}', 1, int(time.time()))

    # Expire old requests (older than 10 minutes)
    redis_client.zremrangebyscore(
        f'demand:{grid_id}',
        '-inf',
        int(time.time()) - 600
    )

# Calculate surge every minute
def update_surge_pricing():
    for grid_id in active_grids:
        demand = redis_client.zcount(f'demand:{grid_id}', '-inf', '+inf')
        supply = count_drivers_in_grid(grid_id)

        ratio = demand / max(supply, 1)
        surge = calculate_surge_from_ratio(ratio)

        redis_client.setex(f'surge:{grid_id}', 60, surge)
```

---

### Q4: How do you handle network failures during a trip?

**Answer:** Use offline-first approach with local state and reconciliation.

**Driver App:**
```python
# Store trip state locally
local_db.save({
    'trip_id': trip_id,
    'status': 'started',
    'route': [(lat1, lng1, timestamp1), (lat2, lng2, timestamp2), ...],
    'sync_status': 'pending'
})

# Periodic sync when online
if is_online():
    sync_trip_data()
else:
    # Continue tracking locally
    continue_trip_offline()

def sync_trip_data():
    pending_trips = local_db.get_pending_syncs()
    for trip in pending_trips:
        try:
            api.sync_trip(trip)
            local_db.mark_synced(trip.id)
        except NetworkError:
            break  # Will retry later
```

**Backend:**
```python
# Accept late updates with idempotency
@app.route('/api/v1/trips/{trip_id}/sync', methods=['POST'])
def sync_trip(trip_id):
    updates = request.json['updates']

    for update in updates:
        # Idempotent: Use timestamp to avoid duplicates
        existing = db.query(
            "SELECT * FROM location_tracking WHERE trip_id = ? AND timestamp = ?",
            [trip_id, update['timestamp']]
        )

        if not existing:
            db.insert_location(trip_id, update)
```

---

### Q5: How do you handle driver assignment timeout?

**Answer:** Multi-level timeout with automatic fallback.

```python
def request_ride_with_timeout(trip_id, nearby_drivers):
    # Level 1: Send to top 3 drivers, wait 30 seconds
    send_to_drivers(nearby_drivers[:3], timeout=30)

    time.sleep(30)

    if not is_trip_assigned(trip_id):
        # Level 2: Send to next 5 drivers, wait 20 seconds
        send_to_drivers(nearby_drivers[3:8], timeout=20)

        time.sleep(20)

        if not is_trip_assigned(trip_id):
            # Level 3: Expand radius, send to any available driver
            expanded_drivers = find_nearby_drivers(radius_km=10)
            send_to_drivers(expanded_drivers, timeout=30)

            time.sleep(30)

            if not is_trip_assigned(trip_id):
                # No driver found
                notify_rider_no_driver(trip_id)
                cancel_trip(trip_id)
```

**Optimization:** Use priority queue to try drivers in order of distance.

---

### Q6: How would you design the rating system?

**Answer:** Use eventual consistency with Redis cache and PostgreSQL.

**Rating Storage:**
```sql
CREATE TABLE ratings (
    id BIGSERIAL PRIMARY KEY,
    trip_id BIGINT REFERENCES trips(id),
    rater_id BIGINT REFERENCES users(id),
    rated_id BIGINT REFERENCES users(id),
    rating SMALLINT NOT NULL, -- 1-5
    comment TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_rated_id (rated_id)
);
```

**Rating Calculation (Cached):**
```python
def get_user_rating(user_id):
    # Try cache first
    cached_rating = redis_client.get(f'rating:{user_id}')
    if cached_rating:
        return float(cached_rating)

    # Calculate from database
    result = db.query("""
        SELECT AVG(rating) as avg_rating, COUNT(*) as total_ratings
        FROM ratings
        WHERE rated_id = ?
    """, [user_id])

    avg_rating = result['avg_rating'] or 5.0

    # Cache for 1 hour
    redis_client.setex(f'rating:{user_id}', 3600, avg_rating)

    return avg_rating

def update_rating(trip_id, rater_id, rated_id, rating):
    # Insert rating
    db.execute("""
        INSERT INTO ratings (trip_id, rater_id, rated_id, rating)
        VALUES (?, ?, ?, ?)
    """, [trip_id, rater_id, rated_id, rating])

    # Invalidate cache (will recalculate on next request)
    redis_client.delete(f'rating:{rated_id}')

    # Update user's average rating (denormalized)
    new_avg = get_user_rating(rated_id)
    db.execute("UPDATE users SET rating = ? WHERE id = ?", [new_avg, rated_id])
```

---

### Q7: How would you implement ride cancellation and refunds?

**Answer:** Use state machine with compensation logic.

**Trip States:**
```
requested → accepted → arriving → started → completed
     ↓          ↓          ↓          ↓
  cancelled  cancelled  cancelled  cancelled
```

**Cancellation Policy:**
```python
def cancel_trip(trip_id, cancelled_by):
    trip = db.get_trip(trip_id)

    # Check if cancellation is allowed
    if trip['status'] == 'completed':
        raise Exception("Cannot cancel completed trip")

    # Determine cancellation fee based on status
    cancellation_fee = 0

    if trip['status'] == 'requested':
        # Free cancellation
        cancellation_fee = 0
    elif trip['status'] == 'accepted' or trip['status'] == 'arriving':
        # Check time since acceptance
        time_since_accepted = (now() - trip['accepted_at']).seconds

        if cancelled_by == 'rider':
            if time_since_accepted < 120:  # 2 minutes
                cancellation_fee = 0
            else:
                cancellation_fee = 5.00  # Flat fee
        else:  # cancelled by driver
            # Penalize driver, compensate rider
            penalize_driver(trip['driver_id'], amount=10.00)
            compensate_rider(trip['rider_id'], amount=5.00)
    elif trip['status'] == 'started':
        # Trip started, calculate partial fare
        distance_km = calculate_distance_from_tracking(trip_id)
        duration_min = (now() - trip['started_at']).seconds / 60

        partial_fare = calculate_fare(distance_km, duration_min, trip['ride_type'])
        cancellation_fee = partial_fare['total_fare']

    # Process cancellation fee/refund
    if cancellation_fee > 0:
        charge_cancellation_fee(trip['rider_id'], cancellation_fee)

    # Update trip status
    db.execute("""
        UPDATE trips
        SET status = 'cancelled',
            cancelled_by = ?,
            cancelled_at = NOW(),
            cancellation_fee = ?
        WHERE id = ?
    """, [cancelled_by, cancellation_fee, trip_id])

    # Release driver
    if trip['driver_id']:
        release_driver(trip['driver_id'])

    # Notify both parties
    notify_cancellation(trip_id, cancelled_by, cancellation_fee)
```

---

### Q8: How do you handle cross-region rides (e.g., ride starts in City A, ends in City B)?

**Answer:** Use distributed transaction with Saga pattern.

**Problem:** Trip data sharded by pickup location (City A), but dropoff is in City B.

**Solution:**
```python
def handle_cross_region_trip(trip_id):
    trip = db.get_trip(trip_id)

    pickup_shard = get_shard_for_location(trip['pickup_lat'], trip['pickup_lng'])
    dropoff_shard = get_shard_for_location(trip['dropoff_lat'], trip['dropoff_lng'])

    if pickup_shard == dropoff_shard:
        # Same region, simple update
        update_trip_on_shard(pickup_shard, trip_id, 'completed')
    else:
        # Cross-region: Use Saga pattern
        # 1. Complete trip on pickup shard
        complete_trip_on_shard(pickup_shard, trip_id)

        # 2. Replicate to dropoff shard for analytics
        try:
            replicate_trip_to_shard(dropoff_shard, trip)
        except Exception as e:
            # Queue for async replication
            replication_queue.enqueue({
                'trip_id': trip_id,
                'from_shard': pickup_shard,
                'to_shard': dropoff_shard
            })

        # 3. Store in global index for cross-region queries
        cassandra.insert('trip_index', {
            'trip_id': trip_id,
            'pickup_region': pickup_shard,
            'dropoff_region': dropoff_shard,
            'completed_at': now()
        })
```

---

### Q9: How would you implement driver heat maps?

**Answer:** Aggregate location data using geohash-based grid.

```python
def generate_driver_heatmap(city_bounds, precision=6):
    """
    Generate heatmap showing driver density
    Returns: {geohash: driver_count}
    """
    heatmap = {}

    # Query all online drivers
    all_drivers = redis_client.georadius(
        'drivers:online',
        longitude=city_bounds['center_lng'],
        latitude=city_bounds['center_lat'],
        radius=50,  # 50km radius
        unit='km',
        withcoord=True
    )

    # Aggregate by geohash grid
    for driver_data in all_drivers:
        driver_id = driver_data[0]
        coords = driver_data[1]  # [lng, lat]

        # Convert to geohash
        grid_hash = get_geohash(coords[1], coords[0], precision=precision)

        # Increment count for this grid cell
        heatmap[grid_hash] = heatmap.get(grid_hash, 0) + 1

    return heatmap

# Example response
{
    "9q8yy9": 45,  # 45 drivers in this grid
    "9q8yy8": 32,
    "9q8yy7": 28,
    ...
}
```

**Visualization:** Frontend converts geohash to polygons and colors by density.

---

### Q10: How do you optimize matching when there are thousands of drivers nearby?

**Answer:** Use spatial indexing with filtering and caching.

**Optimizations:**

1. **Limit radius initially (500m)**, expand if no match
2. **Filter by vehicle type before distance calculation**
3. **Cache recent searches** (same pickup location)
4. **Pre-compute driver clusters** during low traffic

```python
def optimized_driver_matching(pickup_lat, pickup_lng, ride_type):
    # 1. Try cache first (1 minute TTL)
    cache_key = f'drivers:{get_geohash(pickup_lat, pickup_lng, 7)}:{ride_type}'
    cached_drivers = redis_client.get(cache_key)

    if cached_drivers:
        return json.loads(cached_drivers)

    # 2. Query with expanding radius
    for radius_km in [0.5, 1, 2, 5]:
        drivers = redis_client.georadius(
            f'drivers:online:{ride_type}',  # Separate GEO index per vehicle type
            longitude=pickup_lng,
            latitude=pickup_lat,
            radius=radius_km,
            unit='km',
            count=10,  # Limit to top 10
            sort='ASC',
            withdist=True
        )

        if drivers:
            # Filter truly available
            available = [d for d in drivers if is_driver_available(d[0])]

            if available:
                # Cache result
                redis_client.setex(cache_key, 60, json.dumps(available))
                return available

    return []  # No drivers found
```

**Further optimization:** Maintain separate Redis GEO indices per vehicle type:
- `drivers:online:economy`
- `drivers:online:premium`
- `drivers:online:suv`

This reduces search space by 3x.

---

## 11. Trade-offs & Alternatives

| Decision | Chosen Approach | Alternative | Trade-off |
|----------|----------------|-------------|-----------|
| **Location storage** | Redis GEO | PostGIS (PostgreSQL extension) | Redis: Faster reads (in-memory), but no ACID. PostGIS: ACID guarantees but slower |
| **Real-time communication** | WebSocket | Server-Sent Events (SSE) | WebSocket: Bidirectional, more complex. SSE: Simpler, but unidirectional |
| **Driver matching** | Geohashing + Redis GEO | QuadTree in-memory | Redis GEO: Persistent, distributed. QuadTree: Faster but requires custom implementation |
| **Trip storage** | PostgreSQL (sharded) | MongoDB | PostgreSQL: ACID, better for financial data. MongoDB: Easier horizontal scaling |
| **Location history** | Cassandra | TimescaleDB | Cassandra: Better write throughput for time-series. TimescaleDB: SQL compatibility |
| **Payment processing** | External service (Stripe) | In-house | Stripe: PCI-DSS compliant, less liability. In-house: More control, higher cost |
| **Surge pricing** | Grid-based (geohash) | Machine Learning model | Grid: Simpler, predictable. ML: More accurate, complex |

---

## 12. Miscellaneous

### 12.1 Handling Peak Traffic (New Year's Eve)

**Problem:** 10x normal traffic (5 million concurrent rides)

**Solutions:**

1. **Auto-scaling:** Pre-scale servers 2 hours before peak
```yaml
# Kubernetes HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ride-matching-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ride-matching
  minReplicas: 50
  maxReplicas: 500
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

2. **Database read replicas:** Scale PostgreSQL read replicas from 3 to 15
3. **Redis cluster scaling:** Add temporary Redis nodes
4. **Queue overflow handling:** If driver matching queue grows, return "high demand" message to riders

---

### 12.2 Fraud Detection

**Common fraud patterns:**

1. **Fake trips:** Driver and rider collude to generate fake trips
2. **GPS spoofing:** Driver fakes location to appear closer
3. **Payment fraud:** Stolen credit cards

**Detection:**

```python
def detect_fraudulent_trip(trip_id):
    trip = db.get_trip(trip_id)
    fraud_score = 0

    # Check 1: Trip route vs GPS tracking mismatch
    expected_distance = haversine_distance(
        trip['pickup_lat'], trip['pickup_lng'],
        trip['dropoff_lat'], trip['dropoff_lng']
    )

    actual_distance = sum_gps_tracking_distance(trip_id)

    if abs(expected_distance - actual_distance) > expected_distance * 0.3:
        fraud_score += 50  # 30% deviation

    # Check 2: Driver and rider frequently ride together
    shared_trips = db.query("""
        SELECT COUNT(*) FROM trips
        WHERE driver_id = ? AND rider_id = ?
        AND created_at > NOW() - INTERVAL '30 days'
    """, [trip['driver_id'], trip['rider_id']])

    if shared_trips > 10:
        fraud_score += 30

    # Check 3: Impossible speed (GPS spoofing)
    max_speed = get_max_speed_from_tracking(trip_id)
    if max_speed > 200:  # 200 km/h unrealistic for city
        fraud_score += 40

    # Check 4: Payment card flagged
    if is_card_flagged(trip['payment_method']):
        fraud_score += 50

    # Flag for review if score > 70
    if fraud_score > 70:
        flag_trip_for_review(trip_id, fraud_score)
        suspend_payout(trip_id)
```

---

### 12.3 Multi-Region Deployment

**Architecture:**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   US-West       │     │   US-East       │     │   EU-Central    │
│  (San Francisco)│     │  (New York)     │     │   (Frankfurt)   │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│ API Gateway     │     │ API Gateway     │     │ API Gateway     │
│ Location Service│     │ Location Service│     │ Location Service│
│ Ride Matching   │     │ Ride Matching   │     │ Ride Matching   │
│ PostgreSQL      │     │ PostgreSQL      │     │ PostgreSQL      │
│ Redis           │     │ Redis           │     │ Redis           │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                       │                       │
         └───────────────────────┴───────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Global Services   │
                    │  - User Service    │
                    │  - Payment Service │
                    │  - Analytics       │
                    │  (Replicated)      │
                    └────────────────────┘
```

**Data residency:** Store trip data in region where trip occurred (GDPR compliance)

**Cross-region sync:** Cassandra for global analytics, local PostgreSQL for operational data

---

## 13. Summary

✅ **Geospatial matching** - Redis GEO for O(log N) driver search within radius
✅ **Real-time tracking** - WebSocket for bidirectional location updates
✅ **Scalable architecture** - Sharded databases, Redis cache, Cassandra for history
✅ **Low latency** - < 2 seconds driver matching, < 1 second location updates
✅ **High availability** - Multi-region deployment, auto-scaling, circuit breakers
✅ **Surge pricing** - Grid-based demand/supply calculation
✅ **Security** - Payment tokenization, GPS validation, fraud detection
✅ **Fault tolerance** - Offline-first apps, distributed locks, Saga pattern
✅ **Performance** - 100K location updates/sec, 1M concurrent WebSocket connections

**Key Design Decisions:**
- Redis GEO for geospatial indexing (fast, persistent, distributed)
- PostgreSQL for transactional data with geographic sharding
- Cassandra for time-series location history (write-optimized)
- WebSocket with Redis Pub/Sub for multi-server synchronization
- Distributed locks (Redis) to prevent driver double-booking
- Client-side batching and caching to reduce server load
- Saga pattern for distributed transactions (cross-region trips)

---

**Related Concepts:**
- [Geospatial Systems](concepts/geospatial-systems.md) - Geohashing, QuadTree, R-Tree
- [WebSockets & Real-Time](concepts/websockets-realtime.md) - WebSocket scaling, Redis Pub/Sub
- [Notification Systems](concepts/notification-systems.md) - Push notifications for ride events
- [Database Sharding](concepts/database-sharding.md) - Geographic sharding strategy
- [Circuit Breaker & Resilience](concepts/circuit-breaker-resilience.md) - Handling service failures
- [Distributed Transactions](concepts/distributed-transactions.md) - Saga pattern for cross-region trips
