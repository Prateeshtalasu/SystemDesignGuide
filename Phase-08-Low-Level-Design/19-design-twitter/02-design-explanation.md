# 🐦 Design Twitter (Simplified) - Design Explanation

## STEP 2: Detailed Design Explanation

This document covers the design decisions, SOLID principles application, design patterns used, and complexity analysis for the Twitter System.

---

## STEP 3: SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `User` | Store user data and relationships | User model changes |
| `Tweet` | Store tweet content and engagement | Tweet model changes |
| `FeedService` | Generate user feeds | Feed algorithm changes |
| `TwitterService` | Coordinate all operations | Business logic changes |

**SRP in Action:**

```java
// User ONLY manages user data and relationships
public class User {
    private final Set<String> followers;
    private final Set<String> following;
    
    public void addFollower(String userId) { }
    public void addFollowing(String userId) { }
}

// Tweet ONLY manages tweet data and engagement
public class Tweet {
    private final Set<String> likes;
    private final Set<String> retweetedBy;
    
    public void addLike(String userId) { }
    public void addRetweet(String userId) { }
}

// FeedService ONLY handles feed generation
public class FeedService {
    public List<Tweet> getNewsFeed(String userId, int limit) { }
    public List<Tweet> getTimeline(String userId, int limit) { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Feed Algorithms:**

```java
public interface FeedGenerator {
    List<Tweet> generate(String userId, int limit);
}

public class ChronologicalFeed implements FeedGenerator {
    @Override
    public List<Tweet> generate(String userId, int limit) {
        // Sort by timestamp
    }
}

public class RankedFeed implements FeedGenerator {
    @Override
    public List<Tweet> generate(String userId, int limit) {
        // Sort by engagement score
    }
}

// No changes to existing code
public class FeedService {
    private FeedGenerator generator;
    
    public void setFeedGenerator(FeedGenerator generator) {
        this.generator = generator;
    }
}
```

**Adding New Tweet Types:**

```java
public class Tweet {
    // Base tweet
}

public class PollTweet extends Tweet {
    private List<String> options;
    private Map<String, Set<String>> votes;
}

