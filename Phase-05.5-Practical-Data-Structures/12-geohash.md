# 🎯 Geohash

## 0️⃣ Prerequisites

Before diving into Geohash, you need to understand:

### Latitude and Longitude
**Latitude** measures north-south position (-90° to +90°).
**Longitude** measures east-west position (-180° to +180°).

```java
// San Francisco
double latitude = 37.7749;   // North of equator
double longitude = -122.4194; // West of Prime Meridian
```

### Binary Representation
Numbers can be represented in binary (base 2).

```
Decimal 5 = Binary 101
Decimal 10 = Binary 1010
```

### Base32 Encoding
**Base32** uses 32 characters (0-9, b-z excluding a, i, l, o) to represent binary data compactly.

```
Binary: 01101 = Decimal 13 = Base32 'e'
Binary: 11111 = Decimal 31 = Base32 'z'
```

### Prefix Matching
Strings with the same prefix are related. "abc123" and "abc456" share prefix "abc".

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Core Problem: Efficient Proximity Search

Imagine you're building a "find nearby restaurants" feature. You have 10 million restaurants worldwide.

**Naive approach:**
```java
List<Restaurant> findNearby(double lat, double lng, double radiusKm) {
    List<Restaurant> nearby = new ArrayList<>();
    for (Restaurant r : allRestaurants) {
        if (distance(lat, lng, r.lat, r.lng) <= radiusKm) {
            nearby.add(r);
        }
    }
    return nearby;
}
// O(n) for every query - scanning 10 million records!
```

**The challenge:**
- Latitude/longitude are two separate values
- Standard indexes work on single columns
- How do you index two-dimensional data in a one-dimensional database?

### The Geohash Solution

Geohash encodes a 2D location into a 1D string:

```
San Francisco: (37.7749, -122.4194) → "9q8yy"
Nearby location: (37.7750, -122.4190) → "9q8yy"  (same prefix!)
Far location: (40.7128, -74.0060) → "dr5ru"     (different prefix)
```

**Key insight**: Nearby locations share the same geohash prefix!

### Real-World Pain Points

**Scenario 1: Redis Geo Commands**
Redis uses geohash internally for GEOADD, GEORADIUS commands.

**Scenario 2: Elasticsearch Geo Queries**
Elasticsearch uses geohash for efficient geo_distance queries.

**Scenario 3: Database Sharding**
Shard location data by geohash prefix for geographic locality.

### What Breaks Without Geohash?

| Without Geohash | With Geohash |
|-----------------|--------------|
| 2D data, can't use standard index | 1D string, standard B-tree index |
| O(n) proximity search | O(log n) with prefix matching |
| Complex spatial joins | Simple string prefix queries |
| Hard to shard by location | Easy sharding by prefix |

---

## 2️⃣ Intuition and Mental Model

### The Recursive Grid Analogy

Imagine dividing the world map recursively:

**Step 1**: Divide the world into 32 cells (8 columns × 4 rows)
```
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|
| 8 | 9 | b | c | d | e | f | g |
| h | j | k | m | n | p | q | r |
| s | t | u | v | w | x | y | z |

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
├───┼───┼───┼───┼───┼───┼───┼───┤
│ 8 │ 9 │ b │ c │ d │ e │ f │ g │
├───┼───┼───┼───┼───┼───┼───┼───┤
│ h │ j │ k │ m │ n │ p │ q │ r │
├───┼───┼───┼───┼───┼───┼───┼───┤
│ s │ t │ u │ v │ w │ x │ y │ z │
└───┴───┴───┴───┴───┴───┴───┴───┘
```
</details>
```

San Francisco is in cell "9" (second row, second column).

**Step 2**: Divide cell "9" into 32 sub-cells
```
Cell "9":
| 90 | 91 | 92 | 93 | 94 | 95 | 96 | 97 |
|---|---|---|---|---|---|---|---|
| 98 | 99 | 9b | 9c | 9d | 9e | 9f | 9g |
| 9h | 9j | 9k | 9m | 9n | 9p | **9q** ← San Francisco | 9r |
| 9s | 9t | 9u | 9v | 9w | 9x | 9y | 9z |

