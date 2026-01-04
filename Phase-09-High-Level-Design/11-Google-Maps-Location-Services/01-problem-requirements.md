# Google Maps / Location Services - Problem & Requirements

## What is a Location Services System?

A Location Services System (like Google Maps) provides geospatial functionality including map display, location search, routing, navigation, and real-time traffic information. It enables users to find places, get directions, calculate ETAs, and visualize geographic data on interactive maps.

**Example:**

```
User searches: "coffee shops near me"
  ↓
System finds nearby coffee shops
  ↓
User selects one and requests directions
  ↓
System calculates route and ETA
  ↓
User starts navigation with turn-by-turn directions
```

### Why Does This Exist?

1. **Navigation**: Help users get from point A to point B efficiently
2. **Location Discovery**: Find nearby businesses, restaurants, points of interest
3. **Real-time Updates**: Traffic conditions, road closures, accidents
4. **Geospatial Analysis**: Distance calculations, area searches, proximity queries
5. **Business Intelligence**: Location analytics, foot traffic patterns
6. **Emergency Services**: Route emergency vehicles, locate users in distress

### What Breaks Without It?

- Users can't find nearby businesses
- No efficient route planning
- No real-time traffic information
- Emergency services can't locate users quickly
- Delivery services can't optimize routes
- Location-based recommendations don't work

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question                            | Why It Matters                 | Assumed Answer                              |
| ----------------------------------- | ------------------------------ | ------------------------------------------- |
| What's the scale (users, requests)? | Determines infrastructure size | 100M daily active users, 10K QPS            |
| What features are required?         | Affects complexity             | Search, routing, ETA, map tiles, traffic    |
| Do we need real-time traffic?       | Adds significant complexity    | Yes, real-time traffic updates              |
| What's the geographic scope?        | Affects data storage           | Global coverage                             |
| Do we need offline support?         | Affects caching strategy       | Yes, basic offline maps                     |
| What's the accuracy requirement?    | Affects data sources           | Street-level accuracy (5-10 meters)         |
| Do we need turn-by-turn navigation? | Adds complexity                | Yes, voice-guided navigation                |
| What's the budget for map data?     | Major cost driver              | Use third-party providers (Google Maps API) |

---

## Functional Requirements

### Core Features (Must Have)

1. **Location Search**

   - Search by name, address, category
   - Autocomplete suggestions
   - Geocoding (address → coordinates)
   - Reverse geocoding (coordinates → address)

2. **Map Display**

   - Render map tiles at different zoom levels
   - Display points of interest (POIs)
   - Show user location
   - Pan and zoom functionality

3. **Routing**

   - Calculate routes between two points
   - Multiple route options (fastest, shortest, avoid tolls)
   - Support different transportation modes (driving, walking, transit)
   - Handle waypoints

4. **ETA Calculation**

   - Real-time ETA based on current traffic
   - Historical ETA for planning
   - ETA updates during navigation

5. **Traffic Information**

   - Real-time traffic conditions
   - Traffic incidents (accidents, road closures)
   - Traffic layer overlay on map

6. **Nearby Search**
   - Find places within radius
   - Filter by category (restaurants, gas stations, etc.)
   - Sort by distance or rating

### Secondary Features (Nice to Have)

7. **Turn-by-Turn Navigation**

   - Voice-guided directions
   - Lane guidance
   - Real-time rerouting

8. **Offline Maps**

   - Download maps for offline use
   - Basic routing without internet

9. **Location History**

   - Track user location over time
   - Location-based reminders

10. **Geofencing**
    - Define geographic boundaries
    - Trigger events when entering/leaving areas

---

## Non-Functional Requirements

### Performance

| Metric                 | Target       | Rationale                       |
| ---------------------- | ------------ | ------------------------------- |
| Search latency         | < 200ms p95  | User expects instant results    |
| Route calculation      | < 500ms p95  | Acceptable for complex routing  |
| Map tile load          | < 100ms p95  | Smooth map rendering            |
| ETA accuracy           | ±2 minutes   | Useful for planning             |
| Traffic update latency | < 30 seconds | Real-time enough for navigation |

### Scale

| Metric                | Value       | Calculation               |
| --------------------- | ----------- | ------------------------- |
| Daily active users    | 100 million | Given assumption          |
| Requests per user/day | 10          | Average usage             |
| Total requests/day    | 1 billion   | 100M × 10                 |
| QPS (average)         | 11,500      | 1B / 86,400 seconds       |
| QPS (peak)            | 50,000      | 4-5x average (rush hours) |

### Reliability

- **Availability**: 99.9% uptime (8.76 hours downtime/year)
- **Data Freshness**: Traffic data < 1 minute old
- **Map Accuracy**: Street-level accuracy (5-10 meters)
- **Route Accuracy**: 95% of routes are optimal or near-optimal

### Compliance

