# Recommendation System - Production Deep Dives (Core)

## Overview

This document covers the core technical implementation details for the Recommendation System: collaborative filtering, content-based filtering, approximate nearest neighbor search with FAISS, ranking models, and cold start handling. Search is handled as part of the candidate generation process using embedding-based similarity.

---

## 1. Collaborative Filtering Deep Dive

### Matrix Factorization

Matrix factorization learns latent representations (embeddings) for users and items by decomposing the user-item interaction matrix.

```java
@Service
public class MatrixFactorizationService {
    
    private static final int EMBEDDING_DIM = 128;
    private static final int MAX_ITERATIONS = 100;
    private static final double LEARNING_RATE = 0.01;
    private static final double REGULARIZATION = 0.1;
    
    /**
     * Train user and item embeddings using ALS (Alternating Least Squares)
     */
    public EmbeddingModel train(List<Interaction> interactions) {
        // Initialize embeddings randomly
        Map<Long, double[]> userEmbeddings = initializeEmbeddings(getUserIds(interactions));
        Map<Long, double[]> itemEmbeddings = initializeEmbeddings(getItemIds(interactions));
        
        // Build interaction matrix
        Map<Long, Map<Long, Double>> userItemMatrix = buildMatrix(interactions);
        
        for (int iter = 0; iter < MAX_ITERATIONS; iter++) {
            // Fix item embeddings, update user embeddings
            for (Long userId : userEmbeddings.keySet()) {
                userEmbeddings.put(userId, 
                    updateEmbedding(userId, userItemMatrix.get(userId), itemEmbeddings));
            }
            
            // Fix user embeddings, update item embeddings
            Map<Long, Map<Long, Double>> itemUserMatrix = transpose(userItemMatrix);
            for (Long itemId : itemEmbeddings.keySet()) {
                itemEmbeddings.put(itemId,
                    updateEmbedding(itemId, itemUserMatrix.get(itemId), userEmbeddings));
            }
            
            // Calculate loss
            double loss = calculateLoss(userItemMatrix, userEmbeddings, itemEmbeddings);
            log.info("Iteration {}: Loss = {}", iter, loss);
        }
        
        return new EmbeddingModel(userEmbeddings, itemEmbeddings);
    }
    
    private double[] updateEmbedding(Long id, Map<Long, Double> ratings, 
                                      Map<Long, double[]> otherEmbeddings) {
        // Solve: (X^T X + λI) * embedding = X^T * ratings
        // Using gradient descent for simplicity
        
        double[] embedding = new double[EMBEDDING_DIM];
        Arrays.fill(embedding, 0.1);
        
        for (int i = 0; i < 10; i++) {
            double[] gradient = new double[EMBEDDING_DIM];
            
            for (Map.Entry<Long, Double> entry : ratings.entrySet()) {
                double[] otherEmb = otherEmbeddings.get(entry.getKey());
                double predicted = dotProduct(embedding, otherEmb);
                double error = entry.getValue() - predicted;
                
                for (int d = 0; d < EMBEDDING_DIM; d++) {
                    gradient[d] += error * otherEmb[d] - REGULARIZATION * embedding[d];
                }
            }
            
            for (int d = 0; d < EMBEDDING_DIM; d++) {
                embedding[d] += LEARNING_RATE * gradient[d];
            }
        }
        
        return embedding;
    }
}
```

### Two-Tower Model (Neural Collaborative Filtering)

The Two-Tower architecture learns separate neural network representations for users and items, enabling efficient retrieval.

