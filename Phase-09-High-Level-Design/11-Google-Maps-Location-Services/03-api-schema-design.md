# Google Maps / Location Services - API & Schema Design

## API Design Philosophy

Location services APIs prioritize:

1. **Low Latency**: Sub-200ms response times for search
2. **Geographic Accuracy**: Precise coordinates and routing
3. **Real-time Data**: Fresh traffic and location updates
4. **Mobile Optimization**: Efficient payload sizes
5. **Caching-Friendly**: Cacheable responses where possible

---

## Base URL Structure

```
REST API:     https://api.maps.example.com/v1
WebSocket:    wss://ws.maps.example.com/v1
CDN:          https://cdn.maps.example.com
```

---

## API Versioning Strategy

We use URL path versioning (`/v1/`, `/v2/`) because:
- Easy to understand and implement
- Clear in logs and documentation
- Allows running multiple versions simultaneously

**Backward Compatibility Rules:**

Non-breaking changes (no version bump):
- Adding new optional fields
- Adding new endpoints
- Adding new error codes

Breaking changes (require new version):
- Removing fields
- Changing field types
- Changing endpoint paths

**Deprecation Policy:**
1. Announce deprecation 6 months in advance
2. Return Deprecation header
3. Maintain old version for 12 months after new version release

---

## Rate Limiting Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**Rate Limits:**

| Tier | Requests/minute |
|------|-----------------|
| Free | 100 |
| Pro | 1000 |
| Enterprise | 10000 |

---

## Error Model

All error responses follow this standard envelope structure:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "field": "field_name",
      "reason": "Specific reason"
    },
    "request_id": "req_123456"
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | INVALID_COORDINATES | Latitude/longitude out of range |
| 400 | NO_ROUTE_FOUND | No route exists between points |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Location or place not found |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

---

## Idempotency Implementation

**Idempotency-Key Header:**

Clients must include `Idempotency-Key` header for all write operations:
```http
Idempotency-Key: <uuid-v4>
```

**Deduplication Storage:**

Idempotency keys are stored in Redis with 24-hour TTL:
```
Key: idempotency:{key}
Value: Serialized response
TTL: 24 hours
```

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/routes | Yes | Idempotency-Key header |
| POST /v1/locations | Yes | Idempotency-Key header |
| PUT /v1/locations/{id} | Yes | Idempotency-Key or version-based |

---

## Core API Endpoints

### 1. Location Search

**Search Places**

`GET /v1/places/search`

**Query Parameters:**
- `query` (required): Search query string
- `lat` (optional): Latitude for proximity search
- `lng` (optional): Longitude for proximity search
- `radius` (optional): Search radius in meters (default: 5000)
- `category` (optional): Filter by category (restaurant, gas_station, etc.)
- `limit` (optional): Max results (default: 20, max: 50)
- `offset` (optional): Pagination offset

**Request Example:**
```http
GET /v1/places/search?query=coffee&lat=37.7749&lng=-122.4194&radius=1000&limit=10
```

**Response (200 OK):**
```json
{
  "places": [
    {
      "id": "place_abc123",
      "name": "Blue Bottle Coffee",
      "category": "cafe",
      "address": "315 Linden St, San Francisco, CA 94102",
      "location": {
        "lat": 37.7750,
        "lng": -122.4195
      },
      "distance": 250,
      "rating": 4.5,
      "rating_count": 1234,
      "open_now": true,
      "hours": {
        "monday": "7:00 AM - 6:00 PM",
        "tuesday": "7:00 AM - 6:00 PM"
      }
    }
  ],
  "pagination": {
    "total": 45,
    "limit": 10,
    "offset": 0,
    "has_more": true
  }
}
```

---

**Autocomplete**

`GET /v1/places/autocomplete`

**Query Parameters:**
- `query` (required): Partial search query
- `lat` (optional): Latitude for proximity ranking
- `lng` (optional): Longitude for proximity ranking
- `limit` (optional): Max results (default: 5, max: 10)