<details>
<summary>ASCII diagram (reference)</summary>

```text
Cell "9":
┌───┬───┬───┬───┬───┬───┬───┬───┐
│90 │91 │92 │93 │94 │95 │96 │97 │
├───┼───┼───┼───┼───┼───┼───┼───┤
│98 │99 │9b │9c │9d │9e │9f │9g │
├───┼───┼───┼───┼───┼───┼───┼───┤
│9h │9j │9k │9m │9n │9p │9q │9r │  ← San Francisco is in "9q"
├───┼───┼───┼───┼───┼───┼───┼───┤
│9s │9t │9u │9v │9w │9x │9y │9z │
└───┴───┴───┴───┴───┴───┴───┴───┘
```
</details>
```

**Step 3**: Continue subdividing "9q" → "9q8" → "9q8y" → "9q8yy" → ...

Each additional character makes the cell 32× smaller!

### The Key Insight

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────┐
│                     GEOHASH KEY INSIGHT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "Geohash converts 2D coordinates to 1D string"                 │
│                                                                  │
│  Properties:                                                     │
│  1. Nearby points share prefix (usually)                        │
│  2. Longer geohash = more precise location                      │
│  3. Can use standard string indexes                             │
│  4. Hierarchical: "9q8" contains all "9q8*" locations          │
│                                                                  │
│  Precision:                                                      │
│  1 char  → ~5000 km × 5000 km                                   │
│  4 chars → ~40 km × 20 km                                       │
│  6 chars → ~1.2 km × 600 m                                      │
│  8 chars → ~40 m × 20 m                                         │
│  12 chars → ~4 cm × 2 cm                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
</details>

---

## 3️⃣ How It Works Internally

### The Encoding Algorithm

Geohash interleaves the binary representations of latitude and longitude:

**Step 1: Binary search on longitude**
```
Longitude range: [-180, 180]
Target: -122.4194

Is -122.4194 >= 0? NO  → bit 0, range becomes [-180, 0]
Is -122.4194 >= -90? NO → bit 0, range becomes [-180, -90]
Is -122.4194 >= -135? YES → bit 1, range becomes [-135, -90]
Is -122.4194 >= -112.5? NO → bit 0, range becomes [-135, -112.5]
Is -122.4194 >= -123.75? YES → bit 1, range becomes [-123.75, -112.5]
...

Longitude bits: 0 0 1 0 1 ...
```

**Step 2: Binary search on latitude**
```
Latitude range: [-90, 90]
Target: 37.7749

Is 37.7749 >= 0? YES → bit 1, range becomes [0, 90]
Is 37.7749 >= 45? NO  → bit 0, range becomes [0, 45]
Is 37.7749 >= 22.5? YES → bit 1, range becomes [22.5, 45]
Is 37.7749 >= 33.75? YES → bit 1, range becomes [33.75, 45]
Is 37.7749 >= 39.375? NO → bit 0, range becomes [33.75, 39.375]
...

Latitude bits: 1 0 1 1 0 ...
```

**Step 3: Interleave bits (longitude first)**
```
Longitude: 0 0 1 0 1 ...
Latitude:  1 0 1 1 0 ...

Interleaved: 01 00 11 01 10 ...
           = 0 1 0 0 1 1 0 1 1 0 ...
```

**Step 4: Convert to Base32**
```
Group into 5-bit chunks:
01001 10110 ...
  9     q   ...