```python
import tensorflow as tf

class TwoTowerModel(tf.keras.Model):
    def __init__(self, num_users, num_items, embedding_dim=128):
        super().__init__()
        
        # User tower
        self.user_embedding = tf.keras.layers.Embedding(num_users, 64)
        self.user_dense1 = tf.keras.layers.Dense(128, activation='relu')
        self.user_dense2 = tf.keras.layers.Dense(embedding_dim)
        
        # Item tower
        self.item_embedding = tf.keras.layers.Embedding(num_items, 64)
        self.item_dense1 = tf.keras.layers.Dense(128, activation='relu')
        self.item_dense2 = tf.keras.layers.Dense(embedding_dim)
    
    def call(self, inputs):
        user_id, item_id = inputs
        
        # User tower
        user_emb = self.user_embedding(user_id)
        user_emb = self.user_dense1(user_emb)
        user_emb = self.user_dense2(user_emb)
        user_emb = tf.nn.l2_normalize(user_emb, axis=1)
        
        # Item tower
        item_emb = self.item_embedding(item_id)
        item_emb = self.item_dense1(item_emb)
        item_emb = self.item_dense2(item_emb)
        item_emb = tf.nn.l2_normalize(item_emb, axis=1)
        
        # Dot product similarity
        similarity = tf.reduce_sum(user_emb * item_emb, axis=1)
        return similarity
    
    def get_user_embedding(self, user_id):
        user_emb = self.user_embedding(user_id)
        user_emb = self.user_dense1(user_emb)
        user_emb = self.user_dense2(user_emb)
        return tf.nn.l2_normalize(user_emb, axis=1)
    
    def get_item_embedding(self, item_id):
        item_emb = self.item_embedding(item_id)
        item_emb = self.item_dense1(item_emb)
        item_emb = self.item_dense2(item_emb)
        return tf.nn.l2_normalize(item_emb, axis=1)
```

---

## 2. Approximate Nearest Neighbor (FAISS)

FAISS (Facebook AI Similarity Search) enables efficient similarity search over millions of vectors.

### Index Building

```java
@Service
public class FAISSIndexService {
    
    private Index index;
    private Map<Integer, String> indexToItemId;
    
    public void buildIndex(Map<String, float[]> itemEmbeddings) {
        int dimension = 128;
        int numItems = itemEmbeddings.size();
        
        // Create IVF index with product quantization
        int nlist = (int) Math.sqrt(numItems);  // Number of clusters
        int m = 8;  // Number of subquantizers
        int nbits = 8;  // Bits per subquantizer
        
        // Quantizer for clustering
        IndexFlatIP quantizer = new IndexFlatIP(dimension);
        
        // IVF-PQ index
        index = new IndexIVFPQ(quantizer, dimension, nlist, m, nbits);
        
        // Prepare training data
        float[] trainingData = new float[numItems * dimension];
        indexToItemId = new HashMap<>();
        
        int i = 0;
        for (Map.Entry<String, float[]> entry : itemEmbeddings.entrySet()) {
            System.arraycopy(entry.getValue(), 0, trainingData, i * dimension, dimension);
            indexToItemId.put(i, entry.getKey());
            i++;
        }
        
        // Train the index
        index.train(numItems, trainingData);
        
        // Add vectors
        index.add(numItems, trainingData);
        
        log.info("Built FAISS index with {} items", numItems);
    }
    
    public List<ScoredItem> search(float[] queryEmbedding, int k, int nprobe) {
        // Set number of clusters to probe
        ((IndexIVFPQ) index).setNprobe(nprobe);
        
        // Search
        long[] indices = new long[k];
        float[] distances = new float[k];
        
        index.search(1, queryEmbedding, k, distances, indices);
        
        // Convert to results
        List<ScoredItem> results = new ArrayList<>();
        for (int i = 0; i < k; i++) {
            if (indices[i] >= 0) {
                results.add(new ScoredItem(
                    indexToItemId.get((int) indices[i]),
                    distances[i]
                ));
            }
        }
        
        return results;
    }
}
```

### Candidate Generation Service