public class ThreadTweet extends Tweet {
    private List<String> replyIds;
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All feed generators work interchangeably:**

```java
public class FeedService {
    private FeedGenerator generator;
    
    public List<Tweet> getNewsFeed(String userId, int limit) {
        // Works with any FeedGenerator implementation
        return generator.generate(userId, limit);
    }
}

// All these work:
feedService.setFeedGenerator(new ChronologicalFeed());
feedService.setFeedGenerator(new RankedFeed());
feedService.setFeedGenerator(new PersonalizedFeed());
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Followable {
    void addFollower(String userId);
    void removeFollower(String userId);
    Set<String> getFollowers();
}

public interface Engageable {
    void addLike(String userId);
    void removeLike(String userId);
    int getLikeCount();
}

public class User implements Followable { }
public class Tweet implements Engageable { }
```

---

### 5. Dependency Inversion Principle (DIP)

**Better with DIP:**

```java
public interface UserRepository {
    void save(User user);
    User findById(String id);
    User findByUsername(String username);
}

public interface TweetRepository {
    void save(Tweet tweet);
    Tweet findById(String id);
    List<Tweet> findByAuthor(String userId);
}

public class TwitterService {
    private final UserRepository userRepo;
    private final TweetRepository tweetRepo;
    private final FeedGenerator feedGenerator;
    
    public TwitterService(UserRepository users, TweetRepository tweets,
                         FeedGenerator feed) {
        this.userRepo = users;
        this.tweetRepo = tweets;
        this.feedGenerator = feed;
    }
}
```

**Why We Use Concrete Classes in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete classes instead of repository interfaces for the following reasons:

1. **In-Memory Data Structures**: The system operates on in-memory data structures (users, tweets, feeds). Repository interfaces are more relevant for persistent storage, which is often out of scope for LLD.

2. **Core Algorithm Focus**: LLD interviews focus on feed generation algorithms, tweet posting logic, and follower management. Adding repository abstractions shifts focus away from these core concepts.

3. **Single Implementation**: There's no requirement for multiple data access or feed generation implementations in the interview context. The abstraction doesn't provide value for demonstrating LLD skills.

4. **Production vs Interview**: In production systems, we would absolutely extract `UserRepository`, `TweetRepository`, and `FeedGenerator` interfaces for:
   - Testability (mock repositories and generators in unit tests)
   - Data access flexibility (swap database implementations)
   - Algorithm flexibility (swap feed generation strategies)

**The Trade-off:**
- **Interview Scope**: Concrete classes focus on feed algorithms and social graph management
- **Production Scope**: Repository interfaces provide testability and data access flexibility

---

## SOLID Principles Check

| Principle | Rating | Explanation | Fix if WEAK/FAIL | Tradeoff |
|-----------|--------|-------------|------------------|----------|
| **SRP** | PASS | Each class has a single, well-defined responsibility. User stores user data, Tweet stores tweet data, FeedService generates feeds, TwitterService coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new tweet types, feed algorithms) without modifying existing code. Factory pattern and strategy pattern enable this. | N/A | - |
| **LSP** | PASS | All tweet types (Tweet, Retweet) properly implement the tweet contract. They are substitutable. | N/A | - |
| **ISP** | PASS | Interfaces are minimal and focused. No unused methods forced on clients. | N/A | - |
| **DIP** | ACCEPTABLE (LLD Scope) | TwitterService depends on concrete classes. For LLD interview scope, this is acceptable as it focuses on core feed generation and tweet management algorithms. In production, we would depend on UserRepository, TweetRepository, and FeedGenerator interfaces. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability and data access flexibility |

---

## Why Alternatives Were Rejected

### Alternative 1: Store Feeds Pre-computed in User Object

**What it is:**
Store a pre-computed news feed list directly in the User object, updating it whenever a followed user posts a tweet.

**Why rejected:**
- **Scalability issue:** For users with many followers (e.g., celebrities with millions of followers), updating all follower feeds on every tweet would be extremely expensive (O(followers) writes per tweet).
- **Memory overhead:** Each user would store a full feed, duplicating tweet references across many users.
- **Consistency complexity:** Managing feed updates across concurrent operations becomes complex and error-prone.
- **What breaks:** High write amplification, especially for popular users. Feed freshness becomes a bottleneck.

### Alternative 2: Single Monolithic Service Class

**What it is:**
Combine all functionality (user management, tweet storage, feed generation) into a single TwitterService class without separate classes like FeedService.

**Why rejected:**
- **Violates SRP:** A single class would handle too many responsibilities, making it harder to maintain and test.
- **Difficult to optimize:** Feed generation algorithms would be tightly coupled with business logic, making it harder to swap algorithms or optimize independently.
- **Testing complexity:** Testing feed generation would require setting up the entire TwitterService, making unit tests more complex.
- **What breaks:** Code maintainability, testability, and the ability to optimize feed generation independently from other operations.

---

## Design Patterns Used

### 1. Factory Pattern (Static Factory Method)

**Where:** Creating retweets

```java
public class Tweet {
    // Private constructor for retweets
    private Tweet(String retweeterId, Tweet original) {
        this.isRetweet = true;
        this.originalTweetId = original.getId();
    }
    
    // Factory method
    public static Tweet createRetweet(String retweeterId, Tweet original) {
        original.addRetweet(retweeterId);
        return new Tweet(retweeterId, original);
    }
}
```

---

### 2. Strategy Pattern

**Where:** Feed generation algorithms

```java
public interface FeedGenerator {
    List<Tweet> generate(String userId, int limit);
}

public class ChronologicalFeed implements FeedGenerator { }
public class RankedFeed implements FeedGenerator { }
```

---

