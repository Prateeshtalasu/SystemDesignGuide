# Recommendation System - Interview Grilling

## Overview

This document contains common interview questions, trade-off discussions, failure scenarios, and level-specific expectations for a recommendation system design.

---

## Trade-off Questions

### Q1: Why use a two-stage approach (candidate generation + ranking)?

**Answer:**

**The Scale Problem:**
```
500M users × 100M items = 50 quadrillion possible pairs
Scoring all pairs per request: Impossible in real-time
```

**Two-Stage Solution:**

| Stage                | Items Considered | Latency | Model Complexity |
| -------------------- | ---------------- | ------- | ---------------- |
| Candidate Generation | 100M → 1000      | ~5ms    | Simple (ANN)     |
| Ranking              | 1000 → 50        | ~20ms   | Complex (DNN)    |

**Why This Works:**
1. **Candidate Generation**: Fast, approximate - uses embeddings and ANN search
2. **Ranking**: Slow, precise - uses 100+ features and deep neural network

**Alternative: Single-Stage**
- Would need to score all 100M items
- Even at 1μs per item = 100 seconds
- Not feasible for real-time

**Trade-offs:**
- Two-stage may miss good items not in candidates
- Mitigation: Multiple candidate sources (collaborative, content, popularity)

---

### Q2: How do you handle the cold start problem?

**Answer:**

**Cold Start Types:**

| Type      | Problem                        | Solution                           |
| --------- | ------------------------------ | ---------------------------------- |
| New User  | No interaction history         | Onboarding, demographics, popular  |
| New Item  | No user interactions           | Content features, exploration      |

**New User Strategy:**

```java
public List<Candidate> handleNewUser(User user) {
    // 1. Onboarding quiz (if completed)
    if (user.hasOnboardingData()) {
        return recommendFromOnboarding(user);
    }
    
    // 2. Demographics (age, location)
    if (user.hasDemographics()) {
        return recommendFromDemographicCohort(user);
    }
    
    // 3. Fallback: Global popularity
    return getGloballyPopular();
}
```

**New Item Strategy:**
1. **Content-based embedding**: Generate embedding from title, description, category
2. **Exploration**: Force exposure to sample of users
3. **Bandits**: Multi-armed bandit for exploration vs exploitation

**Exploration Budget:**
- New items get guaranteed impressions (e.g., 1000)
- Track engagement rate
- Graduate to regular ranking once enough data

---

### Q3: Collaborative filtering vs Content-based filtering - when to use each?

**Answer:**

| Aspect           | Collaborative Filtering       | Content-Based Filtering        |
| ---------------- | ----------------------------- | ------------------------------ |
| **Data needed**  | User-item interactions        | Item attributes                |
| **Cold start**   | Struggles with new users/items| Handles new items well         |
| **Serendipity**  | High (finds unexpected items) | Low (similar to past)          |
| **Scalability**  | O(users × items)              | O(items × features)            |
| **Explainability**| "Users like you..."          | "Because you liked X..."       |

**When to Use Collaborative:**
- Mature system with lots of interactions
- Discovery is important
- Items are hard to describe (music, art)

**When to Use Content-Based:**
- New system with few interactions
- Items have rich metadata
- Need to explain recommendations

**Best Practice: Hybrid**
```
Final_Score = α × Collaborative_Score + (1-α) × Content_Score

Where α = f(user_interaction_count)
- New users: α = 0.2 (more content-based)
- Active users: α = 0.8 (more collaborative)
```

---

### Q4: How do you prevent filter bubbles?

**Answer:**

**The Problem:**
Users only see content similar to what they've seen → Echo chamber

**Solutions:**

1. **Diversity Constraints in Ranking:**
```java
// Max 3 items from same category in top 20
private List<RankedItem> applyDiversity(List<RankedItem> items) {
    Map<String, Integer> categoryCount = new HashMap<>();
    List<RankedItem> result = new ArrayList<>();
    
    for (RankedItem item : items) {
        String category = item.getCategory();
        if (categoryCount.getOrDefault(category, 0) < 3) {
            result.add(item);
            categoryCount.merge(category, 1, Integer::sum);
        }
    }
    return result;
}
```

