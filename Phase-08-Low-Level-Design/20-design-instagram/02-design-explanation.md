# 📸 Design Instagram (Simplified) - Design Explanation

## SOLID Principles Analysis

### 1. Single Responsibility Principle (SRP)

| Class | Responsibility | Reason for Change |
|-------|---------------|-------------------|
| `User` | Store user data and relationships | User model changes |
| `Post` | Store photo and engagement data | Post model changes |
| `Comment` | Store comment data | Comment model changes |
| `FeedService` | Generate user feeds | Feed algorithm changes |
| `InstagramService` | Coordinate all operations | Business logic changes |

**SRP in Action:**

```java
// User ONLY manages user data and relationships
public class User {
    private final Set<String> followers;
    private final Set<String> following;
    private final List<String> postIds;
    
    public void addFollower(String userId) { }
    public void addPost(String postId) { }
}

// Post ONLY manages post data and engagement
public class Post {
    private final Set<String> likes;
    private final List<Comment> comments;
    
    public void addLike(String userId) { }
    public void addComment(Comment comment) { }
}

// FeedService ONLY handles feed generation
public class FeedService {
    public List<Post> getNewsFeed(String userId, int limit) { }
    public List<Post> getExploreFeed(String userId, int limit) { }
}
```

---

### 2. Open/Closed Principle (OCP)

**Adding New Feed Types:**

```java
public interface FeedGenerator {
    List<Post> generate(String userId, int limit);
}

public class ChronologicalFeed implements FeedGenerator {
    @Override
    public List<Post> generate(String userId, int limit) {
        // Sort by timestamp
    }
}

public class RankedFeed implements FeedGenerator {
    @Override
    public List<Post> generate(String userId, int limit) {
        // Sort by engagement score
    }
}

public class ExploreFeed implements FeedGenerator {
    @Override
    public List<Post> generate(String userId, int limit) {
        // Show popular posts from non-followed users
    }
}
```

**Adding New Post Types:**

```java
public class Post {
    // Base post with image
}

public class CarouselPost extends Post {
    private List<String> imageUrls;  // Multiple images
}

public class ReelPost extends Post {
    private String videoUrl;
    private int durationSeconds;
}

public class StoryPost extends Post {
    private LocalDateTime expiresAt;  // 24 hours
}
```

---

### 3. Liskov Substitution Principle (LSP)

**All post types work interchangeably:**

```java
public class FeedService {
    public List<Post> getNewsFeed(String userId, int limit) {
        // Works with any Post subtype
        return posts.values().stream()
            .filter(p -> shouldIncludeInFeed(p, userId))
            .sorted(byTimestamp())
            .limit(limit)
            .collect(Collectors.toList());
    }
}
```

---

### 4. Interface Segregation Principle (ISP)

**Could be improved:**

```java
public interface Likeable {
    void addLike(String userId);
    void removeLike(String userId);
    int getLikeCount();
    boolean isLikedBy(String userId);
}

public interface Commentable {
    void addComment(Comment comment);
    void removeComment(String commentId);
    List<Comment> getComments();
}

public class Post implements Likeable, Commentable { }
public class Comment implements Likeable { }  // Comments can be liked too
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

public interface PostRepository {
    void save(Post post);
    Post findById(String id);
    List<Post> findByAuthor(String userId);
}

public interface FeedGenerator {
    List<Post> generate(String userId, int limit);
}

public class InstagramService {
    private final UserRepository userRepo;
    private final PostRepository postRepo;
    private final FeedGenerator feedGenerator;
    
    public InstagramService(UserRepository users, PostRepository posts,
                           FeedGenerator feed) {
        this.userRepo = users;
        this.postRepo = posts;
        this.feedGenerator = feed;
    }
}
```

**Why We Use Concrete Classes in This LLD Implementation:**

For low-level design interviews, we intentionally use concrete classes instead of repository interfaces for the following reasons:

1. **In-Memory Data Structures**: The system operates on in-memory data structures (users, posts, feeds). Repository interfaces are more relevant for persistent storage, which is often out of scope for LLD.

2. **Core Algorithm Focus**: LLD interviews focus on feed generation algorithms, post creation logic, and follower management. Adding repository abstractions shifts focus away from these core concepts.

3. **Single Implementation**: There's no requirement for multiple data access or feed generation implementations in the interview context. The abstraction doesn't provide value for demonstrating LLD skills.

4. **Production vs Interview**: In production systems, we would absolutely extract `UserRepository`, `PostRepository`, and `FeedGenerator` interfaces for:
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
| **SRP** | PASS | Each class has a single, well-defined responsibility. User stores user data, Post stores post data, FeedService generates feeds, InstagramService coordinates. Clear separation. | N/A | - |
| **OCP** | PASS | System is open for extension (new post types, feed algorithms) without modifying existing code. Factory pattern and strategy pattern enable this. | N/A | - |
| **LSP** | PASS | All post types properly implement the post contract. They are substitutable. | N/A | - |
| **ISP** | PASS | Interfaces are minimal and focused. No unused methods forced on clients. | N/A | - |
| **DIP** | ACCEPTABLE (LLD Scope) | InstagramService depends on concrete classes. For LLD interview scope, this is acceptable as it focuses on core feed generation and post management algorithms. In production, we would depend on UserRepository, PostRepository, and FeedGenerator interfaces. | See "Why We Use Concrete Classes" section above for detailed justification. This is an intentional design decision for interview context, not an oversight. | Interview: Simpler, focuses on core LLD skills. Production: More abstraction layers, but improves testability and data access flexibility |

---

## Why Alternatives Were Rejected

### Alternative 1: Single Post List with Linear Filtering

**What it is:**

```java
// Rejected approach
public class InstagramService {
    private List<Post> allPosts;  // Single list of all posts
    
    public List<Post> getNewsFeed(String userId, int limit) {
        User user = getUser(userId);
        Set<String> followingIds = user.getFollowing();
        
        // Filter all posts linearly
        return allPosts.stream()
            .filter(p -> followingIds.contains(p.getAuthorId()))
            .sorted((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()))
            .limit(limit)
            .collect(Collectors.toList());
    }
}
```

**Why rejected:**
- O(P) time complexity for every feed request, where P = total posts in system
- Does not scale as system grows - performance degrades linearly with total posts
- Inefficient memory access pattern - scans entire dataset for each user's feed
- Cannot leverage user-specific post lists for optimization

**What breaks:**
- Feed generation becomes extremely slow with millions of posts
- High CPU usage for every feed request
- Poor user experience with slow page loads
- System cannot handle concurrent feed requests efficiently

---

### Alternative 2: Bidirectional Follow Graph as Single Data Structure

**What it is:**

```java
// Rejected approach
public class FollowGraph {
    private Map<String, Set<String>> bidirectionalEdges;  // Stores both directions
    
    public void follow(String followerId, String followeeId) {
        bidirectionalEdges.computeIfAbsent(followerId, k -> new HashSet<>()).add(followeeId);
        bidirectionalEdges.computeIfAbsent(followeeId, k -> new HashSet<>()).add(followerId);
    }
    
    public Set<String> getFollowers(String userId) {
        return bidirectionalEdges.getOrDefault(userId, Collections.emptySet());
    }
    
    public Set<String> getFollowing(String userId) {
        return bidirectionalEdges.getOrDefault(userId, Collections.emptySet());  // Same data
    }
}
```

**Why rejected:**
- Violates single responsibility - User class should own its own relationships
- Confuses followers and following in same data structure
- Makes User class dependent on external graph structure
- Harder to maintain consistency when user relationships change
- Cannot easily track user-specific metadata (when followed, notifications, etc.)