### 3. Observer Pattern (Potential)

**Where:** Notifications for new tweets, likes, follows

```java
public interface TwitterObserver {
    void onNewTweet(Tweet tweet);
    void onNewFollower(User follower, User followee);
    void onLike(User liker, Tweet tweet);
}

public class NotificationService implements TwitterObserver {
    @Override
    public void onNewFollower(User follower, User followee) {
        sendNotification(followee.getId(), 
            "@" + follower.getUsername() + " followed you!");
    }
}
```

---

### 4. Iterator Pattern

**Where:** Optimized feed generation

```java
public List<Tweet> getNewsFeedOptimized(String userId, int limit) {
    // Create iterators for each user's tweets
    List<Iterator<String>> iterators = new ArrayList<>();
    
    // User's own tweets
    iterators.add(user.getTweetIds().iterator());
    
    // Followed users' tweets
    for (String followeeId : user.getFollowing()) {
        iterators.add(followee.getTweetIds().iterator());
    }
    
    // Merge using priority queue
    PriorityQueue<TweetIteratorEntry> pq = new PriorityQueue<>(...);
    
    // Process iterators efficiently
}
```

---

## Feed Generation Algorithm

### Merge K Sorted Lists Approach

```
User follows: A, B, C

A's tweets: [A5, A3, A1]  (newest first)
B's tweets: [B4, B2]
C's tweets: [C6, C3, C1]

Priority Queue (max-heap by timestamp):
Initial: [C6, A5, B4]

Extract C6 → add C3 → [A5, B4, C3]
Extract A5 → add A3 → [B4, C3, A3]
Extract B4 → add B2 → [C3, A3, B2]
...

Result: [C6, A5, B4, C3, A3, B2, ...]
```

**Time Complexity:** O(N log K)
- N = total tweets to return
- K = number of users being merged

**Space Complexity:** O(K)
- Priority queue holds at most K entries

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you handle real-time updates?

**Answer:**

```java
public class RealTimeFeedService {
    private final Map<String, List<FeedListener>> listeners;
    
    public interface FeedListener {
        void onNewTweet(Tweet tweet);
    }
    
    public void subscribe(String userId, FeedListener listener) {
        listeners.computeIfAbsent(userId, k -> new ArrayList<>())
            .add(listener);
    }
    
    public void onTweet(Tweet tweet) {
        User author = getUser(tweet.getAuthorId());
        
        // Notify all followers
        for (String followerId : author.getFollowers()) {
            List<FeedListener> userListeners = listeners.get(followerId);
            if (userListeners != null) {
                for (FeedListener listener : userListeners) {
                    listener.onNewTweet(tweet);
                }
            }
        }
    }
}
```

---

### Q2: How would you implement direct messages?

**Answer:**

```java
public class DirectMessage {
    private final String id;
    private final String senderId;
    private final String receiverId;
    private final String content;
    private final LocalDateTime timestamp;
    private boolean read;
}

public class MessageService {
    private final Map<String, List<DirectMessage>> conversations;
    
    public DirectMessage sendMessage(String senderId, String receiverId, 
                                    String content) {
        DirectMessage dm = new DirectMessage(senderId, receiverId, content);
        
        String conversationKey = getConversationKey(senderId, receiverId);
        conversations.computeIfAbsent(conversationKey, k -> new ArrayList<>())
            .add(dm);
        
        return dm;
    }
    
    public List<DirectMessage> getConversation(String user1, String user2) {
        String key = getConversationKey(user1, user2);
        return conversations.getOrDefault(key, Collections.emptyList());
    }
    
    private String getConversationKey(String user1, String user2) {
        // Consistent ordering
        return user1.compareTo(user2) < 0 
            ? user1 + ":" + user2 
            : user2 + ":" + user1;
    }
}
```

---

### Q3: How would you implement bookmarks?

**Answer:**

