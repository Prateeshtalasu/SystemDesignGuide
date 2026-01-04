# Recommendation System - API & Schema Design

## API Design Philosophy

Recommendation APIs prioritize:

1. **Low Latency**: Recommendations are in the critical rendering path
2. **Flexibility**: Support multiple recommendation types
3. **Context Awareness**: Accept signals for personalization
4. **Explainability**: Return reasons for recommendations

---

## Base URL Structure

```
Production: https://api.recommendations.com/v1
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
| Free | 60 |
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
      "field": "field_name",  // Optional
      "reason": "Specific reason"  // Optional
    },
    "request_id": "req_123456"  // For tracing
  }
}
```

**Error Codes Reference:**

| HTTP Status | Error Code | Description |
|-------------|------------|-------------|
| 400 | INVALID_INPUT | Request validation failed |
| 400 | INVALID_USER_ID | User ID is invalid |
| 401 | UNAUTHORIZED | Authentication required |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource not found |
| 429 | RATE_LIMITED | Rate limit exceeded |
| 500 | INTERNAL_ERROR | Server error |
| 503 | SERVICE_UNAVAILABLE | Service temporarily unavailable |

**Error Response Examples:**

```json
// 400 Bad Request - Invalid user ID
{
  "error": {
    "code": "INVALID_USER_ID",
    "message": "User ID is invalid",
    "details": {
      "field": "user_id",
      "reason": "User ID must be a valid UUID"
    },
    "request_id": "req_abc123"
  }
}

// 429 Rate Limited
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. Please try again later.",
    "details": {
      "limit": 60,
      "remaining": 0,
      "reset_at": "2024-01-15T11:00:00Z"
    },
    "request_id": "req_xyz789"
  }
}
```

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

**Retry Semantics:**

1. Client sends request with Idempotency-Key
2. Server checks Redis for existing key
3. If found: Return cached response (same status code + body)
4. If not found: Process request, cache response, return result
5. Retries with same key within 24 hours return cached response

**Per-Endpoint Idempotency:**

| Endpoint | Idempotent? | Mechanism |
|----------|-------------|-----------|
| POST /v1/feedback | Yes | Idempotency-Key header |
| PUT /v1/feedback/{id} | Yes | Idempotency-Key or version-based |
| DELETE /v1/feedback/{id} | Yes | Safe to retry (idempotent by design) |

**Implementation Example:**

```java
@PostMapping("/v1/feedback")
public ResponseEntity<FeedbackResponse> submitFeedback(
        @RequestBody FeedbackRequest request,
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey) {
    
    // Check for existing idempotency key
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        String cachedResponse = redisTemplate.opsForValue().get(cacheKey);
        if (cachedResponse != null) {
            FeedbackResponse response = objectMapper.readValue(cachedResponse, FeedbackResponse.class);
            return ResponseEntity.status(response.getStatus()).body(response);
        }
    }
    
    // Create feedback
    Feedback feedback = feedbackService.createFeedback(request, idempotencyKey);
    FeedbackResponse response = FeedbackResponse.from(feedback);
    
    // Cache response if idempotency key provided
    if (idempotencyKey != null) {
        String cacheKey = "idempotency:" + idempotencyKey;
        redisTemplate.opsForValue().set(
            cacheKey, 
            objectMapper.writeValueAsString(response),
            Duration.ofHours(24)
        );
    }
    
    return ResponseEntity.status(201).body(response);
}
```

---

## Core API Endpoints

### 1. Get Personalized Recommendations

**Endpoint:** `GET /v1/recommendations`

**Purpose:** Retrieve personalized recommendations for a user

**Request:**

```http
GET /v1/recommendations?user_id=user_123&context=homepage&limit=50 HTTP/1.1
Host: api.recommendations.com
Authorization: Bearer token
X-Session-Id: session_abc
X-Device-Type: mobile
```

**Query Parameters:**

