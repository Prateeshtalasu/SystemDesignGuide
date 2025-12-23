# News Feed - Technology Deep Dives

## Overview

This document covers the technical implementation details for key components: fan-out strategies, ranking algorithms, caching, and real-time updates.

---

## 1. Fan-out Strategies Deep Dive

### Fan-out on Write Implementation

```java
@Service
public class FanoutService {
    
    private final GraphService graphService;
    private final FeedCacheService feedCache;
    private final KafkaTemplate<String, PostEvent> kafkaTemplate;
    
    private static final int CELEBRITY_THRESHOLD = 10_000;
    private static final int BATCH_SIZE = 1000;
    
    @KafkaListener(topics = "post-events")
    public void handlePostEvent(PostEvent event) {
        Post post = event.getPost();
        User author = post.getAuthor();
        
        // Check if celebrity
        if (author.getFollowersCount() > CELEBRITY_THRESHOLD) {
            // Store in celebrity posts table, no fan-out
            celebrityPostStore.save(post);
            log.info("Celebrity post stored without fan-out: {}", post.getId());
            return;
        }
        
        // Get all followers
        List<Long> followers = graphService.getFollowers(author.getId());
        
        // Calculate ranking score
        double baseScore = calculateBaseScore(post);
        
        // Batch process followers
        Lists.partition(followers, BATCH_SIZE).forEach(batch -> {
            processBatch(batch, post, baseScore);
        });
    }
    
    private void processBatch(List<Long> followers, Post post, double baseScore) {
        // Prepare feed entries
        Map<String, Double> feedEntries = new HashMap<>();
        
        for (Long followerId : followers) {
            // Personalize score based on relationship
            double personalizedScore = personalizeScore(followerId, post, baseScore);
            String feedKey = "feed:" + followerId;
            feedEntries.put(feedKey, personalizedScore);
        }
        
        // Batch write to Redis
        feedCache.batchAddToFeeds(feedEntries, post.getId());
        
        // Publish real-time notifications
        notifyActiveUsers(followers, post);
    }
    
    private double calculateBaseScore(Post post) {
        long ageMinutes = ChronoUnit.MINUTES.between(post.getCreatedAt(), Instant.now());
        
        // Time decay: half-life of 6 hours
        double recencyScore = Math.exp(-ageMinutes / 360.0);
        
        // Engagement boost (for posts with early engagement)
        double engagementScore = Math.log1p(
            post.getLikesCount() + 
            post.getCommentsCount() * 2 + 
            post.getSharesCount() * 3
        ) / 10.0;
        
        return recencyScore * 0.7 + engagementScore * 0.3;
    }
}
```

### Fan-out on Read Implementation