2. **Exploration Slots:**
- Reserve 10% of recommendations for exploration
- Show items from categories user hasn't engaged with

3. **Serendipity Injection:**
- Occasionally recommend items with low predicted score but high novelty
- "You might also like..." section

4. **Time Decay on Categories:**
- Reduce weight of categories user has seen recently
- Encourage variety over time

**Metrics:**
- Category coverage: % of categories shown
- Intra-list diversity: Avg distance between items in list
- Novelty: % of items user hasn't seen similar to

---

### Q5: How do you evaluate recommendation quality?

**Answer:**

**Offline Metrics:**

| Metric     | Formula                        | What It Measures               |
| ---------- | ------------------------------ | ------------------------------ |
| Precision@K| Relevant in top K / K          | Accuracy of top recommendations|
| Recall@K   | Relevant in top K / Total relevant | Coverage of relevant items  |
| NDCG@K     | DCG / Ideal DCG                | Ranking quality                |
| AUC        | Area under ROC curve           | Overall ranking ability        |
| Coverage   | Items recommended / Total items| Catalog utilization            |

**Online Metrics (A/B Test):**

| Metric           | Definition                     | Target               |
| ---------------- | ------------------------------ | -------------------- |
| CTR              | Clicks / Impressions           | > 5%                 |
| Conversion Rate  | Purchases / Clicks             | > 2%                 |
| Session Duration | Time spent in session          | > 10 min             |
| Return Rate      | Users returning next day       | > 40%                |

**Offline vs Online Gap:**
- Offline metrics don't capture:
  - Position bias (users click top items)
  - Novelty (users ignore familiar items)
  - Context (time, device, mood)
- Always validate with A/B tests

---

## Scaling Questions

### Q6: How would you scale from 10M to 500M users?

**Answer:**

**Current State (10M users):**
- Single FAISS index: 10M items fits in memory
- Single ranking cluster
- ~10K QPS

**Scaling to 500M users (50x):**

1. **Candidate Generation:**
```
Before: Single FAISS server
After: Sharded FAISS across 10 servers
       - Shard by item_id % 10
       - Query all shards in parallel
       - Merge results
```

2. **Ranking:**
```
Before: 5 GPU servers
After: 50 GPU servers
       - Horizontal scaling
       - Batch inference for efficiency
```

3. **Feature Store:**
```
Before: Single Redis cluster (64 nodes)
After: Sharded Redis (200+ nodes)
       - Shard by user_id
       - Regional clusters
```

4. **Pre-computation:**
```
Before: Pre-compute for 1M active users
After: Pre-compute for 100M active users
       - Tiered approach:
         - Top 10M users: Full pre-computation
         - Next 90M: Partial pre-computation
         - Rest: On-demand
```

**Architecture Changes:**
- Regional deployments (US, EU, APAC)
- Tiered caching (hot/warm/cold users)
- Async ranking for non-critical paths

---

### Q7: What if the ranking model inference is too slow?

**Answer:**

**Current State:**
- 1000 candidates × 100 features = 100K feature values
- Neural network inference: 20ms

**Optimization Strategies:**

1. **Model Distillation:**
```python
# Train smaller model to mimic large model
student_model = SmallRankingModel(hidden_units=[64, 32])
student_model.fit(X_train, teacher_model.predict(X_train))
# Result: 5ms inference instead of 20ms
```

2. **Quantization:**
```python
# Convert float32 to int8
quantized_model = tf.lite.TFLiteConverter.from_saved_model(model)
quantized_model.optimizations = [tf.lite.Optimize.DEFAULT]
# Result: 4x faster inference
```