**Request Example:**
```http
GET /v1/places/autocomplete?query=blue bot&lat=37.7749&lng=-122.4194
```

**Response (200 OK):**
```json
{
  "suggestions": [
    {
      "id": "place_abc123",
      "name": "Blue Bottle Coffee",
      "address": "315 Linden St, San Francisco, CA",
      "category": "cafe"
    },
    {
      "id": "place_def456",
      "name": "Blue Bottle Coffee - Hayes Valley",
      "address": "315 Hayes St, San Francisco, CA",
      "category": "cafe"
    }
  ]
}
```

---

### 2. Geocoding

**Geocode Address**

`GET /v1/geocode`

**Query Parameters:**
- `address` (required): Address string

**Request Example:**
```http
GET /v1/geocode?address=1600+Amphitheatre+Parkway,+Mountain+View,+CA
```

**Response (200 OK):**
```json
{
  "results": [
    {
      "formatted_address": "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
      "location": {
        "lat": 37.4224764,
        "lng": -122.0842499
      },
      "address_components": [
        {
          "long_name": "1600",
          "short_name": "1600",
          "types": ["street_number"]
        },
        {
          "long_name": "Amphitheatre Parkway",
          "short_name": "Amphitheatre Pkwy",
          "types": ["route"]
        },
        {
          "long_name": "Mountain View",
          "short_name": "Mountain View",
          "types": ["locality", "political"]
        }
      ],
      "place_id": "ChIJ2eUgeAK6j4ARbn5u_wAGqWA"
    }
  ]
}
```

---

**Reverse Geocode**

`GET /v1/geocode/reverse`

**Query Parameters:**
- `lat` (required): Latitude
- `lng` (required): Longitude

**Request Example:**
```http
GET /v1/geocode/reverse?lat=37.4224764&lng=-122.0842499
```

**Response (200 OK):**
```json
{
  "results": [
    {
      "formatted_address": "1600 Amphitheatre Parkway, Mountain View, CA 94043, USA",
      "location": {
        "lat": 37.4224764,
        "lng": -122.0842499
      },
      "address_components": [
        {
          "long_name": "1600",
          "short_name": "1600",
          "types": ["street_number"]
        }
      ],
      "place_id": "ChIJ2eUgeAK6j4ARbn5u_wAGqWA"
    }
  ]
}
```

---

### 3. Routing

**Calculate Route**

`POST /v1/routes`

**Request Body:**
```json
{
  "origin": {
    "lat": 37.7749,
    "lng": -122.4194
  },
  "destination": {
    "lat": 37.7849,
    "lng": -122.4094
  },
  "waypoints": [
    {
      "lat": 37.7799,
      "lng": -122.4144
    }
  ],
  "mode": "driving",
  "alternatives": true,
  "avoid": ["tolls", "highways"],
  "departure_time": "2024-01-20T10:00:00Z",
  "traffic_model": "best_guess"
}
```

**Response (200 OK):**
```json
{
  "routes": [
    {
      "route_id": "route_abc123",
      "distance": {
        "value": 5234,
        "text": "5.2 km"
      },
      "duration": {
        "value": 720,
        "text": "12 min"
      },
      "duration_in_traffic": {
        "value": 900,
        "text": "15 min"
      },
      "polyline": "encoded_polyline_string",
      "bounds": {
        "northeast": {
          "lat": 37.7849,
          "lng": -122.4094
        },
        "southwest": {
          "lat": 37.7749,
          "lng": -122.4194
        }
      },
      "legs": [
        {
          "distance": {
            "value": 5234,
            "text": "5.2 km"
          },
          "duration": {
            "value": 720,
            "text": "12 min"
          },
          "start_address": "San Francisco, CA",
          "end_address": "Oakland, CA",
          "steps": [
            {
              "distance": {
                "value": 100,
                "text": "100 m"
              },
              "duration": {
                "value": 30,
                "text": "30 sec"
              },
              "instruction": "Head north on Market St",
              "polyline": "encoded_polyline",
              "maneuver": "turn-left"
            }
          ]
        }
      ]
    }
  ],
  "status": "OK"
}
```