```java
@Service
public class FeedService {
    
    private final FeedCacheService feedCache;
    private final CelebrityPostService celebrityPostService;
    private final GraphService graphService;
    private final RankingService rankingService;
    
    public FeedResponse getFeed(Long userId, String cursor, int limit) {
        // 1. Get precomputed feed (regular users' posts)
        List<FeedEntry> precomputedFeed = feedCache.getFeed(userId, cursor, limit * 2);
        
        // 2. Get celebrity posts (fan-out on read)
        List<Long> followedCelebrities = graphService.getFollowedCelebrities(userId);
        List<Post> celebrityPosts = fetchCelebrityPosts(followedCelebrities, cursor);
        
        // 3. Merge feeds
        List<FeedEntry> merged = mergeFeed(precomputedFeed, celebrityPosts, userId);
        
        // 4. Apply final ranking
        List<FeedEntry> ranked = rankingService.rank(merged, userId);
        
        // 5. Paginate
        List<FeedEntry> page = ranked.subList(0, Math.min(limit, ranked.size()));
        
        return FeedResponse.builder()
            .posts(page)
            .nextCursor(generateCursor(page))
            .hasMore(ranked.size() > limit)
            .build();
    }
    
    private List<Post> fetchCelebrityPosts(List<Long> celebrityIds, String cursor) {
        if (celebrityIds.isEmpty()) {
            return Collections.emptyList();
        }
        
        // Batch fetch recent posts from all followed celebrities
        Instant since = parseCursorTime(cursor);
        
        return celebrityPostService.getRecentPosts(
            celebrityIds,
            since,
            100  // Max celebrity posts to consider
        );
    }
    
    private List<FeedEntry> mergeFeed(
            List<FeedEntry> precomputed, 
            List<Post> celebrityPosts,
            Long userId) {
        
        // Convert celebrity posts to feed entries
        List<FeedEntry> celebrityEntries = celebrityPosts.stream()
            .map(post -> FeedEntry.builder()
                .postId(post.getId())
                .authorId(post.getAuthorId())
                .score(calculateScore(post, userId))
                .createdAt(post.getCreatedAt())
                .source("celebrity")
                .build())
            .collect(Collectors.toList());
        
        // Merge and sort by score
        List<FeedEntry> merged = new ArrayList<>();
        merged.addAll(precomputed);
        merged.addAll(celebrityEntries);
        
        // Deduplicate (same post might appear in both)
        Map<Long, FeedEntry> deduped = merged.stream()
            .collect(Collectors.toMap(
                FeedEntry::getPostId,
                e -> e,
                (e1, e2) -> e1.getScore() > e2.getScore() ? e1 : e2
            ));
        
        return new ArrayList<>(deduped.values());
    }
}
```

---

## 2. Ranking Algorithm Deep Dive

### Multi-Signal Ranking

```java
@Service
public class RankingService {
    
    private final UserInteractionService interactionService;
    private final ContentAnalyzer contentAnalyzer;
    
    public List<FeedEntry> rank(List<FeedEntry> entries, Long viewerId) {
        UserProfile viewerProfile = getUserProfile(viewerId);
        
        // Score each entry
        entries.forEach(entry -> {
            double score = calculateScore(entry, viewerProfile);
            entry.setFinalScore(score);
        });
        
        // Sort by score descending
        entries.sort(Comparator.comparingDouble(FeedEntry::getFinalScore).reversed());
        
        // Apply diversity rules (avoid too many posts from same author)
        return applyDiversity(entries);
    }
    
    private double calculateScore(FeedEntry entry, UserProfile viewer) {
        double score = 0.0;
        
        // 1. RECENCY (30% weight)
        // Exponential decay with 6-hour half-life
        long ageMinutes = entry.getAgeMinutes();
        double recencyScore = Math.exp(-ageMinutes / 360.0);
        score += recencyScore * 0.30;
        
        // 2. ENGAGEMENT (25% weight)
        // Log-normalized engagement
        double engagementScore = calculateEngagementScore(entry);
        score += engagementScore * 0.25;
        
        // 3. AFFINITY (25% weight)
        // How close is viewer to author
        double affinityScore = calculateAffinityScore(viewer, entry.getAuthorId());
        score += affinityScore * 0.25;
        
        // 4. CONTENT RELEVANCE (15% weight)
        // Does content match viewer's interests
        double relevanceScore = calculateRelevanceScore(viewer, entry);
        score += relevanceScore * 0.15;
        
        // 5. AUTHOR QUALITY (5% weight)
        // Verified, engagement rate, spam score
        double authorScore = calculateAuthorScore(entry.getAuthorId());
        score += authorScore * 0.05;
        
        return score;
    }
    
    private double calculateEngagementScore(FeedEntry entry) {
        // Weighted engagement
        double rawEngagement = 
            entry.getLikesCount() * 1.0 +
            entry.getCommentsCount() * 3.0 +
            entry.getSharesCount() * 5.0;
        
        // Log normalization to handle viral posts
        return Math.log1p(rawEngagement) / 15.0;  // Normalize to ~0-1
    }
    
    private double calculateAffinityScore(UserProfile viewer, Long authorId) {
        // Get interaction history
        InteractionStats stats = interactionService.getStats(viewer.getId(), authorId);
        
        double score = 0.0;
        
        // Recent interactions (last 30 days)
        score += Math.min(stats.getRecentLikes() / 10.0, 0.3);
        score += Math.min(stats.getRecentComments() / 5.0, 0.3);
        score += Math.min(stats.getRecentProfileViews() / 3.0, 0.2);
        
        // Relationship type
        if (stats.isCloseFriend()) {
            score += 0.2;
        }
        
        return Math.min(score, 1.0);
    }
    
    private List<FeedEntry> applyDiversity(List<FeedEntry> entries) {
        List<FeedEntry> diversified = new ArrayList<>();
        Map<Long, Integer> authorCounts = new HashMap<>();
        
        for (FeedEntry entry : entries) {
            Long authorId = entry.getAuthorId();
            int count = authorCounts.getOrDefault(authorId, 0);
            
            // Max 3 consecutive posts from same author
            if (count < 3) {
                diversified.add(entry);
                authorCounts.put(authorId, count + 1);
            } else {
                // Add to end of list
                diversified.add(diversified.size(), entry);
            }
        }
        
        return diversified;
    }
}
```