```java
@Service
public class CandidateGenerationService {
    
    private final FAISSIndexService faissIndex;
    private final RedisTemplate<String, byte[]> redis;
    private final PopularityService popularityService;
    
    private static final int COLLABORATIVE_CANDIDATES = 500;
    private static final int CONTENT_CANDIDATES = 300;
    private static final int POPULARITY_CANDIDATES = 150;
    private static final int TRENDING_CANDIDATES = 50;
    
    public List<Candidate> generateCandidates(String userId, Context context) {
        List<Candidate> allCandidates = new ArrayList<>();
        
        // 1. Collaborative filtering (ANN search)
        float[] userEmbedding = getUserEmbedding(userId);
        List<ScoredItem> collaborativeCandidates = faissIndex.search(
            userEmbedding, 
            COLLABORATIVE_CANDIDATES,
            10  // nprobe
        );
        allCandidates.addAll(toCandidates(collaborativeCandidates, "collaborative"));
        
        // 2. Content-based (based on recent interactions)
        List<String> recentItems = getRecentInteractions(userId, 10);
        for (String itemId : recentItems) {
            float[] itemEmbedding = getItemEmbedding(itemId);
            List<ScoredItem> similar = faissIndex.search(
                itemEmbedding,
                CONTENT_CANDIDATES / recentItems.size(),
                5
            );
            allCandidates.addAll(toCandidates(similar, "content_based"));
        }
        
        // 3. Popularity-based
        List<String> popular = popularityService.getPopular(
            context.getCategory(),
            POPULARITY_CANDIDATES
        );
        allCandidates.addAll(toCandidates(popular, "popularity", 0.5));
        
        // 4. Trending
        List<String> trending = popularityService.getTrending(TRENDING_CANDIDATES);
        allCandidates.addAll(toCandidates(trending, "trending", 0.4));
        
        // Deduplicate and merge scores
        return deduplicateAndMerge(allCandidates);
    }
    
    private float[] getUserEmbedding(String userId) {
        byte[] cached = redis.opsForValue().get("user_embedding:" + userId);
        if (cached != null) {
            return deserializeEmbedding(cached);
        }
        
        // Fallback: compute on-the-fly or use average embedding
        return computeUserEmbedding(userId);
    }
}
```

### C) REAL STEP-BY-STEP SIMULATION: FAISS Technology-Level

**Normal Flow: FAISS ANN Search**

```
Step 1: User Request
┌─────────────────────────────────────────────────────────────┐
│ GET /v1/recommendations?user_id=user_123&limit=20          │
│ User ID: user_123                                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: Get User Embedding
┌─────────────────────────────────────────────────────────────┐
│ Redis: GET user_embedding:user_123                          │
│ Returns: [0.23, -0.45, 0.12, ..., 0.67] (128 dimensions)     │
│ Latency: ~1ms                                                │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: FAISS Index Search
┌─────────────────────────────────────────────────────────────┐
│ FAISS Index: IVFPQ (Inverted File with Product Quantization)│
│ - Total vectors: 10M items                                  │
│ - Clusters: 1,024 (centroids)                               │
│ - nprobe: 10 (search 10 nearest clusters)                  │
│                                                              │
│ Process:                                                    │
│ 1. Find 10 nearest cluster centroids to query vector       │
│ 2. Load vectors from those 10 clusters (~100K vectors)     │
│ 3. Compute inner product with query vector                 │
│ 4. Return top 500 by similarity                             │
│                                                              │
│ Result: [(movie_456, 0.95), (movie_789, 0.92), ...]        │
│ Latency: ~5ms                                                │
└─────────────────────────────────────────────────────────────┘
```

**Failure Flow: FAISS Index Corruption**

```
Scenario: Index file corrupted after nightly update

┌─────────────────────────────────────────────────────────────┐
│ T+0s: Nightly index update completes                       │
│ T+1s: New index file written to disk                       │
│ T+2s: Index file corrupted (disk error, network issue)     │
│ T+3s: Service tries to load index → FAILS                  │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ T+4s: Health check fails                                    │
│ T+5s: Alert triggered: "FAISS index load failed"          │
│ T+6s: Service falls back to previous index version        │
│ T+7s: Service resumes with stale index                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ Impact:                                                      │
│ - Recommendations may be stale (missing new items)         │
│ - System continues operating (graceful degradation)        │
│ - New index rebuilt and deployed within 1 hour             │
│ - No user-facing errors                                     │
└─────────────────────────────────────────────────────────────┘
```

**Idempotency Handling:**