Geohash: "9q..."
```

### Precision Table

| Length | Lat Error | Lng Error | km Error |
|--------|-----------|-----------|----------|
| 1 | ±23° | ±23° | ~2500 km |
| 2 | ±2.8° | ±5.6° | ~630 km |
| 3 | ±0.7° | ±0.7° | ~78 km |
| 4 | ±0.087° | ±0.18° | ~20 km |
| 5 | ±0.022° | ±0.022° | ~2.4 km |
| 6 | ±0.0027° | ±0.0055° | ~610 m |
| 7 | ±0.00068° | ±0.00068° | ~76 m |
| 8 | ±0.000086° | ±0.00017° | ~19 m |

### Decoding Algorithm

Reverse the process:
1. Convert Base32 to binary
2. De-interleave into latitude and longitude bits
3. Binary search in reverse to get coordinate ranges

```java
// Decoding returns a bounding box, not exact point
BoundingBox decode("9q8yy") → {
    minLat: 37.7739,
    maxLat: 37.7766,
    minLng: -122.4203,
    maxLng: -122.4149
}
```

---

## 4️⃣ Simulation: Step-by-Step Encoding

### Encode San Francisco (37.7749, -122.4194)

**Longitude: -122.4194**
```
Step  Range              Mid        Bit
1     [-180, 180]        0          0 (target < mid)
2     [-180, 0]          -90        0 (target < mid)
3     [-180, -90]        -135       1 (target >= mid)
4     [-135, -90]        -112.5     0 (target < mid)
5     [-135, -112.5]     -123.75    1 (target >= mid)
6     [-123.75, -112.5]  -118.125   0 (target < mid)
7     [-123.75, -118.125] -120.9375 0 (target < mid)
8     [-123.75, -120.9375] -122.34  0 (target < mid)
9     [-123.75, -122.34]  -123.05   1 (target >= mid)
10    [-123.05, -122.34]  -122.695  1 (target >= mid)

Longitude bits: 0 0 1 0 1 0 0 0 1 1 ...
```

**Latitude: 37.7749**
```
Step  Range          Mid       Bit
1     [-90, 90]      0         1 (target >= mid)
2     [0, 90]        45        0 (target < mid)
3     [0, 45]        22.5      1 (target >= mid)
4     [22.5, 45]     33.75     1 (target >= mid)
5     [33.75, 45]    39.375    0 (target < mid)
6     [33.75, 39.375] 36.5625  1 (target >= mid)
7     [36.5625, 39.375] 37.97  0 (target < mid)
8     [36.5625, 37.97] 37.27   1 (target >= mid)
9     [37.27, 37.97]  37.62    1 (target >= mid)
10    [37.62, 37.97]  37.795   0 (target < mid)

Latitude bits: 1 0 1 1 0 1 0 1 1 0 ...
```

**Interleave (longitude, latitude alternating):**
```
Position: 1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16 17 18 19 20
Lng:      0     0     1     0     1     0     0     0     1     1
Lat:         1     0     1     1     0     1     0     1     1     0

Combined: 0  1  0  0  1  1  0  1  1  0  0  1  0  0  0  1  1  1  1  0
```

**Group into 5-bit chunks:**
```
01001 10110 01000 11110 ...
  9     q     8     y   ...
```

**Result: "9q8y..."**

---

## 5️⃣ How Engineers Use This in Production

### Redis Geospatial Commands

Redis uses geohash internally for its GEO commands:

```bash
# Add locations (stored as geohash internally)
GEOADD restaurants -122.4194 37.7749 "pizza_place"
GEOADD restaurants -122.4089 37.7837 "burger_joint"
GEOADD restaurants -122.4000 37.7900 "sushi_bar"

# Find restaurants within 2km radius
GEORADIUS restaurants -122.41 37.78 2 km WITHDIST
# Returns:
# 1) "pizza_place" - 0.8 km
# 2) "burger_joint" - 1.2 km

# Get geohash of a location
GEOHASH restaurants "pizza_place"
# Returns: "9q8yyk8yuv0"
```

### Elasticsearch Geo Queries

Elasticsearch uses geohash for efficient geo queries:

```json
// Index with geohash
PUT /restaurants
{
  "mappings": {
    "properties": {
      "location": {
        "type": "geo_point"
      }
    }
  }
}

// Query by geohash cell
GET /restaurants/_search
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "top_left": "9q8z",
        "bottom_right": "9q8y"
      }
    }
  }
}

// Aggregation by geohash grid
GET /restaurants/_search
{
  "aggs": {
    "grid": {
      "geohash_grid": {
        "field": "location",
        "precision": 5
      }
    }
  }
}
```

### Database Indexing with Geohash

```sql
-- PostgreSQL: Store geohash as indexed column
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    lat DOUBLE PRECISION,
    lng DOUBLE PRECISION,
    geohash VARCHAR(12)
);