### Real-time Engagement Updates

```java
@Service
public class EngagementService {
    
    private final RedisTemplate<String, String> redis;
    private final KafkaTemplate<String, EngagementEvent> kafka;
    
    public void recordLike(Long postId, Long userId) {
        // 1. Increment counter in Redis (real-time)
        String key = "engagement:" + postId;
        redis.opsForHash().increment(key, "likes", 1);
        
        // 2. Update user's interaction history
        String interactionKey = "interactions:" + userId + ":" + getAuthorId(postId);
        redis.opsForHash().increment(interactionKey, "likes", 1);
        redis.expire(interactionKey, 30, TimeUnit.DAYS);
        
        // 3. Publish event for async processing
        kafka.send("engagement-events", EngagementEvent.builder()
            .type("like")
            .postId(postId)
            .userId(userId)
            .timestamp(Instant.now())
            .build());
        
        // 4. Update ranking score (async)
        recalculatePostScore(postId);
    }
    
    @Async
    public void recalculatePostScore(Long postId) {
        // Get current engagement
        Map<Object, Object> engagement = redis.opsForHash()
            .entries("engagement:" + postId);
        
        int likes = Integer.parseInt((String) engagement.getOrDefault("likes", "0"));
        int comments = Integer.parseInt((String) engagement.getOrDefault("comments", "0"));
        int shares = Integer.parseInt((String) engagement.getOrDefault("shares", "0"));
        
        // Calculate new score
        double score = Math.log1p(likes + comments * 3 + shares * 5);
        
        // Update in database (periodic batch)
        postRepository.updateEngagementScore(postId, score);
    }
}
```

---

## 3. Feed Cache Implementation

### Redis Feed Cache

