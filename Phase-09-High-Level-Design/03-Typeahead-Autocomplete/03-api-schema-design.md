# Typeahead / Autocomplete - API & Schema Design

## API Design Philosophy

Typeahead API is unique because:
1. **Extreme latency sensitivity**: Every millisecond matters
2. **High frequency**: Called on every keystroke
3. **Simple interface**: Just prefix in, suggestions out
4. **Stateless**: No session required (personalization via cookie/header)

---

## Base URL Structure

```
Production: https://suggest.search.com/v1
CDN Edge:   https://suggest-edge.search.com/v1
```

---

## Core API Endpoint

### Get Suggestions

**Endpoint:** `GET /v1/suggestions`

This is the only critical endpoint. Keep it simple and fast.

**Request:**
```http
GET /v1/suggestions?q=how+to&limit=10&lang=en HTTP/1.1
Host: suggest.search.com
Accept: application/json
X-User-ID: abc123  (optional, for personalization)
X-Session-ID: xyz789  (optional, for session context)
```

**Query Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| q | string | Yes | - | Search prefix (URL encoded) |
| limit | int | No | 10 | Max suggestions (1-20) |
| lang | string | No | en | Language code |
| client | string | No | web | Client type (web, ios, android) |
| hl | string | No | - | Highlight matching prefix |

**Response (200 OK):**
```json
{
  "query": "how to",
  "suggestions": [
    {
      "text": "how to tie a tie",
      "score": 0.95,
      "type": "query"
    },
    {
      "text": "how to lose weight",
      "score": 0.92,
      "type": "query"
    },
    {
      "text": "how to make money online",
      "score": 0.89,
      "type": "query"
    },
    {
      "text": "how to screenshot on mac",
      "score": 0.85,
      "type": "query"
    },
    {
      "text": "how to cook rice",
      "score": 0.82,
      "type": "query"
    }
  ],
  "metadata": {
    "took_ms": 12,
    "from_cache": true
  }
}
```

**With Highlighting:**
```http
GET /v1/suggestions?q=how+to&hl=true
```

```json
{
  "query": "how to",
  "suggestions": [
    {
      "text": "how to tie a tie",
      "highlighted": "<b>how to</b> tie a tie",
      "score": 0.95
    }
  ]
}
```

### Compact Response Format

For bandwidth optimization, support compact format:

```http
GET /v1/suggestions?q=how+to&format=compact
```

```json
["how to tie a tie","how to lose weight","how to make money online"]
```

This reduces response size by ~60%.

---

## Response Headers

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Cache-Control: public, max-age=300
X-Response-Time: 12ms
X-Cache: HIT
X-Server: suggest-us-east-1a-42
```

### Cache Headers Explanation

| Header | Value | Purpose |
|--------|-------|---------|
| Cache-Control | public, max-age=300 | CDN/browser cache for 5 min |
| Vary | Accept-Encoding | Different cache for gzip vs plain |
| ETag | "abc123" | Conditional requests |

---

## Error Responses

```json
// 400 Bad Request - Missing query
{
  "error": {
    "code": "MISSING_QUERY",
    "message": "Query parameter 'q' is required"
  }
}

// 400 Bad Request - Query too short
{
  "error": {
    "code": "QUERY_TOO_SHORT",
    "message": "Query must be at least 2 characters"
  }
}

// 429 Too Many Requests
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Slow down.",
    "retry_after": 1
  }
}

// 503 Service Unavailable
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Suggestions temporarily unavailable"
  }
}
```

---

## Data Structures

### The Trie (Prefix Tree)

A Trie is the core data structure for typeahead. It stores strings character by character, enabling O(prefix length) lookups.

**Visual Example:**
```
Root
├── h
│   └── o
│       └── w
│           ├── [END: "how"] (freq: 1000)
│           └── (space)
│               └── t
│                   └── o
│                       ├── [END: "how to"] (freq: 5000)
│                       └── (space)
│                           ├── t
│                           │   └── i
│                           │       └── e
│                           │           └── [END: "how to tie"] (freq: 800)
│                           └── l
│                               └── o
│                                   └── s
│                                       └── e
│                                           └── [END: "how to lose"] (freq: 600)
```

### Trie Node Structure

```java
public class TrieNode {
    // Character at this node (implicit in map key for root)
    private char character;
    