```
Problem: Same query processed multiple times (retry)

Solution: Results are deterministic
- Same user embedding → Same FAISS search results
- Results cached in Redis (5-minute TTL)
- Duplicate queries return cached results (idempotent)

Example:
- Query processed twice → Same FAISS results → Cached → Idempotent
```

**Traffic Spike Handling:**

```
Scenario: Flash sale triggers millions of recommendation requests

┌─────────────────────────────────────────────────────────────┐
│ Normal: 10K requests/second                                 │
│ Spike: 100K requests/second (flash sale)                    │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ FAISS Handling:                                             │
│ - FAISS is read-only (no locks)                            │
│ - Multiple threads can search simultaneously               │
│ - Memory-mapped index (shared across processes)            │
│ - Result: 100K searches/second handled                      │
│                                                              │
│ Redis Caching:                                              │
│ - Cache hit rate: 80% (popular users)                      │
│ - Cache miss: 20K requests/second → FAISS                   │
│ - Result: Most requests served from cache                  │
└─────────────────────────────────────────────────────────────┘
```

**Hot Query Mitigation:**

```
Problem: Popular user's recommendations requested frequently

Mitigation:
1. Redis caching: Cache recommendations for 5 minutes
2. Pre-computation: Pre-compute recommendations for popular users
3. CDN: Cache recommendations at edge (if applicable)

If hot query occurs:
- Monitor cache hit rate
- Increase cache TTL for popular users
- Pre-compute recommendations offline
```

**Recovery Behavior:**

```
Auto-healing:
- Index corruption → Fallback to previous version → Automatic
- Index update failure → Keep current index → Automatic
- Service restart → Load index from disk → Automatic

Human intervention:
- Persistent index corruption → Rebuild index from embeddings
- Performance degradation → Optimize index parameters (nprobe, clusters)
- Memory issues → Reduce index size or use disk-based index
```

### Cache Stampede Prevention (Redis Recommendations Cache)

**What is Cache Stampede?**

Cache stampede occurs when a cached recommendation result expires and many requests simultaneously try to regenerate it, overwhelming the recommendation service.

**Scenario:**
```
T+0s:   Popular user's recommendation cache expires
T+0s:   10,000 requests arrive simultaneously
T+0s:   All 10,000 requests see cache MISS
T+0s:   All 10,000 requests hit FAISS/index simultaneously
T+1s:   Recommendation service overwhelmed, latency spikes
```

**Prevention Strategy: Distributed Lock + Double-Check Pattern**

This system uses distributed locking for Redis recommendation cache:

1. **Distributed locking**: Only one thread fetches from FAISS/index on cache miss
2. **Double-check pattern**: Re-check cache after acquiring lock
3. **TTL jitter**: Add random jitter to TTL to prevent simultaneous expiration

**Implementation:**

```java
@Service
public class RecommendationCacheService {
    
    private final RedisTemplate<String, List<Recommendation>> redis;
    private final DistributedLock lockService;
    
    public List<Recommendation> getCachedRecommendations(String userId) {
        String key = "recs:" + userId;
        
        // 1. Try cache first
        List<Recommendation> cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // 2. Cache miss: Try to acquire lock
        String lockKey = "lock:recs:" + userId;
        boolean acquired = lockService.tryLock(lockKey, Duration.ofSeconds(5));
        
        if (!acquired) {
            // Another thread is fetching, wait briefly
            try {
                Thread.sleep(50);
                cached = redis.opsForValue().get(key);
                if (cached != null) {
                    return cached;
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            // Fall back to generating recommendations (acceptable degradation)
        }
        
        try {
            // 3. Double-check cache (might have been populated while acquiring lock)
            cached = redis.opsForValue().get(key);
            if (cached != null) {
                return cached;
            }
            
            // 4. Generate recommendations (only one thread does this)
            List<Recommendation> recommendations = generateRecommendations(userId);
            
            // 5. Populate cache with TTL jitter (add ±10% random jitter to 5min TTL)
            Duration baseTtl = Duration.ofMinutes(5);
            long jitter = (long)(baseTtl.getSeconds() * 0.1 * (Math.random() * 2 - 1));
            Duration ttl = baseTtl.plusSeconds(jitter);
            
            redis.opsForValue().set(key, recommendations, ttl);
            
            return recommendations;
            
        } finally {
            if (acquired) {
                lockService.unlock(lockKey);
            }
        }
    }
}
```

