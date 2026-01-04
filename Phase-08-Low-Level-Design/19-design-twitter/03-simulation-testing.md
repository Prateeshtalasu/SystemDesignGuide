# 🐦 Design Twitter (Simplified) - Simulation & Testing

## STEP 5: Simulation / Dry Run

### Scenario 1: Happy Path - News Feed Generation

```
Users: Alice, Bob, Carol
Alice follows: Bob
Bob follows: Alice, Carol

Bob posts: "Hello World" (T1)
Carol posts: "Good morning" (T2)
Alice posts: "Hi everyone" (T3)

Alice's Feed: T1 (from Bob)
Bob's Feed: T3 (from Alice), T2 (from Carol)
Carol's Feed: (empty - follows no one)
```

**Final State:**
```
Feeds generated correctly
All tweets displayed in chronological order
All operations completed successfully
```

---

### Scenario 2: Failure/Invalid Input - Tweet Exceeds Character Limit

**Initial State:**
```
User: Alice
Tweet content: 281 characters (exceeds 280 limit)
```

**Step-by-step:**

1. `twitterService.postTweet("alice", longTweet)` (281 characters)
   - Validate tweet length: 281 > 280
   - Validation fails
   - Throws IllegalArgumentException("Tweet exceeds 280 character limit")
   - No tweet created
   - User's tweet list unchanged

2. `twitterService.postTweet("alice", null)` (invalid input)
   - Null content → throws IllegalArgumentException("Tweet content cannot be null")
   - No state change

3. `twitterService.postTweet(null, "Hello")` (invalid input)
   - Null user ID → throws IllegalArgumentException("User ID cannot be null")
   - No state change

**Final State:**
```
User Alice: No new tweet added
Tweet list unchanged
Invalid inputs properly rejected
```

---

### Scenario 3: Concurrency/Race Condition - Concurrent Feed Generation

**Initial State:**
```
User: Alice follows Bob, Charlie
Bob's tweets: [B3, B2, B1]
Charlie's tweets: [C2, C1]
Thread A: Get Alice's feed
Thread B: Bob posts new tweet B4 (concurrent)
Thread C: Get Alice's feed (concurrent with Thread B)
```

**Step-by-step (simulating concurrent operations):**

**Thread A:** `twitterService.getNewsFeed("alice")` at time T0
**Thread B:** `twitterService.postTweet("bob", "New tweet")` at time T0 (concurrent)
**Thread C:** `twitterService.getNewsFeed("alice")` at time T0 (concurrent)

1. **Thread A:** Starts feed generation
   - Gets Alice's following list: [Bob, Charlie]
   - Gets Bob's tweets: [B3, B2, B1]
   - Gets Charlie's tweets: [C2, C1]
   - Merges and sorts: [B3, C2, B2, C1, B1]
   - Returns feed

2. **Thread B:** Posts new tweet (concurrent)
   - Creates tweet B4
   - Adds to Bob's tweet list: [B4, B3, B2, B1]
   - ConcurrentHashMap ensures thread-safe update

3. **Thread C:** Starts feed generation (concurrent with Thread B)
   - Gets Alice's following list: [Bob, Charlie]
   - Gets Bob's tweets: May see [B4, B3, B2, B1] or [B3, B2, B1] depending on timing
   - Gets Charlie's tweets: [C2, C1]
   - Merges and sorts
   - Returns feed (may or may not include B4 depending on when Thread B completes)

**Final State:**
```
Thread A feed: [B3, C2, B2, C1, B1] (B4 may not be included)
Thread C feed: [B4, B3, C2, B2, C1, B1] or [B3, C2, B2, C1, B1] (depends on timing)
Bob's tweets: [B4, B3, B2, B1] (updated)
Thread-safe operations, no data corruption
```

---

## STEP 6: Edge Cases & Testing Strategy

### Boundary Conditions
- **Tweet > 280 chars**: Reject
- **Follow Self**: Reject
- **Unfollow Non-Followed**: No-op
- **Delete Non-Owned Tweet**: Reject

---

## Visual Trace: News Feed Generation

```
Alice follows: Bob, Charlie
Alice's tweets: [A3, A1]
Bob's tweets: [B4, B2]
Charlie's tweets: [C5, C3, C1]

Step 1: Initialize priority queue with first tweet from each
┌─────────────────────────────────────────────────────────────────┐
│ PQ: [C5, B4, A3]  (sorted by timestamp, newest first)          │
│ Iterators: [A→A1, B→B2, C→C3]                                  │
└─────────────────────────────────────────────────────────────────┘

Step 2: Extract C5, add next from Charlie
┌─────────────────────────────────────────────────────────────────┐
│ Feed: [C5]                                                      │
│ PQ: [B4, A3, C3]                                                │
│ Iterators: [A→A1, B→B2, C→C1]                                  │
└─────────────────────────────────────────────────────────────────┘

Step 3: Extract B4, add next from Bob
┌─────────────────────────────────────────────────────────────────┐
│ Feed: [C5, B4]                                                  │
│ PQ: [A3, C3, B2]                                                │
│ Iterators: [A→A1, B→empty, C→C1]                               │
└─────────────────────────────────────────────────────────────────┘

Step 4: Extract A3, add next from Alice
┌─────────────────────────────────────────────────────────────────┐
│ Feed: [C5, B4, A3]                                              │
│ PQ: [C3, B2, A1]                                                │
│ Iterators: [A→empty, B→empty, C→C1]                            │
└─────────────────────────────────────────────────────────────────┘

Continue until limit reached or PQ empty...

Final Feed (limit=5): [C5, B4, A3, C3, B2]
```