- **Privacy**: User location data must be anonymized
- **GDPR**: User consent for location tracking
- **Data Retention**: Location history deleted after 90 days (configurable)

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Street View**: 360-degree imagery
2. **Indoor Maps**: Building floor plans
3. **3D Maps**: Three-dimensional rendering
4. **Satellite Imagery**: High-resolution satellite views
5. **User-Generated Content**: Reviews, photos, ratings
6. **Social Features**: Location sharing, check-ins
7. **Advanced Analytics**: Business intelligence dashboards

---

## System Constraints

### Technical Constraints

1. **Map Data**: Requires third-party providers (Google Maps, Mapbox, HERE)

   - Cost: $5-7 per 1000 requests
   - Rate limits: 25,000 requests/day (free tier)

2. **Geospatial Queries**: Complex spatial indexing required

   - QuadTree, R-tree, or Geohash for proximity searches
   - Cannot use standard SQL indexes efficiently

3. **Real-time Updates**: Traffic data requires streaming infrastructure

   - Millions of location updates per second
   - Low-latency processing required

4. **Storage**: Massive data requirements
   - Map tiles: 100+ TB globally
   - POI data: 100M+ points
   - Traffic data: 1TB+ per day

### Business Constraints

1. **Cost**: Map data licensing is expensive
2. **Legal**: Must comply with geolocation privacy laws
3. **Accuracy**: Liability for incorrect directions

---

## Success Metrics

How do we know if the location service is working well?

| Metric                 | Target     | How to Measure                              |
| ---------------------- | ---------- | ------------------------------------------- |
| Search latency p95     | < 200ms    | Time from query to results                  |
| Route calculation p95  | < 500ms    | Time to compute route                       |
| ETA accuracy           | ±2 minutes | Difference from actual arrival time         |
| Map tile load p95      | < 100ms    | Time to load and render tile                |
| Traffic update latency | < 30s      | Time from event to user notification        |
| Search success rate    | > 95%      | % of queries returning results              |
| Route success rate     | > 99%      | % of route requests successfully calculated |

---

## User Stories

### Story 1: Search for Nearby Coffee Shop

```
As a user,
I want to find coffee shops near my current location,
So that I can get directions to the nearest one.

Acceptance Criteria:
- Search returns results within 5km
- Results sorted by distance
- Each result shows distance and rating
- Clicking result shows on map
- Can request directions
```

### Story 2: Get Driving Directions

```
As a user,
I want to get turn-by-turn directions from my location to a destination,
So that I can navigate efficiently.

Acceptance Criteria:
- Route calculated in < 500ms
- Multiple route options shown (fastest, shortest)
- ETA displayed with traffic consideration
- Turn-by-turn directions available
- Real-time rerouting if traffic changes
```

### Story 3: Real-time Traffic Updates

```
As a user,
I want to see current traffic conditions on my route,
So that I can avoid congestion.

Acceptance Criteria:
- Traffic data updates every 30 seconds
- Traffic incidents displayed on map
- Route automatically rerouted if faster path available
- ETA updated based on current traffic
```

---

## Core Components Overview

### 1. Geocoding Service

- Converts addresses to coordinates
- Reverse geocoding (coordinates to addresses)
- Address normalization and validation

### 2. Search Service

- Full-text search for places
- Autocomplete suggestions
- Category-based filtering

### 3. Routing Engine

- Calculates optimal routes
- Multiple algorithms (Dijkstra, A\*, Contraction Hierarchies)
- Real-time traffic integration

### 4. Map Tile Service

- Generates/serves map tiles
- Multiple zoom levels
- Caching strategy

### 5. Traffic Service

- Aggregates traffic data
- Real-time incident detection
- Historical traffic patterns

### 6. Geospatial Index

- Efficient proximity searches
- QuadTree or R-tree implementation
- Nearby POI queries

### 7. Location Service

- User location tracking
- Geofencing
- Location history

---

## Interview Tips

### What Interviewers Look For

1. **Geospatial awareness**: Do you understand spatial indexing?
2. **Scale understanding**: Can you handle millions of queries?
3. **Real-time processing**: How do you handle traffic updates?
4. **Caching strategy**: How do you optimize map tile delivery?

### Common Mistakes

1. **Ignoring geospatial indexing**: Standard SQL won't work for proximity
2. **Underestimating map tile storage**: 100+ TB globally
3. **Forgetting traffic updates**: Real-time data requires streaming
4. **No caching strategy**: Map tiles are expensive to generate

### Good Follow-up Questions from Interviewer

- "How do you handle geospatial queries at scale?"
- "What's your strategy for real-time traffic updates?"
- "How do you cache map tiles efficiently?"
- "How do you ensure route accuracy?"

---

## Summary

| Aspect              | Decision                                    |
| ------------------- | ------------------------------------------- |
| Primary use case    | Location search, routing, navigation        |
| Scale               | 100M daily active users, 10K QPS            |
| Search latency      | < 200ms p95                                 |
| Route calculation   | < 500ms p95                                 |
| Geospatial indexing | QuadTree/R-tree for proximity searches      |
| Map data            | Third-party providers (Google Maps API)     |
| Traffic updates     | Real-time streaming (< 30s latency)         |
| Caching             | CDN for map tiles, Redis for search results |