```java
@Service
public class FeedCacheService {
    
    private final RedisTemplate<String, String> redis;
    private static final int MAX_FEED_SIZE = 500;
    private static final Duration FEED_TTL = Duration.ofHours(24);
    
    public void addToFeed(Long userId, Long postId, double score) {
        String key = "feed:" + userId;
        
        // Add to sorted set with score
        redis.opsForZSet().add(key, postId.toString(), score);
        
        // Trim to max size (remove lowest scores)
        redis.opsForZSet().removeRange(key, 0, -MAX_FEED_SIZE - 1);
        
        // Refresh TTL
        redis.expire(key, FEED_TTL);
    }
    
    public void batchAddToFeeds(Map<Long, Double> userScores, Long postId) {
        // Use pipeline for efficiency
        redis.executePipelined((RedisCallback<Object>) connection -> {
            userScores.forEach((userId, score) -> {
                byte[] key = ("feed:" + userId).getBytes();
                connection.zAdd(key, score, postId.toString().getBytes());
            });
            return null;
        });
    }
    
    public List<FeedEntry> getFeed(Long userId, String cursor, int limit) {
        String key = "feed:" + userId;
        
        double maxScore = cursor != null ? 
            parseCursorScore(cursor) : Double.MAX_VALUE;
        
        // Get posts with scores less than cursor
        Set<ZSetOperations.TypedTuple<String>> results = redis.opsForZSet()
            .reverseRangeByScoreWithScores(key, 0, maxScore, 0, limit);
        
        if (results == null || results.isEmpty()) {
            return Collections.emptyList();
        }
        
        return results.stream()
            .map(tuple -> FeedEntry.builder()
                .postId(Long.parseLong(tuple.getValue()))
                .score(tuple.getScore())
                .build())
            .collect(Collectors.toList());
    }
    
    public void removeFromFeed(Long userId, Long postId) {
        String key = "feed:" + userId;
        redis.opsForZSet().remove(key, postId.toString());
    }
    
    public void invalidateFeed(Long userId) {
        String key = "feed:" + userId;
        redis.delete(key);
    }
}
```

### Cache Warming

```java
@Service
public class FeedWarmingService {
    
    private final FeedCacheService feedCache;
    private final FeedGenerationService feedGenerator;
    
    // Warm cache for users who will likely open app soon
    @Scheduled(fixedRate = 60000)  // Every minute
    public void warmCachesForActiveUsers() {
        // Get users who were active in last hour but cache is cold
        List<Long> usersToWarm = getUsersNeedingWarmCache();
        
        for (Long userId : usersToWarm) {
            try {
                warmUserFeed(userId);
            } catch (Exception e) {
                log.warn("Failed to warm cache for user {}", userId, e);
            }
        }
    }
    
    private void warmUserFeed(Long userId) {
        // Check if cache exists
        if (feedCache.exists(userId)) {
            return;
        }
        
        // Generate feed
        List<FeedEntry> feed = feedGenerator.generateFeed(userId, 500);
        
        // Populate cache
        feed.forEach(entry -> 
            feedCache.addToFeed(userId, entry.getPostId(), entry.getScore())
        );
        
        log.info("Warmed cache for user {} with {} entries", userId, feed.size());
    }
    
    // Predictive warming based on user patterns
    public void predictiveWarm(Long userId) {
        // Get user's typical active times
        List<Integer> activeHours = getUserActiveHours(userId);
        int currentHour = LocalTime.now().getHour();
        
        // If user typically active in next hour, warm cache
        if (activeHours.contains((currentHour + 1) % 24)) {
            warmUserFeed(userId);
        }
    }
}
```

---

## 4. Real-time Updates

### WebSocket Handler

```java
@Component
public class FeedWebSocketHandler extends TextWebSocketHandler {
    
    private final Map<Long, Set<WebSocketSession>> userSessions = new ConcurrentHashMap<>();
    private final RedisMessageListenerContainer redisListener;
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        Long userId = extractUserId(session);
        
        // Track session
        userSessions.computeIfAbsent(userId, k -> ConcurrentHashMap.newKeySet())
            .add(session);
        
        // Subscribe to user's channel
        subscribeToUserChannel(userId);
        
        log.info("WebSocket connected for user {}", userId);
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        Long userId = extractUserId(session);
        
        Set<WebSocketSession> sessions = userSessions.get(userId);
        if (sessions != null) {
            sessions.remove(session);
            if (sessions.isEmpty()) {
                userSessions.remove(userId);
                unsubscribeFromUserChannel(userId);
            }
        }
    }
    
    public void sendNewPostNotification(Long userId, Post post) {
        Set<WebSocketSession> sessions = userSessions.get(userId);
        if (sessions == null || sessions.isEmpty()) {
            return;
        }
        
        String message = objectMapper.writeValueAsString(Map.of(
            "type", "new_post",
            "post", Map.of(
                "id", post.getId(),
                "author", post.getAuthor().getDisplayName(),
                "preview", truncate(post.getContent(), 100),
                "timestamp", post.getCreatedAt()
            )
        ));
        
        sessions.forEach(session -> {
            try {
                session.sendMessage(new TextMessage(message));
            } catch (IOException e) {
                log.warn("Failed to send message to session", e);
            }
        });
    }
    
    private void subscribeToUserChannel(Long userId) {
        String channel = "user:" + userId + ":feed";
        
        redisListener.addMessageListener(
            (message, pattern) -> {
                String payload = new String(message.getBody());
                handleRedisMessage(userId, payload);
            },
            new ChannelTopic(channel)
        );
    }
}
```