**What breaks:**
- User model becomes tightly coupled to graph structure
- Difficult to add user-specific relationship metadata
- Unclear ownership of relationship data
- Harder to implement features like "follow suggestions" or "mutual friends"
- Violates encapsulation - user relationships should be part of User class

---

## Design Patterns Used

### 1. Builder Pattern (Potential)

**Where:** Creating posts with optional fields

```java
public class Post {
    public static class Builder {
        private String authorId;
        private String imageUrl;
        private String caption;
        private String location;
        private List<String> tags;
        
        public Builder author(String authorId) {
            this.authorId = authorId;
            return this;
        }
        
        public Builder image(String url) {
            this.imageUrl = url;
            return this;
        }
        
        public Builder caption(String caption) {
            this.caption = caption;
            return this;
        }
        
        public Builder location(String location) {
            this.location = location;
            return this;
        }
        
        public Post build() {
            return new Post(this);
        }
    }
}

// Usage
Post post = new Post.Builder()
    .author(userId)
    .image("https://example.com/photo.jpg")
    .caption("Beautiful day! #sunshine")
    .location("Central Park, NYC")
    .build();
```

---

### 2. Strategy Pattern

**Where:** Feed generation algorithms

```java
public interface FeedStrategy {
    List<Post> generate(String userId, int limit);
}

public class ChronologicalStrategy implements FeedStrategy { }
public class RankedStrategy implements FeedStrategy { }
public class ExploreStrategy implements FeedStrategy { }

public class FeedService {
    private FeedStrategy strategy;
    
    public void setStrategy(FeedStrategy strategy) {
        this.strategy = strategy;
    }
    
    public List<Post> getFeed(String userId, int limit) {
        return strategy.generate(userId, limit);
    }
}
```

---

### 3. Observer Pattern (Potential)

**Where:** Notifications for likes, comments, follows

```java
public interface InstagramObserver {
    void onNewFollower(User follower, User followee);
    void onLike(User liker, Post post);
    void onComment(User commenter, Post post, Comment comment);
    void onMention(User mentioner, Post post, User mentioned);
}

public class NotificationService implements InstagramObserver {
    @Override
    public void onNewFollower(User follower, User followee) {
        sendNotification(followee.getId(), 
            "@" + follower.getUsername() + " started following you");
    }
    
    @Override
    public void onLike(User liker, Post post) {
        sendNotification(post.getAuthorId(),
            "@" + liker.getUsername() + " liked your post");
    }
}
```

---

### 4. Composite Pattern (Potential)

**Where:** Carousel posts with multiple images

```java
public interface MediaContent {
    String getUrl();
    String getType();
}

public class ImageContent implements MediaContent {
    private final String imageUrl;
    
    @Override
    public String getType() { return "image"; }
}

public class VideoContent implements MediaContent {
    private final String videoUrl;
    private final int duration;
    
    @Override
    public String getType() { return "video"; }
}

public class CarouselContent implements MediaContent {
    private final List<MediaContent> items;
    
    @Override
    public String getType() { return "carousel"; }
}
```

---

## Feed Generation Algorithms

### Chronological Feed

```java
public List<Post> getChronologicalFeed(String userId, int limit) {
    Set<String> followingIds = getUser(userId).getFollowing();
    
    return posts.values().stream()
        .filter(p -> followingIds.contains(p.getAuthorId()))
        .sorted((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()))
        .limit(limit)
        .collect(Collectors.toList());
}
```

### Ranked Feed (Engagement-based)

```java
public List<Post> getRankedFeed(String userId, int limit) {
    Set<String> followingIds = getUser(userId).getFollowing();
    
    return posts.values().stream()
        .filter(p -> followingIds.contains(p.getAuthorId()))
        .filter(p -> isRecent(p, 7))  // Last 7 days
        .sorted((a, b) -> calculateScore(b) - calculateScore(a))
        .limit(limit)
        .collect(Collectors.toList());
}

private int calculateScore(Post post) {
    int likes = post.getLikeCount();
    int comments = post.getCommentCount() * 2;  // Comments worth more
    int recency = getRecencyBonus(post);
    return likes + comments + recency;
}
```