| Parameter   | Type    | Required | Default | Description                        |
| ----------- | ------- | -------- | ------- | ---------------------------------- |
| user_id     | string  | Yes      | -       | User identifier                    |
| context     | string  | No       | homepage| Where recommendations are shown    |
| limit       | int     | No       | 20      | Number of recommendations          |
| category    | string  | No       | all     | Filter by category                 |
| exclude     | string[]| No       | []      | Item IDs to exclude                |

**Note on Pagination:**
Recommendations are returned as a fixed set and do not require pagination (cursor/offset). This is by design because:
- Recommendations are personalized and context-specific
- Users typically view top 20-50 recommendations
- Each request generates fresh recommendations based on current context
- Pagination would require maintaining state or re-computing recommendations

**Response (200 OK):**

```json
{
  "recommendations": [
    {
      "item_id": "movie_456",
      "score": 0.95,
      "rank": 1,
      "metadata": {
        "title": "Inception",
        "category": "Sci-Fi",
        "thumbnail_url": "https://cdn.example.com/inception.jpg",
        "rating": 4.8
      },
      "explanation": {
        "type": "collaborative",
        "reason": "Because you watched The Matrix",
        "similar_items": ["movie_123"]
      }
    },
    {
      "item_id": "movie_789",
      "score": 0.92,
      "rank": 2,
      "metadata": {
        "title": "Interstellar",
        "category": "Sci-Fi",
        "thumbnail_url": "https://cdn.example.com/interstellar.jpg",
        "rating": 4.7
      },
      "explanation": {
        "type": "content_based",
        "reason": "Popular in Sci-Fi",
        "matching_attributes": ["director", "genre"]
      }
    }
  ],
  "request_id": "req_xyz",
  "model_version": "v2.3.1",
  "generated_at": "2024-01-20T15:30:00Z"
}
```

---

### 2. Get Similar Items

**Endpoint:** `GET /v1/items/{item_id}/similar`

**Purpose:** Find items similar to a given item

**Request:**

```http
GET /v1/items/movie_456/similar?limit=10 HTTP/1.1
Host: api.recommendations.com
Authorization: Bearer token
```

**Response (200 OK):**

```json
{
  "source_item": {
    "item_id": "movie_456",
    "title": "Inception"
  },
  "similar_items": [
    {
      "item_id": "movie_789",
      "similarity_score": 0.89,
      "metadata": {
        "title": "Interstellar",
        "category": "Sci-Fi"
      },
      "similarity_reason": "Same director, similar themes"
    }
  ],
  "request_id": "req_abc"
}
```

---

### 3. Record User Interaction

**Endpoint:** `POST /v1/interactions`

**Purpose:** Record user interactions for real-time personalization

**Request:**

```http
POST /v1/interactions HTTP/1.1
Host: api.recommendations.com
Content-Type: application/json

{
  "user_id": "user_123",
  "item_id": "movie_456",
  "action": "click",
  "context": {
    "source": "homepage",
    "position": 3,
    "session_id": "session_abc"
  },
  "timestamp": "2024-01-20T15:30:00Z"
}
```

**Action Types:**

| Action     | Weight | Description                    |
| ---------- | ------ | ------------------------------ |
| impression | 0.1    | Item shown to user             |
| click      | 1.0    | User clicked on item           |
| add_to_cart| 2.0    | Added to cart (e-commerce)     |
| purchase   | 5.0    | Purchased item                 |
| watch      | 3.0    | Watched content (streaming)    |
| like       | 2.0    | Explicit positive signal       |
| dislike    | -3.0   | Explicit negative signal       |

**Response (202 Accepted):**

```json
{
  "status": "accepted",
  "event_id": "evt_123"
}
```

---

### 4. Get Recommendation Explanation

**Endpoint:** `GET /v1/recommendations/{item_id}/explain`

**Purpose:** Get detailed explanation for why an item was recommended

**Request:**

```http
GET /v1/recommendations/movie_456/explain?user_id=user_123 HTTP/1.1
Host: api.recommendations.com
```

**Response:**