CREATE INDEX idx_geohash ON locations (geohash);

-- Query nearby locations using prefix
SELECT * FROM locations
WHERE geohash LIKE '9q8yy%'
ORDER BY geohash;

-- This uses the B-tree index efficiently!
```

### Sharding by Geohash

<details>
<summary>ASCII diagram (reference)</summary>

```text
┌─────────────────────────────────────────────────────────────────┐
│                    GEOHASH SHARDING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Shard by first 2 characters of geohash:                        │
│                                                                  │
│  Shard "9q" → San Francisco Bay Area                            │
│  Shard "9r" → Los Angeles Area                                  │
│  Shard "dr" → New York Area                                     │
│  Shard "u4" → London Area                                       │
│                                                                  │
│  Benefits:                                                       │
│  - Geographic locality (nearby data on same shard)              │
│  - Queries usually hit single shard                             │
│  - Easy to add capacity per region                              │
│                                                                  │
│  Challenges:                                                     │
│  - Hotspots in dense areas (Manhattan vs rural)                 │
│  - Cross-boundary queries need multiple shards                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
</details>

---

## 6️⃣ Implementation in Java

### Geohash Encoder/Decoder

```java
/**
 * Geohash encoding and decoding implementation.
 */
public class Geohash {
    
    private static final String BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz";
    private static final int[] BITS = {16, 8, 4, 2, 1};
    
    /**
     * Encodes latitude/longitude to geohash string.
     * 
     * @param lat Latitude (-90 to 90)
     * @param lng Longitude (-180 to 180)
     * @param precision Number of characters (1-12)
     * @return Geohash string
     */
    public static String encode(double lat, double lng, int precision) {
        double[] latRange = {-90.0, 90.0};
        double[] lngRange = {-180.0, 180.0};
        
        StringBuilder geohash = new StringBuilder();
        boolean isLng = true;  // Start with longitude
        int bit = 0;
        int ch = 0;
        
        while (geohash.length() < precision) {
            double mid;
            if (isLng) {
                mid = (lngRange[0] + lngRange[1]) / 2;
                if (lng >= mid) {
                    ch |= BITS[bit];
                    lngRange[0] = mid;
                } else {
                    lngRange[1] = mid;
                }
            } else {
                mid = (latRange[0] + latRange[1]) / 2;
                if (lat >= mid) {
                    ch |= BITS[bit];
                    latRange[0] = mid;
                } else {
                    latRange[1] = mid;
                }
            }
            
            isLng = !isLng;
            
            if (bit < 4) {
                bit++;
            } else {
                geohash.append(BASE32.charAt(ch));
                bit = 0;
                ch = 0;
            }
        }
        
        return geohash.toString();
    }
    
    /**
     * Decodes geohash to bounding box.
     * 
     * @param geohash Geohash string
     * @return Bounding box [minLat, maxLat, minLng, maxLng]
     */
    public static double[] decode(String geohash) {
        double[] latRange = {-90.0, 90.0};
        double[] lngRange = {-180.0, 180.0};
        
        boolean isLng = true;
        
        for (char c : geohash.toCharArray()) {
            int cd = BASE32.indexOf(c);
            
            for (int bit : BITS) {
                if (isLng) {
                    double mid = (lngRange[0] + lngRange[1]) / 2;
                    if ((cd & bit) != 0) {
                        lngRange[0] = mid;
                    } else {
                        lngRange[1] = mid;
                    }
                } else {
                    double mid = (latRange[0] + latRange[1]) / 2;
                    if ((cd & bit) != 0) {
                        latRange[0] = mid;
                    } else {
                        latRange[1] = mid;
                    }
                }
                isLng = !isLng;
            }
        }
        
        return new double[]{latRange[0], latRange[1], lngRange[0], lngRange[1]};
    }
    
    /**
     * Decodes geohash to center point.
     */
    public static double[] decodeToPoint(String geohash) {
        double[] bbox = decode(geohash);
        return new double[]{
            (bbox[0] + bbox[1]) / 2,  // center lat
            (bbox[2] + bbox[3]) / 2   // center lng
        };
    }
    
    /**
     * Gets the 8 neighboring geohash cells.
     */
    public static String[] neighbors(String geohash) {
        double[] center = decodeToPoint(geohash);
        double[] bbox = decode(geohash);
        
        double latStep = bbox[1] - bbox[0];
        double lngStep = bbox[3] - bbox[2];
        
        String[] neighbors = new String[8];
        int idx = 0;
        
        for (int dlat = -1; dlat <= 1; dlat++) {
            for (int dlng = -1; dlng <= 1; dlng++) {
                if (dlat == 0 && dlng == 0) continue;
                
                double newLat = center[0] + dlat * latStep;
                double newLng = center[1] + dlng * lngStep;
                
                // Handle wrap-around
                if (newLng > 180) newLng -= 360;
                if (newLng < -180) newLng += 360;
                
                neighbors[idx++] = encode(newLat, newLng, geohash.length());
            }
        }
        
        return neighbors;
    }
    
    /**
     * Calculates the approximate error in meters for a given precision.
     */
    public static double[] precisionError(int precision) {
        // Each character adds 2.5 bits of lat and 2.5 bits of lng precision
        double latError = 90.0 / Math.pow(2, (precision * 5) / 2);
        double lngError = 180.0 / Math.pow(2, (precision * 5 + 1) / 2);
        
        // Convert to meters (approximate)
        double latMeters = latError * 111000;  // 1 degree lat ≈ 111 km
        double lngMeters = lngError * 111000 * Math.cos(Math.toRadians(45));
        
        return new double[]{latMeters, lngMeters};
    }
}
```