---

### 4. ETA Calculation

**Get ETA**

`GET /v1/eta`

**Query Parameters:**
- `origin_lat` (required): Origin latitude
- `origin_lng` (required): Origin longitude
- `destination_lat` (required): Destination latitude
- `destination_lng` (required): Destination longitude
- `mode` (optional): Transportation mode (default: driving)
- `departure_time` (optional): ISO 8601 timestamp

**Request Example:**
```http
GET /v1/eta?origin_lat=37.7749&origin_lng=-122.4194&destination_lat=37.7849&destination_lng=-122.4094&mode=driving
```

**Response (200 OK):**
```json
{
  "eta": {
    "duration": {
      "value": 720,
      "text": "12 min"
    },
    "duration_in_traffic": {
      "value": 900,
      "text": "15 min"
    },
    "distance": {
      "value": 5234,
      "text": "5.2 km"
    },
    "traffic_level": "moderate",
    "estimated_arrival": "2024-01-20T10:15:00Z"
  }
}
```

---

### 5. Map Tiles

**Get Map Tile**

`GET /v1/tiles/{z}/{x}/{y}.{format}`

**Path Parameters:**
- `z`: Zoom level (0-18)
- `x`: Tile X coordinate
- `y`: Tile Y coordinate
- `format`: Image format (png, jpg, webp)

**Request Example:**
```http
GET /v1/tiles/10/512/512.png
```

**Response (200 OK):**
- Content-Type: image/png
- Body: Binary image data

**Headers:**
```http
Cache-Control: public, max-age=86400
ETag: "abc123def456"
Last-Modified: Mon, 20 Jan 2024 10:00:00 GMT
```

---

### 6. Traffic Information

**Get Traffic Conditions**

`GET /v1/traffic`

**Query Parameters:**
- `bounds` (required): Bounding box (northeast_lat,northeast_lng,southwest_lat,southwest_lng)
- `zoom` (optional): Zoom level for detail (default: 10)

**Request Example:**
```http
GET /v1/traffic?bounds=37.8,-122.4,37.7,-122.5&zoom=12
```

**Response (200 OK):**
```json
{
  "traffic_conditions": [
    {
      "segment_id": "seg_abc123",
      "polyline": "encoded_polyline",
      "speed": 45,
      "speed_limit": 65,
      "congestion_level": "moderate",
      "incidents": [
        {
          "incident_id": "inc_xyz789",
          "type": "accident",
          "severity": "moderate",
          "description": "Accident on right lane",
          "location": {
            "lat": 37.7750,
            "lng": -122.4195
          }
        }
      ]
    }
  ],
  "updated_at": "2024-01-20T10:00:00Z"
}
```

---

## Database Schema Design

### Database Choices

| Data Type          | Database       | Rationale                                |
| ------------------ | -------------- | ---------------------------------------- |
| POI Data           | PostgreSQL + PostGIS | Geospatial queries, full-text search |
| Route Cache        | Redis          | Fast lookups, TTL support                |
| Map Tiles          | S3 + CDN       | Blob storage, global distribution        |
| Traffic Data       | TimescaleDB    | Time-series data, efficient aggregation  |
| User Locations     | PostgreSQL     | Relational queries, privacy controls      |

---