3. **Reduce Candidates:**
```java
// Pre-filter candidates before ranking
List<Candidate> filtered = candidates.stream()
    .filter(c -> c.getCandidateScore() > 0.3)  // Remove low-quality
    .limit(500)  // Reduce from 1000 to 500
    .collect(Collectors.toList());
```

4. **Caching Partial Results:**
```java
// Cache user-independent item scores
double itemScore = itemScoreCache.get(itemId);  // Pre-computed
double userItemScore = computeUserItemScore(userId, itemId);  // Real-time
double finalScore = 0.7 * itemScore + 0.3 * userItemScore;
```

5. **Two-Phase Ranking:**
```
Phase 1: Fast model on 1000 candidates → Top 200
Phase 2: Complex model on 200 candidates → Top 50
```

---

## Failure Scenarios

### Scenario 1: FAISS Index Corruption

**Problem:** FAISS index returns wrong results after update.

**Detection:**
- Monitor recommendation quality metrics
- Compare results with backup index
- Automated sanity checks

**Solution:**

```java
public List<ScoredItem> searchWithFallback(float[] query, int k) {
    try {
        List<ScoredItem> results = primaryIndex.search(query, k);
        
        // Sanity check
        if (results.isEmpty() || results.get(0).getScore() < 0.1) {
            log.warn("Primary index returning poor results");
            return backupIndex.search(query, k);
        }
        
        return results;
    } catch (Exception e) {
        log.error("Primary index failed", e);
        return backupIndex.search(query, k);
    }
}
```

**Prevention:**
- Blue-green deployment for index updates
- Keep last known good index as backup
- Gradual rollout with quality monitoring

---

### Scenario 2: Feature Store Latency Spike

**Problem:** Redis latency increases from 1ms to 100ms.

**Impact:**
- Ranking latency increases
- Timeouts for some requests

**Solution:**

```java
public RankingFeatures getFeaturesWithTimeout(String userId, String itemId) {
    try {
        return featureStore.getFeatures(userId, itemId)
            .orTimeout(10, TimeUnit.MILLISECONDS)
            .get();
    } catch (TimeoutException e) {
        // Use default features
        return getDefaultFeatures(userId, itemId);
    }
}

private RankingFeatures getDefaultFeatures(String userId, String itemId) {
    return RankingFeatures.builder()
        .userEmbedding(getAverageUserEmbedding())
        .itemEmbedding(getItemEmbedding(itemId))  // From local cache
        .useDefaults(true)
        .build();
}
```

**Prevention:**
- Local cache for frequently accessed features
- Circuit breaker on feature store
- Feature importance: rank with subset if needed

---

### Scenario 3: Model Serving Outage

**Problem:** GPU cluster for ranking model is unavailable.

**Solution:**

```java
public List<RankedItem> rankWithFallback(List<Candidate> candidates) {
    if (modelCircuitBreaker.isOpen()) {
        return fallbackRanking(candidates);
    }
    
    try {
        return modelService.rank(candidates);
    } catch (Exception e) {
        modelCircuitBreaker.recordFailure();
        return fallbackRanking(candidates);
    }
}

private List<RankedItem> fallbackRanking(List<Candidate> candidates) {
    // Simple rule-based ranking
    return candidates.stream()
        .map(c -> RankedItem.builder()
            .itemId(c.getItemId())
            .score(calculateSimpleScore(c))
            .build())
        .sorted(Comparator.comparingDouble(RankedItem::getScore).reversed())
        .collect(Collectors.toList());
}

private double calculateSimpleScore(Candidate c) {
    return 0.5 * c.getCandidateScore() +
           0.3 * c.getPopularity() +
           0.2 * c.getFreshness();
}
```

---

## Level-Specific Expectations

### L4 (Entry-Level)

**What's Expected:**
- Understand collaborative vs content-based filtering
- Basic system components
- Simple database schema

**Sample Answer Quality:**

> "I'd store user interactions in a database and use them to find similar users. For each user, I'd recommend items that similar users liked. I'd cache popular recommendations to reduce load."