### Geohash-based Proximity Search

```java
import java.util.*;

/**
 * Proximity search using geohash.
 */
public class GeohashProximitySearch<T> {
    
    private final Map<String, List<Entry<T>>> index = new HashMap<>();
    private final int precision;
    
    private static class Entry<T> {
        double lat, lng;
        T data;
        
        Entry(double lat, double lng, T data) {
            this.lat = lat;
            this.lng = lng;
            this.data = data;
        }
    }
    
    public GeohashProximitySearch(int precision) {
        this.precision = precision;
    }
    
    /**
     * Adds a location to the index.
     */
    public void add(double lat, double lng, T data) {
        String geohash = Geohash.encode(lat, lng, precision);
        index.computeIfAbsent(geohash, k -> new ArrayList<>())
             .add(new Entry<>(lat, lng, data));
    }
    
    /**
     * Finds all items within approximately the given radius.
     * Note: This is approximate due to geohash cell boundaries.
     */
    public List<T> findNearby(double lat, double lng, double radiusMeters) {
        List<T> results = new ArrayList<>();
        
        // Get the geohash cell for the query point
        String centerHash = Geohash.encode(lat, lng, precision);
        
        // Get neighboring cells (to handle edge cases)
        Set<String> cellsToSearch = new HashSet<>();
        cellsToSearch.add(centerHash);
        for (String neighbor : Geohash.neighbors(centerHash)) {
            cellsToSearch.add(neighbor);
        }
        
        // Search all relevant cells
        for (String cell : cellsToSearch) {
            List<Entry<T>> entries = index.get(cell);
            if (entries != null) {
                for (Entry<T> entry : entries) {
                    double distance = haversineDistance(lat, lng, entry.lat, entry.lng);
                    if (distance <= radiusMeters) {
                        results.add(entry.data);
                    }
                }
            }
        }
        
        return results;
    }
    
    /**
     * Haversine formula for distance between two points.
     */
    private double haversineDistance(double lat1, double lng1, double lat2, double lng2) {
        double R = 6371000;  // Earth radius in meters
        
        double dLat = Math.toRadians(lat2 - lat1);
        double dLng = Math.toRadians(lng2 - lng1);
        
        double a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                   Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2)) *
                   Math.sin(dLng / 2) * Math.sin(dLng / 2);
        
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        
        return R * c;
    }
}
```

### Testing the Implementation