### Explore Feed

```java
public List<Post> getExploreFeed(String userId, int limit) {
    Set<String> followingIds = getUser(userId).getFollowing();
    
    return posts.values().stream()
        // Exclude posts from followed users
        .filter(p -> !followingIds.contains(p.getAuthorId()))
        .filter(p -> !p.getAuthorId().equals(userId))
        // Sort by engagement
        .sorted((a, b) -> calculateEngagement(b) - calculateEngagement(a))
        .limit(limit)
        .collect(Collectors.toList());
}
```

---

## Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `uploadPhoto` | O(t) | t = tags to extract |
| `follow/unfollow` | O(1) | Set operations |
| `likePost` | O(1) | Set add |
| `addComment` | O(1) | List add |
| `getUserPosts` | O(n) | n = posts to return |
| `getNewsFeed` | O(P log P) | P = total posts from followed |
| `getExploreFeed` | O(T log T) | T = total posts |

### Space Complexity

| Component | Space |
|-----------|-------|
| Users | O(U) |
| Posts | O(P) |
| Comments | O(C) |
| Followers per user | O(F) |
| Feed generation | O(K) | K = followed users |

---

## STEP 8: Interviewer Follow-ups with Answers

### Q1: How would you implement Stories?

```java
public class Story {
    private final String id;
    private final String authorId;
    private final String mediaUrl;
    private final LocalDateTime createdAt;
    private final LocalDateTime expiresAt;
    private final Set<String> viewers;
    
    public Story(String authorId, String mediaUrl) {
        this.id = generateId();
        this.authorId = authorId;
        this.mediaUrl = mediaUrl;
        this.createdAt = LocalDateTime.now();
        this.expiresAt = createdAt.plusHours(24);
        this.viewers = new HashSet<>();
    }
    
    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }
    
    public void markViewed(String userId) {
        viewers.add(userId);
    }
}

public class StoryService {
    private final Map<String, List<Story>> userStories;
    
    public List<Story> getStoriesForFeed(String userId) {
        Set<String> following = getUser(userId).getFollowing();
        
        return following.stream()
            .flatMap(followeeId -> getUserStories(followeeId).stream())
            .filter(s -> !s.isExpired())
            .filter(s -> !s.getViewers().contains(userId))
            .collect(Collectors.toList());
    }
    
    // Cleanup expired stories periodically
    public void cleanupExpiredStories() {
        userStories.values().forEach(stories -> 
            stories.removeIf(Story::isExpired));
    }
}
```

### Q2: How would you implement saved posts?

```java
public class User {
    private final List<String> savedPostIds;
    private final Map<String, List<String>> collections;  // name -> postIds
    
    public void savePost(String postId) {
        if (!savedPostIds.contains(postId)) {
            savedPostIds.add(0, postId);
        }
    }
    
    public void saveToCollection(String postId, String collectionName) {
        savePost(postId);
        collections.computeIfAbsent(collectionName, k -> new ArrayList<>())
            .add(postId);
    }
    
    public void unsavePost(String postId) {
        savedPostIds.remove(postId);
        collections.values().forEach(posts -> posts.remove(postId));
    }
}
```

### Q3: How would you implement location-based features?