    // Children nodes (a-z, 0-9, space, etc.)
    private Map<Character, TrieNode> children;
    
    // Is this a complete word/query?
    private boolean isEndOfQuery;
    
    // Frequency/popularity score (for ranking)
    private long frequency;
    
    // Pre-computed top suggestions from this prefix
    // This is the key optimization!
    private List<Suggestion> topSuggestions;
    
    // Metadata
    private long lastUpdated;
}

public class Suggestion {
    private String text;
    private double score;
    private String type;  // query, trending, personalized
}
```

### Why Pre-compute Top Suggestions?

Without pre-computation:
```
1. Find all words with prefix "how to" (could be millions)
2. Sort by frequency
3. Return top 10
Time: O(N log N) where N = matching queries
```

With pre-computation:
```
1. Navigate to "how to" node
2. Return node.topSuggestions
Time: O(prefix length) = O(6) for "how to"
```

**Trade-off**: More memory, but O(1) lookup for suggestions.

---

## Database Schema (For Pipeline)

### Query Aggregates Table

```sql
CREATE TABLE query_aggregates (
    query_hash BIGINT PRIMARY KEY,  -- Hash of normalized query
    query_text VARCHAR(200) NOT NULL,
    
    -- Frequency data
    hourly_count BIGINT DEFAULT 0,
    daily_count BIGINT DEFAULT 0,
    weekly_count BIGINT DEFAULT 0,
    monthly_count BIGINT DEFAULT 0,
    
    -- Timestamps
    first_seen TIMESTAMP,
    last_seen TIMESTAMP,
    
    -- Metadata
    language VARCHAR(5) DEFAULT 'en',
    is_filtered BOOLEAN DEFAULT FALSE,  -- Blocked content
    
    -- Computed score
    popularity_score DOUBLE PRECISION
);

-- Index for building Trie
CREATE INDEX idx_query_popularity ON query_aggregates(popularity_score DESC)
    WHERE is_filtered = FALSE;

-- Index for language filtering
CREATE INDEX idx_query_language ON query_aggregates(language, popularity_score DESC);
```

### Trending Queries Table

```sql
CREATE TABLE trending_queries (
    query_hash BIGINT PRIMARY KEY,
    query_text VARCHAR(200) NOT NULL,
    
    -- Trending metrics
    current_hour_count BIGINT,
    previous_hour_count BIGINT,
    growth_rate DOUBLE PRECISION,  -- (current - previous) / previous
    
    -- Time window
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    
    -- Ranking
    trend_score DOUBLE PRECISION
);

CREATE INDEX idx_trending_score ON trending_queries(trend_score DESC);
```

### Blocked Queries Table

```sql
CREATE TABLE blocked_queries (
    query_hash BIGINT PRIMARY KEY,
    query_text VARCHAR(200) NOT NULL,
    
    block_reason VARCHAR(50),  -- profanity, illegal, spam
    blocked_at TIMESTAMP DEFAULT NOW(),
    blocked_by VARCHAR(100)
);
```

---

## Scoring Algorithm

### Base Popularity Score

```java
public double calculatePopularityScore(QueryAggregate query) {
    // Time-weighted frequency
    double hourlyWeight = 10.0;
    double dailyWeight = 5.0;
    double weeklyWeight = 2.0;
    double monthlyWeight = 1.0;
    
    double weightedCount = 
        query.getHourlyCount() * hourlyWeight +
        query.getDailyCount() * dailyWeight +
        query.getWeeklyCount() * weeklyWeight +
        query.getMonthlyCount() * monthlyWeight;
    
    // Apply log smoothing (prevents very popular queries from dominating)
    return Math.log10(weightedCount + 1);
}
```

### Trending Boost

```java
public double calculateTrendingBoost(QueryAggregate query) {
    if (query.getGrowthRate() > 2.0) {  // 200% growth
        return 1.5;  // 50% boost
    } else if (query.getGrowthRate() > 1.0) {  // 100% growth
        return 1.2;  // 20% boost
    }
    return 1.0;  // No boost
}
```

### Final Score

```java
public double calculateFinalScore(QueryAggregate query, UserContext user) {
    double baseScore = calculatePopularityScore(query);
    double trendingBoost = calculateTrendingBoost(query);
    double personalizationBoost = calculatePersonalizationBoost(query, user);
    
    return baseScore * trendingBoost * personalizationBoost;
}
```

---

## Personalization

### User Context

```java
public class UserContext {
    private String userId;
    private List<String> recentSearches;  // Last 20 searches
    private String location;              // Country/region
    private String language;
    private Set<String> interests;        // Inferred from history
}
```

### Personalization Boost

```java
public double calculatePersonalizationBoost(String query, UserContext user) {
    // Exact match with recent search
    if (user.getRecentSearches().contains(query)) {
        return 2.0;  // Strong boost
    }
    
    // Prefix match with recent search
    for (String recent : user.getRecentSearches()) {
        if (recent.startsWith(query)) {
            return 1.5;  // Moderate boost
        }
    }
    
    // Interest match (simplified)
    for (String interest : user.getInterests()) {
        if (query.contains(interest)) {
            return 1.2;  // Small boost
        }
    }
    
    return 1.0;  // No boost
}
```

---

## Query Normalization

Before storing or looking up queries, normalize them:

```java
public class QueryNormalizer {
    