```java
public class GeohashTest {
    
    public static void main(String[] args) {
        testEncodeDecode();
        testNeighbors();
        testProximitySearch();
        testPrecision();
    }
    
    static void testEncodeDecode() {
        System.out.println("=== Encode/Decode ===");
        
        // San Francisco
        double lat = 37.7749;
        double lng = -122.4194;
        
        for (int precision = 1; precision <= 8; precision++) {
            String geohash = Geohash.encode(lat, lng, precision);
            double[] decoded = Geohash.decodeToPoint(geohash);
            double[] error = Geohash.precisionError(precision);
            
            System.out.printf("Precision %d: %s → (%.4f, %.4f) ±%.0fm\n",
                precision, geohash, decoded[0], decoded[1], error[0]);
        }
    }
    
    static void testNeighbors() {
        System.out.println("\n=== Neighbors ===");
        
        String geohash = "9q8yy";
        String[] neighbors = Geohash.neighbors(geohash);
        
        System.out.println("Neighbors of " + geohash + ":");
        for (String neighbor : neighbors) {
            System.out.println("  " + neighbor);
        }
    }
    
    static void testProximitySearch() {
        System.out.println("\n=== Proximity Search ===");
        
        GeohashProximitySearch<String> search = new GeohashProximitySearch<>(6);
        
        // Add some restaurants in SF
        search.add(37.7749, -122.4194, "Pizza Place");
        search.add(37.7751, -122.4180, "Burger Joint");
        search.add(37.7760, -122.4200, "Sushi Bar");
        search.add(37.7900, -122.4000, "Taco Shop");  // Farther away
        
        // Find restaurants within 500m
        List<String> nearby = search.findNearby(37.7749, -122.4194, 500);
        
        System.out.println("Restaurants within 500m:");
        for (String restaurant : nearby) {
            System.out.println("  " + restaurant);
        }
    }
    
    static void testPrecision() {
        System.out.println("\n=== Precision Levels ===");
        
        System.out.println("Precision | Lat Error | Lng Error | ~Distance");
        System.out.println("----------|-----------|-----------|----------");
        
        for (int p = 1; p <= 12; p++) {
            double[] error = Geohash.precisionError(p);
            System.out.printf("    %2d    | %7.0fm  | %7.0fm  | ~%.0fm\n",
                p, error[0], error[1], Math.max(error[0], error[1]));
        }
    }
}
```

---

## 7️⃣ Edge Cases and Limitations

### The Edge Problem

**Problem**: Two nearby points can have completely different geohashes if they're on opposite sides of a cell boundary.

```
┌─────────────┬─────────────┐
│             │             │
│    9q8yy    │    9q8yz    │
│         A ● │ ● B         │
│             │             │
└─────────────┴─────────────┘

A and B are very close, but have different geohash prefixes!
```

**Solution**: Always search neighboring cells:

```java
// Don't just search the target cell
String targetCell = Geohash.encode(lat, lng, precision);

// Also search all 8 neighbors
Set<String> cellsToSearch = new HashSet<>();
cellsToSearch.add(targetCell);
cellsToSearch.addAll(Arrays.asList(Geohash.neighbors(targetCell)));
```

### The Pole Problem

Near the poles, geohash cells become very distorted:

```
At equator: cells are roughly square
At poles: cells are very elongated (narrow and tall)
```

**Mitigation**: Use appropriate precision based on latitude.

### The Antimeridian Problem

The 180°/-180° longitude line causes issues:

```
Point A: (0, 179.9)  → geohash "x..."
Point B: (0, -179.9) → geohash "8..."

These are only 0.2° apart but have completely different geohashes!
```

**Solution**: Special handling for queries near the antimeridian.

### Non-Uniform Cell Sizes

Geohash cells are not uniform in size:
- At equator: ~5km × 5km for precision 5
- At 60° latitude: ~2.5km × 5km for precision 5

---

## 8️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Tradeoffs