```json
{
  "item_id": "movie_456",
  "user_id": "user_123",
  "explanation": {
    "primary_reason": "collaborative_filtering",
    "contributing_factors": [
      {
        "factor": "similar_users",
        "weight": 0.45,
        "description": "Users with similar taste enjoyed this"
      },
      {
        "factor": "genre_preference",
        "weight": 0.30,
        "description": "You often watch Sci-Fi movies"
      },
      {
        "factor": "recency",
        "weight": 0.15,
        "description": "Recently released"
      },
      {
        "factor": "popularity",
        "weight": 0.10,
        "description": "Trending this week"
      }
    ],
    "similar_items_watched": ["movie_123", "movie_234"],
    "predicted_rating": 4.5
  }
}
```

---

### 5. User Feedback

**Endpoint:** `POST /v1/feedback`

**Purpose:** Record explicit user feedback on recommendations

**Request:**

```http
POST /v1/feedback HTTP/1.1
Host: api.recommendations.com
Content-Type: application/json

{
  "user_id": "user_123",
  "item_id": "movie_456",
  "feedback_type": "not_interested",
  "reason": "already_seen"
}
```

**Response:**

```json
{
  "status": "recorded",
  "action_taken": "item_excluded_from_future_recommendations"
}
```

---

## Database Schema Design

### Database Choices

| Data Type           | Database       | Rationale                                |
| ------------------- | -------------- | ---------------------------------------- |
| User profiles       | PostgreSQL     | ACID, complex queries                    |
| Item catalog        | PostgreSQL     | Relational data, metadata                |
| Interactions        | Cassandra      | High write volume, time-series           |
| User embeddings     | Redis + FAISS  | Fast retrieval, ANN search               |
| Item embeddings     | FAISS          | Approximate nearest neighbor             |
| Feature store       | Redis          | Real-time feature serving                |

---

### Users Table

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(50) UNIQUE NOT NULL,
    
    -- Profile
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_active_at TIMESTAMP WITH TIME ZONE,
    
    -- Preferences (explicit)
    preferred_categories TEXT[],
    language VARCHAR(10),
    country VARCHAR(3),
    
    -- Computed features
    engagement_level VARCHAR(20),  -- high, medium, low
    total_interactions INTEGER DEFAULT 0,
    
    -- Cold start status
    is_new_user BOOLEAN DEFAULT TRUE,
    onboarding_completed BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_users_active ON users(last_active_at DESC);