**TTL Jitter Benefits:**

- Prevents simultaneous expiration of related cache entries
- Reduces cache stampede probability
- Smooths out cache refresh load over time

### Step-by-Step Simulation: Candidate Generation

**Scenario**: Generate candidates for user_123 requesting movie recommendations

```
Step 1: Get User Embedding
┌─────────────────────────────────────────────────────────────┐
│ Query Redis: user_embedding:user_123                         │
│                                                              │
│ Result: [0.23, -0.45, 0.12, ..., 0.67] (128 dimensions)     │
│ Latency: 1ms                                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 2: FAISS ANN Search (Collaborative Filtering)
┌─────────────────────────────────────────────────────────────┐
│ Query: user_embedding                                        │
│ k: 500 candidates                                            │
│ nprobe: 10 clusters                                          │
│                                                              │
│ Process:                                                     │
│ 1. Find 10 nearest clusters to query vector                 │
│ 2. Search within those clusters (~10,000 vectors)           │
│ 3. Return top 500 by inner product similarity               │
│                                                              │
│ Result: [(movie_456, 0.95), (movie_789, 0.92), ...]        │
│ Latency: 5ms                                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 3: Content-Based Candidates
┌─────────────────────────────────────────────────────────────┐
│ Get recent interactions: [movie_123, movie_234, ...]        │
│                                                              │
│ For each recent item:                                        │
│ - Get item embedding                                         │
│ - FAISS search for similar items                            │
│ - Add 30 candidates per item                                 │
│                                                              │
│ Result: 300 content-based candidates                        │
│ Latency: 15ms (10 searches × 1.5ms)                         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 4: Popularity & Trending
┌─────────────────────────────────────────────────────────────┐
│ Popular in Sci-Fi: 150 items from Redis sorted set          │
│ Trending this week: 50 items                                │
│                                                              │
│ Latency: 2ms                                                 │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
Step 5: Merge & Deduplicate
┌─────────────────────────────────────────────────────────────┐
│ Total candidates: 500 + 300 + 150 + 50 = 1000               │
│ After deduplication: ~850 unique items                      │
│                                                              │
│ Merge strategy:                                              │
│ - If item appears in multiple sources, keep highest score   │
│ - Tag with source for explainability                        │
│                                                              │
│ Result: 1000 candidates ready for ranking                   │
│ Total latency: ~25ms                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Ranking Model Deep Dive

### Wide & Deep Model

The Wide & Deep architecture combines memorization (wide) with generalization (deep).

```python
import tensorflow as tf

class WideAndDeepModel(tf.keras.Model):
    def __init__(self, feature_columns, deep_hidden_units=[256, 128, 64]):
        super().__init__()
        
        # Wide component (linear model for memorization)
        self.wide_features = ['user_category_cross', 'item_category', 
                              'user_engagement_bucket']
        self.wide_layer = tf.keras.layers.Dense(1, activation=None)
        
        # Deep component (DNN for generalization)
        self.deep_features = ['user_embedding', 'item_embedding', 
                              'user_features', 'item_features', 'context_features']
        
        self.deep_layers = []
        for units in deep_hidden_units:
            self.deep_layers.append(tf.keras.layers.Dense(units, activation='relu'))
            self.deep_layers.append(tf.keras.layers.Dropout(0.2))
        
        self.deep_output = tf.keras.layers.Dense(1, activation=None)
        
        # Final combination
        self.final_layer = tf.keras.layers.Dense(1, activation='sigmoid')
    
    def call(self, features, training=False):
        # Wide path
        wide_input = tf.concat([features[f] for f in self.wide_features], axis=1)
        wide_output = self.wide_layer(wide_input)
        
        # Deep path
        deep_input = tf.concat([features[f] for f in self.deep_features], axis=1)
        x = deep_input
        for layer in self.deep_layers:
            x = layer(x, training=training)
        deep_output = self.deep_output(x)
        
        # Combine
        combined = tf.concat([wide_output, deep_output], axis=1)
        return self.final_layer(combined)