---

## Testing Approach

### Unit Tests

```java
// UserTest.java
public class UserTest {
    
    @Test
    void testFollowUnfollow() {
        User alice = new User("alice", "Alice");
        User bob = new User("bob", "Bob");
        
        alice.addFollowing(bob.getId());
        bob.addFollower(alice.getId());
        
        assertTrue(alice.isFollowing(bob.getId()));
        assertTrue(bob.getFollowers().contains(alice.getId()));
        
        alice.removeFollowing(bob.getId());
        bob.removeFollower(alice.getId());
        
        assertFalse(alice.isFollowing(bob.getId()));
    }
}

// TweetTest.java
public class TweetTest {
    
    @Test
    void testTweetLength() {
        assertThrows(IllegalArgumentException.class, () -> 
            new Tweet("user1", "a".repeat(281)));  // Over 280 chars
    }
    
    @Test
    void testLikes() {
        Tweet tweet = new Tweet("user1", "Hello!");
        
        tweet.addLike("user2");
        tweet.addLike("user3");
        tweet.addLike("user2");  // Duplicate
        
        assertEquals(2, tweet.getLikeCount());
        assertTrue(tweet.isLikedBy("user2"));
    }
}
```

### Integration Tests

```java
// TwitterServiceTest.java
public class TwitterServiceTest {
    
    private TwitterService twitter;
    
    @BeforeEach
    void setUp() {
        twitter = new TwitterService();
    }
    
    @Test
    void testNewsFeed() {
        User alice = twitter.createUser("alice", "Alice");
        User bob = twitter.createUser("bob", "Bob");
        
        twitter.follow(alice.getId(), bob.getId());
        
        twitter.postTweet(bob.getId(), "Bob's tweet 1");
        twitter.postTweet(alice.getId(), "Alice's tweet");
        twitter.postTweet(bob.getId(), "Bob's tweet 2");
        
        List<Tweet> feed = twitter.getNewsFeed(alice.getId());
        
        assertEquals(3, feed.size());
        assertEquals("Bob's tweet 2", feed.get(0).getContent());
        assertEquals("Alice's tweet", feed.get(1).getContent());
        assertEquals("Bob's tweet 1", feed.get(2).getContent());
    }
    
    @Test
    void testRetweet() {
        User alice = twitter.createUser("alice", "Alice");
        User bob = twitter.createUser("bob", "Bob");
        
        Tweet original = twitter.postTweet(alice.getId(), "Original tweet");
        Tweet retweet = twitter.retweet(bob.getId(), original.getId());
        
        assertTrue(retweet.isRetweet());
        assertEquals(original.getId(), retweet.getOriginalTweetId());
        assertEquals(1, original.getRetweetCount());
    }
}
```

### Feed Algorithm Tests

```java
// FeedServiceTest.java
public class FeedServiceTest {
    
    @Test
    void testFeedOrdering() {
        // Setup users and tweets with known timestamps
        // Verify feed returns tweets in correct order
    }
    
    @Test
    void testFeedLimit() {
        // Create many tweets
        // Verify feed respects limit
    }
    
    @Test
    void testFeedIncludesOwnTweets() {
        // User should see their own tweets in feed
    }
    
    @Test
    void testFeedExcludesUnfollowedUsers() {
        // After unfollow, their tweets shouldn't appear
    }
}
```


### postTweet

```
Time: O(1)
  - Create tweet: O(1)
  - Add to user's list: O(1) (prepend)
  - Store in map: O(1)

Space: O(1) for the operation
```

### getNewsFeed (Optimized)

```
Time: O(N log K)
  - K = number of followed users + 1 (self)
  - N = number of tweets to return
  - Each tweet: O(log K) for heap operations

Space: O(K)
  - Priority queue holds at most K entries
  - One per user being merged
```

### follow/unfollow

```
Time: O(1)
  - Set add/remove: O(1)

Space: O(1)
```

---

**Note:** Interview follow-ups have been moved to `02-design-explanation.md`, STEP 8.

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

### Q2: How would you implement direct messages?

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

### Q3: How would you implement bookmarks?

```java
public class User {
    private final List<String> bookmarkedTweetIds;
    
    public void bookmark(String tweetId) {
        if (!bookmarkedTweetIds.contains(tweetId)) {
            bookmarkedTweetIds.add(0, tweetId);  // Most recent first
        }
    }
    
    public void removeBookmark(String tweetId) {
        bookmarkedTweetIds.remove(tweetId);
    }
    
    public List<String> getBookmarks() {
        return Collections.unmodifiableList(bookmarkedTweetIds);
    }
}

public class TwitterService {
    public List<Tweet> getBookmarkedTweets(String userId) {
        User user = getUser(userId);
        return user.getBookmarks().stream()
            .map(tweets::get)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

### Q4: What would you do differently with more time?

1. **Add media support** - Images, videos, GIFs
2. **Add quote tweets** - Retweet with comment
3. **Add lists** - Curated groups of users
4. **Add mute/block** - User moderation
5. **Add analytics** - Tweet impressions, engagement rates
6. **Add scheduled tweets** - Post at specific time
7. **Add polls** - Interactive tweet type