    public String normalize(String query) {
        if (query == null) return "";
        
        return query
            .toLowerCase()              // Case insensitive
            .trim()                      // Remove leading/trailing spaces
            .replaceAll("\\s+", " ")     // Collapse multiple spaces
            .replaceAll("[^a-z0-9 ]", ""); // Remove special chars (simplified)
    }
    
    public long hash(String normalizedQuery) {
        // Use consistent hashing for distributed storage
        return Hashing.murmur3_128().hashString(normalizedQuery, StandardCharsets.UTF_8)
            .asLong();
    }
}
```

---

## Content Filtering

### Filter Pipeline

```java
public class ContentFilter {
    
    private final Set<String> blockedWords;
    private final Set<Long> blockedHashes;
    private final Pattern profanityPattern;
    
    public FilterResult filter(String query) {
        String normalized = normalize(query);
        
        // Check blocklist
        if (blockedHashes.contains(hash(normalized))) {
            return FilterResult.blocked("BLOCKLIST");
        }
        
        // Check profanity
        if (profanityPattern.matcher(normalized).find()) {
            return FilterResult.blocked("PROFANITY");
        }
        
        // Check for PII patterns (email, phone, SSN)
        if (containsPII(normalized)) {
            return FilterResult.blocked("PII");
        }
        
        return FilterResult.allowed();
    }
}
```

---

## API Versioning

### Strategy

Use query parameter versioning for simplicity:

```
/v1/suggestions?q=...  (current)
/v2/suggestions?q=...  (future)
```

### Backward Compatibility

- v1 response format maintained indefinitely
- New fields added as optional
- Deprecated fields marked in documentation

---

## Rate Limiting

### Limits

| Client Type | Limit | Window |
|-------------|-------|--------|
| Anonymous | 100 req/min | Per IP |
| Authenticated | 1000 req/min | Per user |
| API Partner | 10000 req/min | Per API key |

### Implementation

```java
@Component
public class SuggestionRateLimiter {
    
    private final RedisTemplate<String, Long> redis;
    
    public boolean isAllowed(String clientId) {
        String key = "ratelimit:suggest:" + clientId;
        Long count = redis.opsForValue().increment(key);
        
        if (count == 1) {
            redis.expire(key, Duration.ofMinutes(1));
        }
        
        return count <= getLimit(clientId);
    }
}
```

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary endpoint | `GET /v1/suggestions?q={prefix}` |
| Response format | JSON with suggestions array |
| Data structure | Trie with pre-computed suggestions |
| Scoring | Popularity + trending + personalization |
| Normalization | Lowercase, trim, collapse spaces |
| Content filtering | Blocklist + profanity + PII |
| Caching | 5 min at CDN, browser |
| Rate limiting | Per-IP and per-user |