```

### Feature Engineering

```java
@Service
public class FeatureEngineeringService {
    
    public RankingFeatures buildFeatures(String userId, String itemId, Context context) {
        UserFeatures userFeatures = getUserFeatures(userId);
        ItemFeatures itemFeatures = getItemFeatures(itemId);
        
        return RankingFeatures.builder()
            // User features
            .userEmbedding(userFeatures.getEmbedding())
            .userEngagementScore(userFeatures.getEngagementScore())
            .userPreferredCategories(encodeCategories(userFeatures.getPreferredCategories()))
            .userActivityLevel(encodeActivityLevel(userFeatures.getActivityLevel()))
            .daysSinceLastInteraction(userFeatures.getDaysSinceLastInteraction())
            
            // Item features
            .itemEmbedding(itemFeatures.getEmbedding())
            .itemCategory(encodeCategory(itemFeatures.getCategory()))
            .itemPopularity(normalizePopularity(itemFeatures.getViewCount()))
            .itemFreshness(calculateFreshness(itemFeatures.getCreatedAt()))
            .itemAvgRating(itemFeatures.getAvgRating())
            
            // Cross features
            .userItemCategoryCross(crossEncode(
                userFeatures.getPreferredCategories(), 
                itemFeatures.getCategory()
            ))
            .userItemInteractionHistory(getInteractionHistory(userId, itemId))
            
            // Context features
            .timeOfDay(encodeTimeOfDay(context.getTimestamp()))
            .dayOfWeek(encodeDayOfWeek(context.getTimestamp()))
            .deviceType(encodeDevice(context.getDeviceType()))
            .sessionDepth(context.getSessionDepth())
            
            .build();
    }
    
    private double calculateFreshness(Instant createdAt) {
        long ageHours = ChronoUnit.HOURS.between(createdAt, Instant.now());
        // Exponential decay with 7-day half-life
        return Math.exp(-ageHours / (7 * 24.0));
    }
    
    private double[] crossEncode(List<String> userCategories, String itemCategory) {
        // One-hot encode the cross of user preferences and item category
        double[] cross = new double[100];  // Assuming 100 possible crosses
        
        for (String userCat : userCategories) {
            int index = hashCross(userCat, itemCategory) % 100;
            cross[index] = 1.0;
        }
        
        return cross;
    }
}
```

### Ranking Service

```java
@Service
public class RankingService {
    
    private final FeatureEngineeringService featureService;
    private final ModelServingClient modelClient;
    
    public List<RankedItem> rank(List<Candidate> candidates, String userId, Context context) {
        // Build features for all candidates
        List<RankingFeatures> featuresList = candidates.parallelStream()
            .map(c -> featureService.buildFeatures(userId, c.getItemId(), context))
            .collect(Collectors.toList());
        
        // Batch inference
        float[] scores = modelClient.predict(featuresList);
        
        // Combine with candidates
        List<RankedItem> rankedItems = new ArrayList<>();
        for (int i = 0; i < candidates.size(); i++) {
            rankedItems.add(RankedItem.builder()
                .itemId(candidates.get(i).getItemId())
                .score(scores[i])
                .candidateSource(candidates.get(i).getSource())
                .build());
        }
        
        // Sort by score
        rankedItems.sort(Comparator.comparingDouble(RankedItem::getScore).reversed());
        
        // Apply diversity
        return applyDiversity(rankedItems);
    }
    
    private List<RankedItem> applyDiversity(List<RankedItem> items) {
        List<RankedItem> diversified = new ArrayList<>();
        Map<String, Integer> categoryCount = new HashMap<>();
        
        for (RankedItem item : items) {
            String category = getCategory(item.getItemId());
            int count = categoryCount.getOrDefault(category, 0);
            
            // Max 3 items from same category in top 20
            if (diversified.size() < 20 && count >= 3) {
                continue;
            }
            
            diversified.add(item);
            categoryCount.put(category, count + 1);
            
            if (diversified.size() >= items.size()) {
                break;
            }
        }
        
        return diversified;
    }
}
```

---

## 4. Cold Start Handling

### New User Strategy

```java
@Service
public class ColdStartService {
    