```java
public class LocationService {
    private final Map<String, List<String>> locationToPosts;  // location -> postIds
    
    public void indexPost(Post post) {
        if (post.getLocation() != null) {
            String normalizedLocation = normalizeLocation(post.getLocation());
            locationToPosts.computeIfAbsent(normalizedLocation, k -> new ArrayList<>())
                .add(post.getId());
        }
    }
    
    public List<Post> getPostsByLocation(String location, int limit) {
        String normalized = normalizeLocation(location);
        List<String> postIds = locationToPosts.getOrDefault(normalized, 
            Collections.emptyList());
        
        return postIds.stream()
            .map(posts::get)
            .filter(Objects::nonNull)
            .sorted((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()))
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    public List<String> getNearbyLocations(double lat, double lng, double radiusKm) {
        // Would use geospatial indexing in production
        // For simplicity, return locations that match
        return locationToPosts.keySet().stream()
            .filter(loc -> isNearby(loc, lat, lng, radiusKm))
            .collect(Collectors.toList());
    }
}
```

### Q4: How would you implement activity feed?

```java
public class Activity {
    private final String id;
    private final ActivityType type;
    private final String actorId;
    private final String targetId;
    private final String postId;
    private final LocalDateTime timestamp;
}

public enum ActivityType {
    LIKE, COMMENT, FOLLOW, MENTION
}

public class ActivityService {
    private final Map<String, List<Activity>> userActivities;
    
    public void recordActivity(Activity activity) {
        // Add to target user's activity feed
        userActivities.computeIfAbsent(activity.getTargetId(), 
            k -> new ArrayList<>()).add(0, activity);
    }
    
    public List<Activity> getActivityFeed(String userId, int limit) {
        return userActivities.getOrDefault(userId, Collections.emptyList())
            .stream()
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    // Group similar activities
    public List<GroupedActivity> getGroupedActivityFeed(String userId) {
        List<Activity> activities = getActivityFeed(userId, 100);
        
        // Group by type and post
        Map<String, List<Activity>> grouped = activities.stream()
            .collect(Collectors.groupingBy(a -> 
                a.getType() + ":" + a.getPostId()));
        
        return grouped.entrySet().stream()
            .map(e -> new GroupedActivity(e.getValue()))
            .sorted((a, b) -> b.getLatestTime().compareTo(a.getLatestTime()))
            .collect(Collectors.toList());
    }
}
```

### Q5: How would you implement carousel posts (multiple images)?

**Answer:**

```java
public class CarouselPost extends Post {
    private final List<String> imageUrls;
    private int currentIndex;
    
    public CarouselPost(String authorId, List<String> imageUrls, String caption) {
        super(authorId, imageUrls.get(0), caption);  // First image as primary
        this.imageUrls = new ArrayList<>(imageUrls);
        this.currentIndex = 0;
    }
    
    public List<String> getImageUrls() {
        return Collections.unmodifiableList(imageUrls);
    }
    
    public int getImageCount() {
        return imageUrls.size();
    }
    
    public String getCurrentImage() {
        return imageUrls.get(currentIndex);
    }
    
    public void setCurrentIndex(int index) {
        if (index >= 0 && index < imageUrls.size()) {
            this.currentIndex = index;
        }
    }
}

public class InstagramService {
    public Post uploadCarouselPost(String userId, List<String> imageUrls, String caption) {
        if (imageUrls == null || imageUrls.isEmpty()) {
            throw new IllegalArgumentException("Carousel must have at least one image");
        }
        if (imageUrls.size() > 10) {
            throw new IllegalArgumentException("Carousel cannot exceed 10 images");
        }
        
        User user = getUser(userId);
        CarouselPost post = new CarouselPost(userId, imageUrls, caption);
        posts.put(post.getId(), post);
        user.addPost(post.getId());
        return post;
    }
}
```

### Q6: How would you implement user mentions in comments and captions?

**Answer:**