| Aspect | Geohash | Quadtree | R-Tree |
|--------|---------|----------|--------|
| Index type | String (B-tree) | Custom | Custom |
| Precision | Configurable | Adaptive | Adaptive |
| Edge handling | Problematic | Natural | Natural |
| Implementation | Simple | Medium | Complex |
| Database support | Universal | Limited | PostGIS |

### Common Pitfalls

**1. Forgetting neighbor cells**

```java
// BAD: Only query exact cell
String hash = Geohash.encode(lat, lng, 5);
List<Location> nearby = db.findByGeohashPrefix(hash);
// Misses points just across the cell boundary!

// GOOD: Always include neighbors
String[] cells = Geohash.neighbors(hash);
List<Location> nearby = db.findByGeohashPrefixes(cells);
```

**2. Wrong precision for use case**

```java
// BAD: Precision 12 for "nearby restaurants" (1cm cells!)
// Generates millions of cells to check

// BAD: Precision 3 for delivery radius (156km cells!)
// Returns too many irrelevant results

// GOOD: Match precision to query radius
// ~5km radius → precision 5
// ~1km radius → precision 6
// ~100m radius → precision 7
```

**3. Using geohash as exact distance**

```java
// BAD: Assuming same prefix means "close enough"
if (hash1.startsWith(hash2.substring(0, 5))) {
    // Close enough!  // WRONG: Could be 5km apart!
}

// GOOD: Use geohash for filtering, then calculate actual distance
List<Location> candidates = findByGeohash(prefix);
return candidates.stream()
    .filter(loc -> distance(loc, target) < radiusKm)
    .collect(toList());
```

**4. Ignoring latitude distortion**

```java
// BAD: Using same precision worldwide
// At equator: 5km cells
// At 60° latitude: 2.5km × 5km cells

// GOOD: Adjust precision or radius based on latitude
int precision = calculatePrecisionForRadius(lat, radiusKm);
```

### Performance Gotchas

**1. Too many prefix queries**

```java
// BAD: 9 separate queries for each neighbor
for (String cell : neighbors) {
    results.addAll(db.query("geohash LIKE ?", cell + "%"));
}

// GOOD: Single query with IN clause or UNION
db.query("geohash LIKE ? OR geohash LIKE ? ...", cells);
```

**2. Not indexing properly**

```java
// BAD: Full table scan
SELECT * FROM locations WHERE geohash LIKE '9q8yy%';
// Without index, this is O(n)!

// GOOD: Create index on geohash column
CREATE INDEX idx_geohash ON locations(geohash);
// Now prefix query uses index
```

---

## 9️⃣ When NOT to Use Geohash

### Anti-Patterns

**1. Exact distance queries**

```java
// WRONG: "Find all points within exactly 500m"
// Geohash cells don't align with circles!

// RIGHT: Use geohash for rough filtering,
// then Haversine formula for exact distance
```

**2. Polygon containment**

```java
// WRONG: "Find points inside this irregular polygon"
// Geohash is for rectangles only

// RIGHT: Use R-Tree or PostGIS with ST_Contains
```

**3. Path/route queries**

```java
// WRONG: "Find points along this route"
// Route may cross many cells inefficiently

// RIGHT: Use specialized routing algorithms
// or segment the route and query each segment
```

**4. Polar regions**

```java
// WRONG: Geohash near poles
// Cells become extremely distorted

// RIGHT: Use different projection or
// specialized polar coordinate systems
```

### Better Alternatives

| Use Case | Better Alternative |
|----------|-------------------|
| Exact distance | Haversine + filtering |
| Complex polygons | PostGIS, R-Tree |
| High precision | S2 Geometry (Google) |
| 3D spatial | Octree, 3D R-Tree |
| Polar regions | UPS projection |

---

## 🔟 Interview Follow-Up Questions with Answers

### L4 (Entry-Level) Questions

**Q1: What is geohash and why is it useful?**

**Answer**: Geohash is a system that encodes a 2D geographic location (latitude/longitude) into a 1D string. It works by recursively dividing the world into cells and encoding which cell a point falls into. It's useful because: (1) Nearby locations share the same prefix, enabling efficient proximity queries, (2) You can use standard string indexes (B-tree) for spatial queries, (3) It's easy to adjust precision by truncating the string, (4) It enables geographic sharding of data.