### Push Notification Service

```java
@Service
public class PushNotificationService {
    
    private final FirebaseMessaging firebaseMessaging;
    private final APNsService apnsService;
    private final UserDeviceRepository deviceRepository;
    
    @Async
    public void notifyNewPost(Long userId, Post post) {
        // Check user's notification preferences
        NotificationPreferences prefs = getPreferences(userId);
        if (!prefs.isNewPostsEnabled()) {
            return;
        }
        
        // Check if user is currently active (don't push if app is open)
        if (isUserCurrentlyActive(userId)) {
            return;
        }
        
        // Get user's devices
        List<UserDevice> devices = deviceRepository.findByUserId(userId);
        
        // Build notification
        String title = post.getAuthor().getDisplayName();
        String body = truncate(post.getContent(), 100);
        
        for (UserDevice device : devices) {
            try {
                if (device.getPlatform().equals("ios")) {
                    sendAPNs(device.getToken(), title, body, post.getId());
                } else {
                    sendFCM(device.getToken(), title, body, post.getId());
                }
            } catch (Exception e) {
                log.warn("Failed to send push to device {}", device.getId(), e);
                handleInvalidToken(device);
            }
        }
    }
    
    private void sendFCM(String token, String title, String body, Long postId) {
        Message message = Message.builder()
            .setToken(token)
            .setNotification(Notification.builder()
                .setTitle(title)
                .setBody(body)
                .build())
            .putData("post_id", postId.toString())
            .putData("type", "new_post")
            .build();
        
        firebaseMessaging.send(message);
    }
}
```

---

## 5. Social Graph Management

### Graph Service