**Red Flags:**
- No mention of scale challenges
- Doesn't understand cold start
- No discussion of latency

---

### L5 (Mid-Level)

**What's Expected:**
- Two-stage architecture
- Cold start handling
- Feature engineering basics
- Caching strategies

**Sample Answer Quality:**

> "I'd use a two-stage approach. First, candidate generation using collaborative filtering with ANN search - this reduces 100M items to 1000 candidates in ~5ms. Then, a ranking model scores these candidates using user features, item features, and context.

> For cold start users, I'd use an onboarding flow to capture preferences, then recommend popular items in those categories. For cold start items, I'd generate embeddings from content features and use exploration to gather initial engagement data.

> I'd pre-compute recommendations for active users and cache in Redis, with a 24-hour TTL. This gives ~60% cache hit rate."

**Red Flags:**
- Can't explain why two-stage
- No understanding of embeddings
- Missing real-time personalization

---

### L6 (Senior)

**What's Expected:**
- System evolution and trade-offs
- A/B testing infrastructure
- Model deployment strategies
- Filter bubble mitigation

**Sample Answer Quality:**

> "Let me walk through the evolution. For MVP, we'd use simple collaborative filtering with item-item similarity - it's interpretable and works with limited data. As we scale, we move to matrix factorization for user and item embeddings.

> The key architectural decision is the two-stage pipeline. Candidate generation is cheap but imprecise - we use multiple sources (collaborative, content-based, popularity) to ensure coverage. Ranking is expensive but precise - we use a Wide & Deep model that combines memorization (wide) and generalization (deep).

> For A/B testing, we use hash-based assignment to ensure consistent user experience. We run experiments for at least 2 weeks to account for novelty effects. We track not just CTR but also long-term metrics like return rate and diversity.

> Filter bubbles are a real concern. We address this through diversity constraints in ranking (max 3 items per category), exploration slots (10% of recommendations), and tracking intra-list diversity as a metric. We also use time-decay on categories to encourage variety.

> For model deployment, we use shadow mode first - new model runs alongside production without serving. We compare predictions and only promote if quality metrics improve. Then gradual rollout: 1% → 10% → 50% → 100%."

**Red Flags:**
- Can't discuss trade-offs
- No awareness of filter bubbles
- Missing model deployment strategy

---

## Common Interviewer Pushbacks

### "Your recommendations will be stale"

**Response:**
"There's a trade-off between freshness and compute cost. We handle this at multiple levels:

1. **Pre-computed recommendations** have 24-hour TTL but are re-ranked at serving time with fresh context (time of day, recent clicks).

2. **Real-time signals** are captured via Kafka and update user features in Redis within seconds. The ranking model sees these fresh signals.

3. **Session context** is used to boost items related to what user just clicked, providing immediate responsiveness.

So while candidate generation might be 24 hours old, the final ranking incorporates signals from the last few seconds."

### "How do you ensure fairness?"

**Response:**
"Fairness has multiple dimensions:

1. **Item fairness**: We track coverage (% of items recommended) and ensure new/small items get exposure through exploration. We also audit for bias against certain categories.

2. **User fairness**: We ensure all users get quality recommendations, not just power users. We track recommendation quality across user segments.

3. **Algorithmic fairness**: We audit our models for demographic bias. If recommendations differ significantly by demographic group, we investigate and mitigate.

We have a fairness dashboard that tracks these metrics and alerts on significant deviations."

---

## Summary

| Question Type       | Key Points to Cover                                |
| ------------------- | -------------------------------------------------- |
| Trade-offs          | Two-stage, collaborative vs content, filter bubbles|
| Scaling             | Sharding, pre-computation, regional deployment     |
| Failures            | Fallback ranking, circuit breakers, graceful degradation |
| Cold start          | Onboarding, demographics, exploration              |
| Quality             | Offline metrics, A/B testing, long-term metrics    |