    private final PopularityService popularityService;
    private final OnboardingService onboardingService;
    
    public List<Candidate> handleNewUser(String userId, Context context) {
        User user = userRepository.findById(userId);
        
        if (user.hasCompletedOnboarding()) {
            // Use onboarding preferences
            return generateFromOnboarding(user);
        }
        
        // Use demographic-based recommendations
        if (user.hasDemographicData()) {
            return generateFromDemographics(user);
        }
        
        // Fallback to popularity
        return generatePopularItems(context);
    }
    
    private List<Candidate> generateFromOnboarding(User user) {
        List<String> selectedCategories = onboardingService.getSelectedCategories(user.getId());
        List<String> selectedItems = onboardingService.getSelectedItems(user.getId());
        
        List<Candidate> candidates = new ArrayList<>();
        
        // Popular in selected categories
        for (String category : selectedCategories) {
            candidates.addAll(
                popularityService.getPopularInCategory(category, 50)
                    .stream()
                    .map(itemId -> new Candidate(itemId, "onboarding_category", 0.8))
                    .collect(Collectors.toList())
            );
        }
        
        // Similar to selected items
        for (String itemId : selectedItems) {
            candidates.addAll(
                similarItemService.getSimilar(itemId, 20)
                    .stream()
                    .map(similar -> new Candidate(similar, "onboarding_similar", 0.9))
                    .collect(Collectors.toList())
            );
        }
        
        return deduplicateAndMerge(candidates);
    }
    
    private List<Candidate> generateFromDemographics(User user) {
        // Find users with similar demographics
        List<String> similarUsers = findSimilarDemographicUsers(user);
        
        // Get popular items among similar users
        return getPopularAmongUsers(similarUsers, 200);
    }
}
```

### New Item Strategy

```java
@Service
public class NewItemService {
    
    public void handleNewItem(Item item) {
        // 1. Generate content-based embedding
        float[] contentEmbedding = generateContentEmbedding(item);
        
        // 2. Store embedding for ANN search
        embeddingStore.save(item.getId(), contentEmbedding);
        
        // 3. Add to exploration pool
        explorationPool.add(item.getId(), ExplorationConfig.builder()
            .initialExposure(1000)  // Show to 1000 users
            .decayRate(0.9)
            .minExposure(100)
            .build());
        
        // 4. Find similar existing items
        List<String> similarItems = faissIndex.search(contentEmbedding, 100);
        
        // 5. Target users who liked similar items
        for (String similarItemId : similarItems) {
            List<String> interestedUsers = getInterestedUsers(similarItemId);
            for (String userId : interestedUsers) {
                explorationQueue.add(userId, item.getId());
            }
        }
    }
    
    private float[] generateContentEmbedding(Item item) {
        // Use pre-trained model to generate embedding from item attributes
        ContentFeatures features = ContentFeatures.builder()
            .title(item.getTitle())
            .description(item.getDescription())
            .category(item.getCategory())
            .tags(item.getTags())
            .build();
        
        return contentEncoder.encode(features);
    }
}
```

---

## 5. Real-time Feature Updates

### Feature Update Pipeline

```java
@Service
public class RealTimeFeatureUpdater {
    
    private final RedisTemplate<String, String> redis;
    
    @KafkaListener(topics = "interaction-events")
    public void handleInteraction(InteractionEvent event) {
        String userId = event.getUserId();
        String itemId = event.getItemId();
        String action = event.getAction();
        
        // Update user features
        updateUserFeatures(userId, itemId, action);
        
        // Update item features
        updateItemFeatures(itemId, action);
        
        // Update user-item interaction features
        updateInteractionFeatures(userId, itemId, action);
    }
    