```java
@Service
public class GraphService {
    
    private final FollowRepository followRepository;
    private final RedisTemplate<String, String> redis;
    
    private static final int CELEBRITY_THRESHOLD = 10_000;
    private static final Duration CACHE_TTL = Duration.ofHours(1);
    
    public void follow(Long followerId, Long followingId) {
        // 1. Create follow relationship
        Follow follow = Follow.builder()
            .followerId(followerId)
            .followingId(followingId)
            .createdAt(Instant.now())
            .build();
        
        followRepository.save(follow);
        
        // 2. Update counts
        userRepository.incrementFollowersCount(followingId);
        userRepository.incrementFollowingCount(followerId);
        
        // 3. Update cache
        redis.opsForSet().add("following:" + followerId, followingId.toString());
        redis.opsForSet().add("followers:" + followingId, followerId.toString());
        
        // 4. Check if now celebrity
        long followersCount = userRepository.getFollowersCount(followingId);
        if (followersCount >= CELEBRITY_THRESHOLD) {
            markAsCelebrity(followingId);
        }
        
        // 5. Backfill recent posts to follower's feed
        backfillRecentPosts(followerId, followingId);
    }
    
    public void unfollow(Long followerId, Long followingId) {
        // 1. Remove relationship
        followRepository.delete(followerId, followingId);
        
        // 2. Update counts
        userRepository.decrementFollowersCount(followingId);
        userRepository.decrementFollowingCount(followerId);
        
        // 3. Update cache
        redis.opsForSet().remove("following:" + followerId, followingId.toString());
        redis.opsForSet().remove("followers:" + followingId, followerId.toString());
        
        // 4. Remove posts from feed
        removePostsFromFeed(followerId, followingId);
    }
    
    public List<Long> getFollowers(Long userId) {
        String cacheKey = "followers:" + userId;
        
        // Try cache first
        Set<String> cached = redis.opsForSet().members(cacheKey);
        if (cached != null && !cached.isEmpty()) {
            return cached.stream()
                .map(Long::parseLong)
                .collect(Collectors.toList());
        }
        
        // Load from database
        List<Long> followers = followRepository.findFollowerIds(userId);
        
        // Populate cache
        if (!followers.isEmpty()) {
            String[] followerStrs = followers.stream()
                .map(String::valueOf)
                .toArray(String[]::new);
            redis.opsForSet().add(cacheKey, followerStrs);
            redis.expire(cacheKey, CACHE_TTL);
        }
        
        return followers;
    }
    
    public List<Long> getFollowedCelebrities(Long userId) {
        // Get all users this person follows
        List<Long> following = getFollowing(userId);
        
        // Filter to celebrities only
        return following.stream()
            .filter(this::isCelebrity)
            .collect(Collectors.toList());
    }
    
    private void backfillRecentPosts(Long followerId, Long followingId) {
        // Get recent posts from new following
        List<Post> recentPosts = postRepository.findRecentByAuthor(
            followingId, 
            Instant.now().minus(7, ChronoUnit.DAYS),
            20
        );
        
        // Add to follower's feed
        for (Post post : recentPosts) {
            double score = calculatePostScore(post, followerId);
            feedCacheService.addToFeed(followerId, post.getId(), score);
        }
    }
}
```

---

## 6. Monitoring & Observability

### Key Metrics

```java
@Component
public class FeedMetrics {
    
    private final MeterRegistry registry;
    
    // Latency
    private final Timer feedLoadLatency;
    private final Timer fanoutLatency;
    private final Timer rankingLatency;
    
    // Throughput
    private final Counter feedRequests;
    private final Counter postsCreated;
    private final Counter fanoutWrites;
    
    // Cache
    private final Counter cacheHits;
    private final Counter cacheMisses;
    
    public FeedMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.feedLoadLatency = Timer.builder("feed.load.latency")
            .description("Feed load latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.fanoutLatency = Timer.builder("fanout.latency")
            .description("Fan-out processing latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
        
        this.cacheHits = Counter.builder("feed.cache.hits")
            .description("Feed cache hits")
            .register(registry);
    }
    
    public void recordFeedLoad(long durationMs, boolean cacheHit, int postsReturned) {
        feedLoadLatency.record(durationMs, TimeUnit.MILLISECONDS);
        feedRequests.increment();
        
        if (cacheHit) {
            cacheHits.increment();
        } else {
            cacheMisses.increment();
        }
    }
}
```

### Alerting Thresholds

| Metric               | Warning    | Critical   | Action                |
| -------------------- | ---------- | ---------- | --------------------- |
| Feed P99 latency     | > 400ms    | > 800ms    | Scale feed servers    |
| Fan-out lag          | > 30s      | > 60s      | Scale fan-out workers |
| Cache hit rate       | < 80%      | < 60%      | Investigate cache     |
| Error rate           | > 0.1%     | > 1%       | Check dependencies    |

---

## Summary

| Component       | Technology/Algorithm          | Key Configuration                |
| --------------- | ----------------------------- | -------------------------------- |
| Fan-out         | Hybrid (write + read)         | 10K follower threshold           |
| Ranking         | Multi-signal ML               | 5 signals, weighted combination  |
| Feed cache      | Redis Sorted Sets             | 500 posts, 24h TTL               |
| Real-time       | WebSocket + Redis Pub/Sub     | Per-user channels                |
| Social graph    | PostgreSQL + Redis cache      | 1h cache TTL                     |