```java
public class MentionExtractor {
    private static final Pattern MENTION_PATTERN = Pattern.compile("@(\\w+)");
    
    public List<String> extractMentions(String text) {
        List<String> mentions = new ArrayList<>();
        if (text == null) return mentions;
        
        Matcher matcher = MENTION_PATTERN.matcher(text);
        while (matcher.find()) {
            mentions.add(matcher.group(1).toLowerCase());
        }
        return mentions;
    }
}

public class Post {
    private final List<String> mentions;
    
    private List<String> extractMentions(String text) {
        MentionExtractor extractor = new MentionExtractor();
        return extractor.extractMentions(text);
    }
    
    public Post(String authorId, String imageUrl, String caption) {
        // ... existing code ...
        this.mentions = extractMentions(caption);
    }
    
    public List<String> getMentions() {
        return Collections.unmodifiableList(mentions);
    }
}

public class Comment {
    private final List<String> mentions;
    
    public Comment(String authorId, String postId, String text) {
        // ... existing code ...
        this.mentions = new MentionExtractor().extractMentions(text);
    }
    
    public List<String> getMentions() {
        return Collections.unmodifiableList(mentions);
    }
}

public class InstagramService {
    public void notifyMentions(Post post) {
        for (String username : post.getMentions()) {
            try {
                User mentionedUser = getUserByUsername(username);
                // Send notification to mentioned user
            } catch (IllegalArgumentException e) {
                // User doesn't exist, skip
            }
        }
    }
    
    public List<Post> getPostsMentioningUser(String userId) {
        User user = getUser(userId);
        String username = user.getUsername().toLowerCase();
        
        return posts.values().stream()
            .filter(p -> p.getMentions().contains(username))
            .sorted((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()))
            .collect(Collectors.toList());
    }
}
```

### Q7: How would you implement search functionality (users, hashtags, posts)?

**Answer:**

```java
public class SearchService {
    private final Map<String, User> users;
    private final Map<String, Post> posts;
    private final Map<String, Set<String>> hashtagIndex;  // tag -> postIds
    
    public SearchService(Map<String, User> users, Map<String, Post> posts) {
        this.users = users;
        this.posts = posts;
        this.hashtagIndex = new HashMap<>();
        buildHashtagIndex();
    }
    
    private void buildHashtagIndex() {
        for (Post post : posts.values()) {
            for (String tag : post.getTags()) {
                hashtagIndex.computeIfAbsent(tag, k -> new HashSet<>())
                    .add(post.getId());
            }
        }
    }
    
    public List<User> searchUsers(String query, int limit) {
        String lowerQuery = query.toLowerCase();
        
        return users.values().stream()
            .filter(u -> u.getUsername().toLowerCase().contains(lowerQuery) ||
                        u.getDisplayName().toLowerCase().contains(lowerQuery))
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    public List<String> searchHashtags(String query, int limit) {
        String lowerQuery = query.toLowerCase();
        
        return hashtagIndex.keySet().stream()
            .filter(tag -> tag.contains(lowerQuery))
            .sorted((a, b) -> {
                // Sort by popularity (number of posts)
                int countA = hashtagIndex.get(a).size();
                int countB = hashtagIndex.get(b).size();
                return countB - countA;
            })
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    public List<Post> searchPosts(String query, int limit) {
        String lowerQuery = query.toLowerCase();
        
        return posts.values().stream()
            .filter(p -> p.getCaption().toLowerCase().contains(lowerQuery))
            .sorted((a, b) -> b.getTimestamp().compareTo(a.getTimestamp()))
            .limit(limit)
            .collect(Collectors.toList());
    }
}
```

### Q8: How would you implement blocking and muting users?

**Answer:**