**Q2: How do you find nearby points using geohash?**

**Answer**: 
1. Encode the query point to geohash
2. Get all 8 neighboring cells (to handle edge cases)
3. Query all points in these 9 cells from your index
4. Filter results by actual distance (since cells are approximate)

The key insight is that you can use a simple prefix query (like `WHERE geohash LIKE '9q8yy%'`) with a standard B-tree index.

### L5 (Senior) Questions

**Q3: What are the limitations of geohash for proximity search?**

**Answer**: Several limitations:

1. **Edge problem**: Two nearby points can have different prefixes if they're on opposite sides of a cell boundary. Solution: Always search neighboring cells.

2. **Non-uniform cells**: Cells are not square, and size varies with latitude. At high latitudes, cells are very elongated.

3. **Precision trade-off**: Coarse precision misses nearby points; fine precision requires checking many cells.

4. **Antimeridian**: Points near 180°/-180° longitude have very different geohashes even if close.

5. **Not a distance metric**: Geohash prefix length doesn't directly correspond to distance.

**Q4: How would you design a "find nearby" feature for a ride-sharing app?**

**Answer**:

```
Architecture:

1. Storage:
   - Redis with GEOADD for active drivers
   - Geohash stored internally
   - TTL to expire inactive drivers

2. Driver Location Updates:
   - Drivers send GPS every 5 seconds
   - Update Redis: GEOADD drivers lng lat driver_id
   - Set TTL on driver key

3. Find Nearby Query:
   - Use GEORADIUS: GEORADIUS drivers lng lat 5 km COUNT 10
   - Returns closest 10 drivers within 5km
   - Already sorted by distance

4. Optimizations:
   - Precision 6 geohash (~1km cells) for initial filter
   - Exact distance calculation for final ranking
   - Cache hot areas (airports, downtown)
   - Pre-compute driver availability

5. Scaling:
   - Shard Redis by geohash prefix
   - Most queries hit single shard
   - Cross-shard for boundary queries
```

### L6 (Staff) Questions

**Q5: Design a global location-based service handling 1 billion locations.**

**Answer**:

```
Architecture:

1. Data Partitioning:
   - Primary partition: geohash prefix (first 2-3 chars)
   - Secondary index: full geohash for queries
   - ~32³ = 32,768 partitions for 3-char prefix

2. Storage Tiers:
   - Hot data (active locations): Redis clusters
   - Warm data (recent): Elasticsearch
   - Cold data (historical): S3 + Athena

3. Query Path:
   - Parse query bounds
   - Determine geohash cells to query
   - Route to appropriate partition(s)
   - Aggregate and filter results

4. Handling Hotspots:
   - Dense areas (Manhattan) need finer partitioning
   - Use adaptive precision based on density
   - Replicate hot partitions

5. Cross-Region:
   - Deploy in multiple regions
   - Route queries to nearest region
   - Async replication for global data

6. Caching:
   - Cache popular queries (city centers)
   - Cache geohash → partition mapping
   - Invalidate on location updates
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Geohash encodes a 2D location (latitude/longitude) into a 1D string by recursively dividing the world into cells. Each character adds precision, and nearby locations share the same prefix. This enables using standard string indexes for spatial queries: "find all locations starting with 9q8yy" is a simple prefix query. The main limitation is the edge problem, where nearby points across cell boundaries have different prefixes, so always search neighboring cells. Geohash is used by Redis (GEOADD), Elasticsearch, and many databases for efficient proximity search.

---

## Summary

Geohash is essential for:
- **Proximity search**: Finding nearby restaurants, drivers, friends
- **Database indexing**: Using B-tree indexes for spatial data
- **Data sharding**: Partitioning by geographic region
- **Caching**: Geographic locality for cache efficiency

Key takeaways:
1. Converts 2D coordinates to 1D string
2. Nearby points share prefix (usually)
3. Longer string = more precision
4. Always search neighboring cells for edge cases
5. Used by Redis, Elasticsearch, MongoDB internally
6. Not perfect: edge problem, non-uniform cells, pole issues