CREATE INDEX idx_users_engagement ON users(engagement_level);
```

---

### Items Table

```sql
CREATE TABLE items (
    id BIGSERIAL PRIMARY KEY,
    item_id VARCHAR(50) UNIQUE NOT NULL,
    
    -- Metadata
    title VARCHAR(500) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    subcategory VARCHAR(100),
    tags TEXT[],
    
    -- Content features
    content_features JSONB,  -- Extracted features
    
    -- Popularity metrics
    view_count BIGINT DEFAULT 0,
    interaction_count BIGINT DEFAULT 0,
    avg_rating DECIMAL(3, 2),
    
    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Status
    is_active BOOLEAN DEFAULT TRUE,
    is_new_item BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_items_category ON items(category, subcategory);
CREATE INDEX idx_items_popularity ON items(interaction_count DESC);
CREATE INDEX idx_items_new ON items(created_at DESC) WHERE is_new_item = TRUE;
```

---

### Interactions Table (Cassandra)

```sql
-- Partition by user_id for efficient user history queries
CREATE TABLE interactions (
    user_id TEXT,
    event_time TIMESTAMP,
    item_id TEXT,
    action_type TEXT,
    context MAP<TEXT, TEXT>,
    session_id TEXT,
    PRIMARY KEY (user_id, event_time, item_id)
) WITH CLUSTERING ORDER BY (event_time DESC, item_id ASC);

-- For item-based queries
CREATE TABLE item_interactions (
    item_id TEXT,
    event_time TIMESTAMP,
    user_id TEXT,
    action_type TEXT,
    PRIMARY KEY (item_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

---

### User Embeddings (Redis)

```
# User embedding vector (128 dimensions)
HSET user_embedding:{user_id} 
    vector <binary_vector>
    version "v2.3.1"
    updated_at "2024-01-20T00:00:00Z"

# User features for ranking
HSET user_features:{user_id}
    avg_session_length 15.5
    preferred_categories "sci-fi,action,drama"
    engagement_score 0.75
    last_interaction_time 1705764600
```

---

### Item Embeddings (FAISS Index)

```python
# FAISS index structure
index = faiss.IndexIVFFlat(
    quantizer,           # Coarse quantizer
    embedding_dim,       # 128 dimensions
    n_clusters,          # 1000 clusters
    faiss.METRIC_INNER_PRODUCT
)

# Mapping: FAISS index position -> item_id
item_id_mapping = {
    0: "movie_001",
    1: "movie_002",
    ...
}
```

---

### Pre-computed Recommendations

```sql
-- Store pre-computed recommendations for batch serving
CREATE TABLE precomputed_recommendations (
    user_id VARCHAR(50),
    recommendation_type VARCHAR(50),  -- homepage, similar, etc.
    item_id VARCHAR(50),
    rank INTEGER,
    score FLOAT,
    explanation_type VARCHAR(50),
    computed_at TIMESTAMP WITH TIME ZONE,
    expires_at TIMESTAMP WITH TIME ZONE,
    PRIMARY KEY (user_id, recommendation_type, rank)
);

CREATE INDEX idx_precomputed_expiry ON precomputed_recommendations(expires_at);
```

---

## Entity Relationship Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│       users         │       │       items         │
├─────────────────────┤       ├─────────────────────┤
│ id (PK)             │       │ id (PK)             │
│ user_id (unique)    │       │ item_id (unique)    │
│ preferred_categories│       │ title               │
│ engagement_level    │       │ category            │
│ is_new_user         │       │ content_features    │
└─────────────────────┘       │ is_new_item         │
         │                    └─────────────────────┘
         │                             │
         │    ┌────────────────────────┘
         │    │
         ▼    ▼
┌─────────────────────────────────────────────────────┐
│                   interactions                       │
│              (Cassandra - Time Series)              │
├─────────────────────────────────────────────────────┤
│ user_id (PK)                                        │
│ event_time (CK)                                     │
│ item_id (CK)                                        │
│ action_type                                         │
│ context                                             │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  EMBEDDING STORES                    │
├─────────────────────────────────────────────────────┤
│ Redis: user_embedding:{user_id} → vector            │
│ Redis: user_features:{user_id} → feature hash       │
│ FAISS: item_embeddings index                        │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│              precomputed_recommendations             │
├─────────────────────────────────────────────────────┤
│ user_id (PK)                                        │
│ recommendation_type (PK)                            │
│ rank (PK)                                           │
│ item_id                                             │
│ score                                               │
│ expires_at                                          │
└─────────────────────────────────────────────────────┘
```

---

## Feature Store Schema

### User Features

```json
{
  "user_id": "user_123",
  "features": {
    "demographic": {
      "age_bucket": "25-34",
      "country": "US",
      "language": "en"
    },
    "behavioral": {
      "avg_session_length_minutes": 15.5,
      "sessions_per_week": 5,
      "preferred_time_of_day": "evening",
      "device_type": "mobile"
    },
    "preference": {
      "top_categories": ["sci-fi", "action", "drama"],
      "avg_rating_given": 4.2,
      "completion_rate": 0.75
    },
    "recency": {
      "days_since_last_interaction": 1,
      "interactions_last_7_days": 45,
      "interactions_last_30_days": 180
    }
  },
  "embedding": [0.12, -0.34, 0.56, ...],  // 128 dimensions
  "version": "v2.3.1",
  "updated_at": "2024-01-20T00:00:00Z"
}
```

### Item Features

```json
{
  "item_id": "movie_456",
  "features": {
    "content": {
      "category": "sci-fi",
      "subcategories": ["thriller", "action"],
      "duration_minutes": 148,
      "release_year": 2010,
      "director": "Christopher Nolan",
      "cast": ["Leonardo DiCaprio", "Joseph Gordon-Levitt"]
    },
    "popularity": {
      "total_views": 15000000,
      "views_last_7_days": 50000,
      "avg_rating": 4.8,
      "rating_count": 2500000
    },
    "engagement": {
      "completion_rate": 0.92,
      "rewatch_rate": 0.15,
      "share_rate": 0.05
    }
  },
  "embedding": [0.23, 0.45, -0.12, ...],  // 128 dimensions
  "version": "v2.3.1"
}
```

---

## Idempotency Model

### What is Idempotency?

An operation is **idempotent** if performing it multiple times has the same effect as performing it once. This is critical for recommendation systems to prevent duplicate interaction tracking and ensure data consistency.

### Idempotent Operations in Recommendation System

| Operation | Idempotent? | Mechanism |
|-----------|-------------|-----------|
| **Record Interaction** | ✅ Yes | Event ID prevents duplicate tracking |
| **Update User Preferences** | ✅ Yes | Idempotent state update |
| **Get Recommendations** | ✅ Yes | Read-only, no side effects |
| **Update Item Features** | ✅ Yes | Version-based updates prevent conflicts |
| **A/B Test Assignment** | ⚠️ At-least-once | Client-side deduplication by user_id + experiment_id |

### Idempotency Implementation

**1. Record Interaction with Event ID:**

```java
@PostMapping("/v1/interactions")
public ResponseEntity<InteractionResponse> recordInteraction(
        @RequestBody RecordInteractionRequest request,
        @RequestHeader("X-Event-ID") String eventId) {
    
    // Check if interaction already recorded
    Interaction existing = interactionRepository.findByEventId(eventId);
    if (existing != null) {
        return ResponseEntity.ok(InteractionResponse.from(existing));
    }
    
    // Create interaction record
    Interaction interaction = Interaction.builder()
        .eventId(eventId)
        .userId(request.getUserId())
        .itemId(request.getItemId())
        .interactionType(request.getType()) // click, view, like, purchase
        .context(request.getContext()) // homepage, search, detail_page
        .timestamp(Instant.now())
        .metadata(request.getMetadata())
        .build();
    
    interaction = interactionRepository.save(interaction);
    
    // Publish to Kafka for async processing (idempotent via event_id)
    kafkaTemplate.send("interaction-events", 
        request.getUserId(), 
        InteractionEvent.from(interaction)
    );
    
    return ResponseEntity.status(201).body(InteractionResponse.from(interaction));
}
```

**2. User Preference Update Idempotency:**

```java
@PutMapping("/v1/users/{userId}/preferences")
public ResponseEntity<UserPreferencesResponse> updatePreferences(
        @PathVariable String userId,
        @RequestBody UpdatePreferencesRequest request,
        @RequestHeader("X-Idempotency-Key") String idempotencyKey) {
    
    // Check if update already processed
    UserPreferences existing = preferencesRepository.findByUserIdAndIdempotencyKey(
        userId, idempotencyKey
    );
    if (existing != null) {
        return ResponseEntity.ok(UserPreferencesResponse.from(existing));
    }
    
    // Update preferences (idempotent state update)
    UserPreferences preferences = preferencesRepository.findByUserId(userId)
        .orElse(UserPreferences.builder().userId(userId).build());
    
    preferences.setFavoriteCategories(request.getFavoriteCategories());
    preferences.setExcludedItems(request.getExcludedItems());
    preferences.setIdempotencyKey(idempotencyKey);
    preferences.setUpdatedAt(Instant.now());
    
    preferences = preferencesRepository.save(preferences);
    
    // Invalidate user embedding cache (will be recomputed)
    redisTemplate.delete("user_embedding:" + userId);
    
    return ResponseEntity.ok(UserPreferencesResponse.from(preferences));
}
```

**3. A/B Test Assignment Deduplication:**

```java
// Deduplicate A/B test assignments by (user_id, experiment_id)
public String assignExperiment(String userId, String experimentId) {
    String dedupKey = String.format("ab_test:%s:%s", userId, experimentId);
    
    // Try to get existing assignment
    String existing = (String) redisTemplate.opsForValue().get(dedupKey);
    if (existing != null) {
        return existing; // Already assigned
    }
    
    // Assign variant (consistent hashing ensures same user gets same variant)
    String variant = assignVariant(userId, experimentId);
    
    // Store assignment (24 hour TTL)
    redisTemplate.opsForValue().set(dedupKey, variant, Duration.ofHours(24));
    
    return variant;
}
```

**4. Item Feature Update Idempotency:**

```java
@PutMapping("/v1/items/{itemId}/features")
public ResponseEntity<ItemFeaturesResponse> updateItemFeatures(
        @PathVariable String itemId,
        @RequestBody UpdateItemFeaturesRequest request,
        @RequestHeader("X-Idempotency-Key") String idempotencyKey) {
    
    // Check if update already processed
    ItemFeatures existing = itemFeaturesRepository.findByItemIdAndIdempotencyKey(
        itemId, idempotencyKey
    );
    if (existing != null) {
        return ResponseEntity.ok(ItemFeaturesResponse.from(existing));
    }
    
    // Update item features (version-based for conflict resolution)
    ItemFeatures features = itemFeaturesRepository.findByItemId(itemId)
        .orElseThrow(() -> new ItemNotFoundException(itemId));
    
    features.setContentFeatures(request.getContentFeatures());
    features.setPopularityFeatures(request.getPopularityFeatures());
    features.setIdempotencyKey(idempotencyKey);
    features.setVersion(features.getVersion() + 1);
    features.setUpdatedAt(Instant.now());
    
    features = itemFeaturesRepository.save(features);
    
    // Invalidate item embedding cache
    redisTemplate.delete("item_embedding:" + itemId);
    
    return ResponseEntity.ok(ItemFeaturesResponse.from(features));
}
```

### Idempotency Key Generation

**Client-Side (Recommended):**
```javascript
// Generate UUID v4 for each interaction
const eventId = crypto.randomUUID();

fetch('/v1/interactions', {
    method: 'POST',
    headers: {
        'Authorization': `Bearer ${token}`,
        'X-Event-ID': eventId
    },
    body: interactionData
});
```

**Server-Side (Fallback):**
```java
// If client doesn't provide event ID, generate from request hash
String eventId = DigestUtils.sha256Hex(
    request.getUserId() + 
    request.getItemId() +
    request.getType() +
    request.getTimestamp().toString()
);
```

### Duplicate Detection Window

| Operation | Deduplication Window | Storage |
|-----------|---------------------|---------|
| Record Interaction | 24 hours | PostgreSQL (event_id unique index) |
| Update Preferences | 24 hours | PostgreSQL (idempotency_key unique index) |
| A/B Test Assignment | 24 hours | Redis |
| Update Item Features | 24 hours | PostgreSQL (idempotency_key unique index) |

### Idempotency Key Storage

```sql
-- Add event_id to interactions
ALTER TABLE interactions ADD COLUMN event_id VARCHAR(255);
CREATE UNIQUE INDEX idx_interactions_event_id ON interactions(event_id) 
WHERE event_id IS NOT NULL;

-- Add idempotency_key to user_preferences
ALTER TABLE user_preferences ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_user_preferences_idempotency_key 
ON user_preferences(user_id, idempotency_key) 
WHERE idempotency_key IS NOT NULL;

-- Add idempotency_key to item_features
ALTER TABLE item_features ADD COLUMN idempotency_key VARCHAR(255);
CREATE UNIQUE INDEX idx_item_features_idempotency_key 
ON item_features(item_id, idempotency_key) 
WHERE idempotency_key IS NOT NULL;
```

---

## Summary

| Component           | Technology/Approach                     |
| ------------------- | --------------------------------------- |
| Recommendation API  | REST with context parameters            |
| User data           | PostgreSQL                              |
| Item catalog        | PostgreSQL                              |
| Interactions        | Cassandra (time-series)                 |
| User embeddings     | Redis + FAISS                           |
| Item embeddings     | FAISS (ANN search)                      |
| Feature store       | Redis (real-time serving)               |
| Pre-computed recs   | PostgreSQL with TTL                     |