```java
public class User {
    private final List<String> bookmarkedTweetIds;
    
    public void bookmarkTweet(String tweetId) {
        if (!bookmarkedTweetIds.contains(tweetId)) {
            bookmarkedTweetIds.add(tweetId);
        }
    }
    
    public void unbookmarkTweet(String tweetId) {
        bookmarkedTweetIds.remove(tweetId);
    }
    
    public List<Tweet> getBookmarkedTweets(TwitterService service) {
        return bookmarkedTweetIds.stream()
            .map(service::getTweet)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

---

### Q4: What would you do differently with more time?

**Answer:**

1. **Add media support** - Images, videos, GIFs
2. **Add quote tweets** - Retweet with comment
3. **Add lists** - Curated groups of users
4. **Add mute/block** - User moderation
5. **Add analytics** - Tweet impressions, engagement rates
6. **Add scheduled tweets** - Post at specific time
7. **Add polls** - Interactive tweet type
8. **Add search functionality** - Full-text search of tweets

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `postTweet` | O(1) | Add to list |
| `follow/unfollow` | O(1) | Set operations |
| `likeTweet` | O(1) | Set add |
| `getTimeline` | O(n) | n = tweets to return |
| `getNewsFeed` | O(N log K) | N = tweets, K = followed users |
| `searchTweets` | O(T) | T = total tweets |

### Space Complexity

| Component | Space |
|-----------|-------|
| Users | O(U) |
| Tweets | O(T) |
| Followers per user | O(F) |
| Feed generation | O(K) | K = followed users |

### Bottlenecks at Scale

**10x Usage (1K → 10K users, 100 → 1K tweets/second):**
- Problem: Feed generation becomes slow (O(K × T) for merging), follower graph storage grows, tweet storage increases
- Solution: Pre-compute timelines, use caching layer (Redis) for feeds, implement tweet archiving
- Tradeoff: Additional infrastructure (Redis), cache invalidation complexity

**100x Usage (1K → 100K users, 100 → 10K tweets/second):**
- Problem: Single instance can't handle all users, feed generation too slow, real-time updates impossible
- Solution: Shard users by user ID, use distributed feed generation, implement push model for active users, pull model for inactive
- Tradeoff: Distributed system complexity, need feed aggregation and consistency handling


### Q1: How would you implement hashtags?

```java
public class HashtagService {
    private final Map<String, Set<String>> hashtagToTweets;
    
    public void indexTweet(Tweet tweet) {
        Set<String> hashtags = extractHashtags(tweet.getContent());
        for (String hashtag : hashtags) {
            hashtagToTweets.computeIfAbsent(hashtag.toLowerCase(), 
                k -> new LinkedHashSet<>()).add(tweet.getId());
        }
    }
    
    private Set<String> extractHashtags(String content) {
        Set<String> hashtags = new HashSet<>();
        Pattern pattern = Pattern.compile("#(\\w+)");
        Matcher matcher = pattern.matcher(content);
        while (matcher.find()) {
            hashtags.add(matcher.group(1));
        }
        return hashtags;
    }
    
    public List<Tweet> getTweetsByHashtag(String hashtag, int limit) {
        Set<String> tweetIds = hashtagToTweets.get(hashtag.toLowerCase());
        if (tweetIds == null) return Collections.emptyList();
        
        return tweetIds.stream()
            .limit(limit)
            .map(tweets::get)
            .collect(Collectors.toList());
    }
}
```

### Q2: How would you implement mentions?

```java
public class MentionService {
    public void processTweet(Tweet tweet) {
        Set<String> mentions = extractMentions(tweet.getContent());
        for (String username : mentions) {
            User mentioned = usersByUsername.get(username.toLowerCase());
            if (mentioned != null) {
                notifyUser(mentioned.getId(), 
                    "@" + tweet.getAuthorId() + " mentioned you!");
            }
        }
    }
    