    private void updateUserFeatures(String userId, String itemId, String action) {
        String key = "user_features:" + userId;
        
        // Increment interaction count
        redis.opsForHash().increment(key, "total_interactions", 1);
        
        // Update last interaction time
        redis.opsForHash().put(key, "last_interaction_time", 
            String.valueOf(System.currentTimeMillis()));
        
        // Update category preferences
        String category = getItemCategory(itemId);
        redis.opsForHash().increment(key, "category:" + category, 
            getActionWeight(action));
        
        // Update session features
        String sessionKey = "session:" + userId;
        redis.opsForList().leftPush(sessionKey, itemId);
        redis.opsForList().trim(sessionKey, 0, 99);  // Keep last 100
        redis.expire(sessionKey, 30, TimeUnit.MINUTES);
    }
    
    private void updateItemFeatures(String itemId, String action) {
        String key = "item_features:" + itemId;
        
        // Increment view/click/purchase count
        redis.opsForHash().increment(key, action + "_count", 1);
        
        // Update trending score (time-decayed)
        String trendingKey = "trending:" + itemId;
        double score = getActionWeight(action);
        redis.opsForZSet().incrementScore("trending_items", itemId, score);
    }
    
    private double getActionWeight(String action) {
        switch (action) {
            case "view": return 0.1;
            case "click": return 1.0;
            case "add_to_cart": return 2.0;
            case "purchase": return 5.0;
            default: return 0.5;
        }
    }
}
```

---

## 6. Search (Embedding-Based Retrieval)

For recommendation systems, "search" is implemented as embedding-based retrieval during candidate generation. Unlike traditional text search with inverted indexes, recommendations use vector similarity.

### How Search Works in Recommendations

```java
@Service
public class RecommendationSearchService {
    
    private final FAISSIndexService faissIndex;
    
    /**
     * Search for items similar to a query (can be user embedding or item embedding)
     */
    public List<ScoredItem> search(float[] queryEmbedding, SearchConfig config) {
        // FAISS handles the "search" via approximate nearest neighbor
        return faissIndex.search(
            queryEmbedding,
            config.getNumResults(),
            config.getNprobe()
        );
    }
    
    /**
     * Search for items similar to a text query (for hybrid search)
     */
    public List<ScoredItem> searchByText(String query, String userId) {
        // 1. Convert text query to embedding using sentence transformer
        float[] queryEmbedding = textEncoder.encode(query);
        
        // 2. Get user embedding for personalization
        float[] userEmbedding = getUserEmbedding(userId);
        
        // 3. Combine query and user embeddings
        float[] combinedEmbedding = combineEmbeddings(queryEmbedding, userEmbedding, 0.7);
        
        // 4. Search FAISS index
        return faissIndex.search(combinedEmbedding, 100, 10);
    }
    
    private float[] combineEmbeddings(float[] query, float[] user, double queryWeight) {
        float[] combined = new float[query.length];
        for (int i = 0; i < query.length; i++) {
            combined[i] = (float) (queryWeight * query[i] + (1 - queryWeight) * user[i]);
        }
        return normalize(combined);
    }
}
```

### Key Differences from Traditional Search

| Aspect              | Traditional Search (Inverted Index) | Recommendation Search (Embeddings) |
| ------------------- | ----------------------------------- | ---------------------------------- |
| Query type          | Keywords                            | User/Item vectors                  |
| Matching            | Exact token match                   | Semantic similarity                |
| Ranking             | BM25, TF-IDF                        | Dot product, cosine similarity     |
| Personalization     | Query rewriting                     | User embedding fusion              |
| Index structure     | Inverted index                      | FAISS (IVF, HNSW)                  |

---

## Summary

| Component           | Technology/Algorithm          | Key Configuration                |
| ------------------- | ----------------------------- | -------------------------------- |
| Collaborative       | Matrix Factorization / Two-Tower | 128-dim embeddings            |
| ANN Search          | FAISS IVF-PQ                  | 1000 clusters, nprobe=10         |
| Ranking             | Wide & Deep                   | 256-128-64 hidden units          |
| Cold start (user)   | Onboarding + Demographics     | 1000 initial items               |
| Cold start (item)   | Content embedding + Exploration | 1000 user exposure             |
| Real-time features  | Kafka → Redis                 | 15-min feature TTL               |