```java
public class User {
    private final Set<String> blockedUsers;  // Users this user blocked
    private final Set<String> mutedUsers;    // Users this user muted
    
    public User(String username, String displayName) {
        // ... existing code ...
        this.blockedUsers = new HashSet<>();
        this.mutedUsers = new HashSet<>();
    }
    
    public void blockUser(String userId) {
        blockedUsers.add(userId);
        // Also unfollow if following
        following.remove(userId);
    }
    
    public void unblockUser(String userId) {
        blockedUsers.remove(userId);
    }
    
    public void muteUser(String userId) {
        mutedUsers.add(userId);
    }
    
    public void unmuteUser(String userId) {
        mutedUsers.remove(userId);
    }
    
    public boolean isBlocked(String userId) {
        return blockedUsers.contains(userId);
    }
    
    public boolean isMuted(String userId) {
        return mutedUsers.contains(userId);
    }
    
    public Set<String> getBlockedUsers() {
        return Collections.unmodifiableSet(blockedUsers);
    }
    
    public Set<String> getMutedUsers() {
        return Collections.unmodifiableSet(mutedUsers);
    }
}

public class InstagramService {
    public void blockUser(String blockerId, String blockedId) {
        User blocker = getUser(blockerId);
        getUser(blockedId);  // Validate blocked user exists
        
        blocker.blockUser(blockedId);
        // Also remove from blocked user's followers/following
        User blocked = getUser(blockedId);
        blocked.removeFollower(blockerId);
        blocked.removeFollowing(blockerId);
    }
    
    public void muteUser(String muterId, String mutedId) {
        User muter = getUser(muterId);
        getUser(mutedId);  // Validate muted user exists
        muter.muteUser(mutedId);
    }
    
    public List<Post> getNewsFeed(String userId, int limit) {
        User user = getUser(userId);
        Set<String> blockedAndMuted = new HashSet<>();
        blockedAndMuted.addAll(user.getBlockedUsers());
        blockedAndMuted.addAll(user.getMutedUsers());
        
        // Filter out posts from blocked/muted users
        return feedService.getNewsFeed(userId, limit * 2).stream()
            .filter(p -> !blockedAndMuted.contains(p.getAuthorId()))
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    public void follow(String followerId, String followeeId) {
        User follower = getUser(followerId);
        User followee = getUser(followeeId);
        
        // Check if either user blocked the other
        if (follower.isBlocked(followeeId) || followee.isBlocked(followerId)) {
            throw new IllegalStateException("Cannot follow blocked user");
        }
        
        // ... existing follow logic ...
    }
}
```

---

## STEP 7: Complexity Analysis

### Time Complexity

| Operation | Complexity | Explanation |
|-----------|------------|-------------|
| `uploadPhoto` | O(w) | w = words in caption (for tag extraction) |
| `follow/unfollow` | O(1) | Set operations |
| `likePost` | O(1) | Set add |
| `addComment` | O(1) | List add |
| `getUserPosts` | O(n) | n = posts to return |
| `getNewsFeed` | O(P log P) | P = total posts from followed users |
| `getExploreFeed` | O(T log T) | T = total posts |
| `getPostsByTag` | O(T) | T = total posts (linear scan) |

### Space Complexity

| Component | Space | Notes |
|-----------|-------|-------|
| Users | O(U) | U = number of users |
| Posts | O(P) | P = number of posts |
| Comments | O(C) | C = total comments |
| Followers per user | O(F) | F = average followers |
| Feed generation | O(K) | K = posts from followed users |

### Bottlenecks at Scale

**10x Usage (1K → 10K users, 100 → 1K posts/second):**
- Problem: Feed generation becomes slow (O(K × P) for merging), follower graph storage grows, post storage increases, hashtag indexing overhead
- Solution: Pre-compute feeds, use caching layer (Redis) for feeds, implement post archiving, optimize hashtag indexing
- Tradeoff: Additional infrastructure (Redis), cache invalidation complexity

**100x Usage (1K → 100K users, 100 → 10K posts/second):**
- Problem: Single instance can't handle all users, feed generation too slow, real-time updates impossible, hashtag search becomes bottleneck
- Solution: Shard users by user ID, use distributed feed generation, implement push model for active users, use distributed search index (Elasticsearch) for hashtags
- Tradeoff: Distributed system complexity, need feed aggregation and consistency handling

### With Inverted Index for Hashtags

| Operation | Without Index | With Index |
|-----------|---------------|------------|
| `getPostsByTag` | O(T) | O(M) where M = posts with that tag |
| Index space | - | O(T × avg_tags) |
```