    private Set<String> extractMentions(String content) {
        Set<String> mentions = new HashSet<>();
        Pattern pattern = Pattern.compile("@(\\w+)");
        Matcher matcher = pattern.matcher(content);
        while (matcher.find()) {
            mentions.add(matcher.group(1));
        }
        return mentions;
    }
}
```

### Q3: How would you implement trending topics?

```java
public class TrendingService {
    private final Map<String, AtomicInteger> hashtagCounts;
    private final ScheduledExecutorService scheduler;
    
    public TrendingService() {
        this.hashtagCounts = new ConcurrentHashMap<>();
        
        // Reset counts periodically
        scheduler.scheduleAtFixedRate(
            this::decayCounts, 1, 1, TimeUnit.HOURS);
    }
    
    public void recordHashtag(String hashtag) {
        hashtagCounts.computeIfAbsent(hashtag.toLowerCase(), 
            k -> new AtomicInteger(0)).incrementAndGet();
    }
    
    public List<String> getTrending(int limit) {
        return hashtagCounts.entrySet().stream()
            .sorted((a, b) -> b.getValue().get() - a.getValue().get())
            .limit(limit)
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
    }
    
    private void decayCounts() {
        hashtagCounts.forEach((k, v) -> {
            int newCount = (int) (v.get() * 0.5);  // 50% decay
            if (newCount == 0) {
                hashtagCounts.remove(k);
            } else {
                v.set(newCount);
            }
        });
    }
}
```

### Q4: How would you implement tweet replies/threads?

```java
public class Tweet {
    private String replyToId;
    private List<String> replyIds;
    
    public void addReply(String replyTweetId) {
        if (replyIds == null) {
            replyIds = new ArrayList<>();
        }
        replyIds.add(replyTweetId);
    }
}

public class TwitterService {
    public Tweet replyToTweet(String userId, String parentTweetId, 
                             String content) {
        Tweet parent = getTweet(parentTweetId);
        
        Tweet reply = new Tweet(userId, content);
        reply.setReplyToId(parentTweetId);
        
        parent.addReply(reply.getId());
        tweets.put(reply.getId(), reply);
        
        return reply;
    }
    
    public List<Tweet> getThread(String tweetId) {
        Tweet root = getTweet(tweetId);
        List<Tweet> thread = new ArrayList<>();
        thread.add(root);
        
        // Add all replies recursively
        collectReplies(root, thread);
        
        return thread;
    }
    
    private void collectReplies(Tweet tweet, List<Tweet> thread) {
        if (tweet.getReplyIds() == null) return;
        
        for (String replyId : tweet.getReplyIds()) {
            Tweet reply = tweets.get(replyId);
            if (reply != null) {
                thread.add(reply);
                collectReplies(reply, thread);
            }
        }
    }
}
```

### Q5: How would you handle celebrity users with millions of followers?

```java
// Fan-out on write vs Fan-out on read

// For regular users: Fan-out on write
// When user tweets, push to all followers' feeds

// For celebrities: Fan-out on read
// When follower requests feed, pull celebrity tweets

public class HybridFeedService {
    private static final int CELEBRITY_THRESHOLD = 100000;
    
    public void onTweet(Tweet tweet) {
        User author = getUser(tweet.getAuthorId());
        
        if (author.getFollowerCount() < CELEBRITY_THRESHOLD) {
            // Fan-out on write for regular users
            for (String followerId : author.getFollowers()) {
                pushToFeed(followerId, tweet);
            }
        }
        // Celebrities: their tweets are pulled on read
    }
    
    public List<Tweet> getNewsFeed(String userId) {
        List<Tweet> feed = new ArrayList<>();
        
        // Get pre-computed feed (from regular users)
        feed.addAll(getPrecomputedFeed(userId));
        
        // Pull celebrity tweets
        for (String followeeId : getFollowedCelebrities(userId)) {
            feed.addAll(getRecentTweets(followeeId));
        }
        
        // Merge and sort
        feed.sort((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()));
        
        return feed;
    }
}
```