### Places Table (PostgreSQL with PostGIS)

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE places (
    place_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(50) NOT NULL,
    
    -- Geospatial data (PostGIS)
    location GEOGRAPHY(POINT, 4326) NOT NULL,
    
    -- Address
    formatted_address TEXT,
    address_components JSONB,
    
    -- Business info
    rating DECIMAL(2,1),
    rating_count INTEGER DEFAULT 0,
    price_level SMALLINT,  -- 1-4 (inexpensive to very expensive)
    
    -- Hours
    hours JSONB,  -- { "monday": "9:00 AM - 5:00 PM", ... }
    open_now BOOLEAN,
    
    -- Metadata
    place_types TEXT[],  -- Array of types
    website_url TEXT,
    phone_number VARCHAR(20),
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Geospatial index for proximity searches
CREATE INDEX idx_places_location ON places USING GIST(location);

-- Full-text search index
CREATE INDEX idx_places_name_search ON places USING GIN(to_tsvector('english', name));

-- Category index
CREATE INDEX idx_places_category ON places(category);

-- Composite index for category + location queries
CREATE INDEX idx_places_category_location ON places(category, location);
```

---

### Routes Table

```sql
CREATE TABLE routes (
    route_id VARCHAR(50) PRIMARY KEY,
    
    -- Origin and destination
    origin_lat DECIMAL(10, 8) NOT NULL,
    origin_lng DECIMAL(11, 8) NOT NULL,
    destination_lat DECIMAL(10, 8) NOT NULL,
    destination_lng DECIMAL(11, 8) NOT NULL,
    
    -- Route data
    distance_meters INTEGER NOT NULL,
    duration_seconds INTEGER NOT NULL,
    polyline TEXT NOT NULL,  -- Encoded polyline
    
    -- Route options
    mode VARCHAR(20) NOT NULL,  -- driving, walking, transit
    alternatives BOOLEAN DEFAULT FALSE,
    
    -- Traffic data
    duration_in_traffic_seconds INTEGER,
    traffic_level VARCHAR(20),
    
    -- Caching
    cached_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    
    -- Metadata
    waypoints JSONB,  -- Array of waypoint coordinates
    avoid JSONB,  -- ["tolls", "highways"]
    
    CONSTRAINT valid_mode CHECK (mode IN ('driving', 'walking', 'transit', 'bicycling'))
);

-- Index for route lookups
CREATE INDEX idx_routes_origin_dest ON routes(origin_lat, origin_lng, destination_lat, destination_lng, mode);

-- Index for cache expiration
CREATE INDEX idx_routes_expires_at ON routes(expires_at) WHERE expires_at IS NOT NULL;
```

---

### Traffic Segments Table (TimescaleDB)

```sql
CREATE TABLE traffic_segments (
    segment_id VARCHAR(50) NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    
    -- Traffic data
    speed_kmh INTEGER NOT NULL,
    speed_limit_kmh INTEGER,
    congestion_level VARCHAR(20),  -- free_flow, light, moderate, heavy, severe
    volume INTEGER,  -- Number of vehicles
    
    -- Location
    polyline TEXT,  -- Encoded polyline for segment
    
    PRIMARY KEY (segment_id, timestamp)
);

-- Convert to hypertable for time-series optimization
SELECT create_hypertable('traffic_segments', 'timestamp');

-- Index for segment queries
CREATE INDEX idx_traffic_segments_segment_id ON traffic_segments(segment_id, timestamp DESC);

-- Index for time-range queries
CREATE INDEX idx_traffic_segments_timestamp ON traffic_segments(timestamp DESC);
```

---

### Traffic Incidents Table

```sql
CREATE TABLE traffic_incidents (
    incident_id VARCHAR(50) PRIMARY KEY,
    segment_id VARCHAR(50) NOT NULL,
    
    -- Incident details
    type VARCHAR(50) NOT NULL,  -- accident, road_closure, construction, etc.
    severity VARCHAR(20) NOT NULL,  -- minor, moderate, severe
    description TEXT,
    
    -- Location
    location_lat DECIMAL(10, 8) NOT NULL,
    location_lng DECIMAL(11, 8) NOT NULL,
    
    -- Timing
    reported_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    estimated_end TIMESTAMP WITH TIME ZONE,
    resolved_at TIMESTAMP WITH TIME ZONE,
    
    -- Status
    status VARCHAR(20) DEFAULT 'active',  -- active, resolved
    
    CONSTRAINT valid_status CHECK (status IN ('active', 'resolved'))
);

-- Index for active incidents
CREATE INDEX idx_incidents_status ON traffic_incidents(status, reported_at DESC) WHERE status = 'active';

-- Geospatial index for location queries
CREATE INDEX idx_incidents_location ON traffic_incidents USING GIST(
    ST_MakePoint(location_lng, location_lat)
);
```

---

### User Locations Table

```sql
CREATE TABLE user_locations (
    location_id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,
    
    -- Location
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    accuracy_meters INTEGER,
    
    -- Metadata
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    source VARCHAR(20),  -- gps, network, passive
    
    -- Privacy
    anonymized BOOLEAN DEFAULT FALSE,
    retention_until TIMESTAMP WITH TIME ZONE
);

-- Index for user location history
CREATE INDEX idx_user_locations_user_timestamp ON user_locations(user_id, timestamp DESC);

-- Index for retention cleanup
CREATE INDEX idx_user_locations_retention ON user_locations(retention_until) WHERE retention_until IS NOT NULL;

-- Partition by month for efficient cleanup
-- (Implementation depends on PostgreSQL version)
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐
│      places         │
├─────────────────────┤
│ place_id (PK)       │
│ name                │
│ category            │
│ location (GEOGRAPHY) │
│ rating              │
│ address_components │
└─────────────────────┘

┌─────────────────────┐
│      routes         │
├─────────────────────┤
│ route_id (PK)       │
│ origin_lat/lng      │
│ dest_lat/lng        │
│ distance_meters     │
│ duration_seconds    │
│ polyline            │
│ mode                │
│ cached_at           │
└─────────────────────┘

┌─────────────────────┐
│ traffic_segments   │
├─────────────────────┤
│ segment_id (PK)     │
│ timestamp (PK)      │
│ speed_kmh           │
│ congestion_level    │
│ polyline            │
└─────────────────────┘
         │
         │ 1:N
         ▼
┌─────────────────────┐
│ traffic_incidents   │
├─────────────────────┤
│ incident_id (PK)    │
│ segment_id (FK)     │
│ type                │
│ severity            │
│ location_lat/lng    │
│ status              │
└─────────────────────┘

┌─────────────────────┐
│  user_locations     │
├─────────────────────┤
│ location_id (PK)    │
│ user_id             │
│ latitude          │
│ longitude           │
│ timestamp          │
│ retention_until    │
└─────────────────────┘
```

---

## Pagination Strategy

**Cursor-Based Pagination:**

For search results, we use cursor-based pagination:

```json
{
  "places": [...],
  "pagination": {
    "cursor": "eyJvZmZzZXQiOjIwLCJxdWVyeSI6ImNvZmZlZSJ9",
    "has_more": true,
    "next_cursor": "eyJvZmZzZXQiOjQwLCJxdWVyeSI6ImNvZmZlZSJ9"
  }
}
```

**Why Cursor-Based?**
- Consistent results even if data changes
- No duplicate results on concurrent updates
- Efficient for large result sets

**Implementation:**
```java
public class CursorPagination {
    
    public String encodeCursor(int offset, String query) {
        Map<String, Object> cursor = Map.of(
            "offset", offset,
            "query", query
        );
        return Base64.getEncoder().encodeToString(
            objectMapper.writeValueAsBytes(cursor)
        );
    }
    
    public Map<String, Object> decodeCursor(String cursor) {
        byte[] decoded = Base64.getDecoder().decode(cursor);
        return objectMapper.readValue(decoded, Map.class);
    }
}
```

---

## Summary

| Component         | Technology/Approach                     |
| ----------------- | --------------------------------------- |
| POI Storage      | PostgreSQL + PostGIS                    |
| Route Cache       | Redis with TTL                          |
| Map Tiles         | S3 + CDN                                |
| Traffic Data      | TimescaleDB (time-series)                |
| Geospatial Index  | PostGIS GIST indexes                    |
| Full-text Search  | PostgreSQL GIN indexes                  |
| Pagination        | Cursor-based                            |


