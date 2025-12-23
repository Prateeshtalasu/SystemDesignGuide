# News Feed (Facebook/Twitter) - Problem & Requirements

## What is a News Feed?

A News Feed is a personalized, continuously updating stream of content from people and pages a user follows. When you open Facebook or Twitter, the feed shows posts from friends, pages, and groups, ranked by relevance and recency.

**Example:**

```
User opens app
  ↓
System retrieves posts from 500 friends + 100 pages
  ↓
Ranks by relevance (engagement, recency, relationship)
  ↓
Returns top 50 posts for initial view
  ↓
User scrolls → fetch more posts
```

### Why Does This Exist?

1. **Information Overload**: Users follow hundreds of accounts; can't show everything
2. **Engagement**: Relevant content keeps users on platform longer
3. **Discovery**: Surface content users would miss otherwise
4. **Monetization**: Ads are inserted into the feed
5. **Real-time**: Users expect fresh content immediately

### What Breaks Without It?

- Users see stale or irrelevant content
- Engagement drops, users leave platform
- Ad revenue decreases
- Viral content doesn't spread
- Platform loses to competitors with better feeds

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question                                  | Why It Matters                           | Assumed Answer                                     |
| ----------------------------------------- | ---------------------------------------- | -------------------------------------------------- |
| What's the user scale?                    | Determines infrastructure size           | 500M DAU, 2B total users                           |
| What's the average follow count?          | Affects fan-out strategy                 | 200 friends/follows average                        |
| What content types?                       | Affects storage and rendering            | Text, images, videos, links                        |
| How fresh should the feed be?             | Affects architecture complexity          | New posts visible within 5 seconds                 |
| Do we need ranking or just chronological? | Ranking adds ML complexity               | Ranked feed with recency factor                    |
| Are there celebrities/influencers?        | "Celebrity problem" affects design       | Yes, some users have 10M+ followers                |
| Do we need real-time updates?             | Push vs pull architecture                | Yes, live updates for active users                 |

---

## Functional Requirements

### Core Features (Must Have)

1. **Post Creation**
   - Users can create posts (text, images, videos)
   - Posts are visible to followers
   - Support for privacy settings (public, friends, private)

2. **Feed Generation**
   - Show posts from followed users/pages
   - Rank posts by relevance
   - Support pagination (infinite scroll)

3. **Feed Refresh**
   - Pull-to-refresh for latest posts
   - Real-time updates for new posts
   - Show "X new posts" indicator

4. **Interactions**
   - Like, comment, share posts
   - Interactions update engagement scores
   - Interactions visible to friends

5. **Follow/Unfollow**
   - Follow users and pages
   - Unfollow removes from feed
   - Changes reflected in next feed refresh

### Secondary Features (Nice to Have)

6. **Stories**
   - Ephemeral content (24-hour expiry)
   - Separate from main feed

7. **Groups**
   - Posts from joined groups
   - Group-specific feeds

8. **Notifications**
   - Notify on interactions
   - Notify on posts from close friends

9. **Ads Integration**
   - Insert sponsored posts
   - Target based on user interests

---

## Non-Functional Requirements

### Performance

| Metric                 | Target            | Rationale                               |
| ---------------------- | ----------------- | --------------------------------------- |
| Feed load time         | < 500ms           | Users expect instant feed               |
| Post visibility        | < 5 seconds       | Near real-time for engagement           |
| Feed refresh           | < 200ms           | Smooth pull-to-refresh                  |
| Availability           | 99.99%            | Feed is core product                    |

### Scale

| Metric                 | Value             | Calculation                             |
| ---------------------- | ----------------- | --------------------------------------- |
| Daily Active Users     | 500 million       | Given assumption                        |
| Posts per day          | 500 million       | 1 post per DAU average                  |
| Feed requests per day  | 50 billion        | 100 feed loads per user per day         |
| Peak QPS              | 1 million         | 50B / 86,400 × 2 (peak factor)          |

### Reliability

- **No data loss**: Posts must be durable
- **Eventual consistency**: Feed can be slightly stale (seconds)
- **Graceful degradation**: Show cached feed if ranking fails

### Freshness

- New posts from friends: < 5 seconds
- New posts from pages: < 30 seconds
- Ranking updates: < 1 minute

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Messenger/DMs**: Direct messaging is separate system
2. **Search**: Content search is separate
3. **Ads Bidding**: Ad auction system is separate
4. **Content Moderation**: Spam/abuse detection is separate
5. **Analytics Dashboard**: Creator analytics is separate
6. **Video Streaming**: Video delivery is separate concern
7. **Stories**: Ephemeral content is separate feature

---

## The Celebrity Problem

### What is it?

When a celebrity with 10M followers posts:
- **Fan-out on write**: Write to 10M user feeds = expensive
- **Fan-out on read**: 10M users read from celebrity's feed = hot spot

### Why it matters

| User Type    | Followers | Posts/Day | Fan-out Cost |
| ------------ | --------- | --------- | ------------ |
| Regular      | 200       | 1         | 200 writes   |
| Influencer   | 100K      | 5         | 500K writes  |
| Celebrity    | 10M       | 3         | 30M writes   |

**At scale:** 1000 celebrities × 30M = 30 billion writes/day just for celebrities!

### Solution Preview

Hybrid approach:
- Regular users: Fan-out on write
- Celebrities: Fan-out on read
- Threshold: ~10K followers

---

## System Constraints

### Technical Constraints

1. **Feed Size**: Show ~50 posts initially, paginate
   - Can't load thousands of posts at once

2. **Ranking Latency**: Must rank quickly
   - Can't run complex ML on every request

3. **Storage**: Must store posts and feeds efficiently
   - Trade-off between precomputation and storage

### Business Constraints

1. **Global Service**: Users worldwide
2. **Mobile-First**: Most users on mobile
3. **Engagement Focus**: Optimize for time spent

---

## Success Metrics

How do we know if the system is working well?

| Metric                    | Target      | How to Measure                          |
| ------------------------- | ----------- | --------------------------------------- |
| Feed load success rate    | > 99.99%    | Successful loads / total requests       |
| P50 feed latency          | < 200ms     | Percentile from monitoring              |
| P99 feed latency          | < 500ms     | Percentile from monitoring              |
| Post visibility latency   | < 5s        | Time from post to appearing in feeds    |
| User engagement           | > 30 min/day| Average session time                    |

---

## User Stories

### Story 1: User Opens App

```
As a user,
I want to see my personalized feed when I open the app,
So that I can catch up on what my friends are doing.

Acceptance Criteria:
- Feed loads in under 500ms
- Shows relevant posts from friends and pages
- Most recent posts appear first (with ranking)
- Can scroll to load more posts
```

### Story 2: User Creates a Post

```
As a user,
I want to create a post with text and images,
So that my friends can see what I'm up to.

Acceptance Criteria:
- Post is saved immediately
- Post appears in my profile
- Post appears in friends' feeds within 5 seconds
- Friends are notified (if enabled)
```

### Story 3: Celebrity Posts Update

```
As a celebrity with millions of followers,
I want my posts to reach all my followers quickly,
So that I can engage with my audience.

Acceptance Criteria:
- Post is visible to followers within 30 seconds
- System handles load without degradation
- Engagement metrics are tracked
```

---

## Core Components Overview

### 1. Post Service
- Handles post creation and storage
- Manages post metadata
- Triggers fan-out

### 2. Feed Service
- Generates personalized feeds
- Handles feed requests
- Manages feed cache

### 3. Fan-out Service
- Distributes posts to follower feeds
- Handles celebrity vs regular fan-out
- Manages async processing

### 4. Ranking Service
- Scores posts for relevance
- Combines multiple signals
- Real-time and batch ranking

### 5. Graph Service
- Manages follow relationships
- Provides follower/following lists
- Handles graph queries

### 6. Timeline Cache
- Stores precomputed feeds
- Handles cache invalidation
- Manages feed pagination

---

## Fan-out Strategies

### Fan-out on Write (Push Model)

```
User A posts
  ↓
Get A's followers: [B, C, D, E, ...]
  ↓
Write post to each follower's feed cache
  ↓
When B opens app, feed is ready
```

**Pros:**
- Fast read (feed is precomputed)
- Simple read path

**Cons:**
- Slow write (must update all followers)
- Celebrity problem (10M writes)
- Wasted work if followers don't check feed

### Fan-out on Read (Pull Model)

```
User B opens app
  ↓
Get B's followings: [A, C, D, ...]
  ↓
Fetch recent posts from each
  ↓
Merge and rank
  ↓
Return feed
```

**Pros:**
- Fast write (just store post)
- No wasted work

**Cons:**
- Slow read (must fetch from many sources)
- Hot spots for popular users

### Hybrid Approach (Chosen)

```
Regular users (< 10K followers): Fan-out on write
Celebrities (> 10K followers): Fan-out on read

Feed = Precomputed feed + Real-time fetch from celebrities
```

---

## Interview Tips

### What Interviewers Look For

1. **Scale awareness**: Can you handle 500M DAU?
2. **Trade-off thinking**: Fan-out on write vs read
3. **Celebrity problem**: How do you handle it?
4. **Ranking understanding**: How do you order posts?

### Common Mistakes

1. **Ignoring celebrity problem**: This breaks naive designs
2. **Only fan-out on write**: Doesn't scale for celebrities
3. **Only fan-out on read**: Too slow for regular users
4. **Forgetting ranking**: Chronological isn't enough

### Good Follow-up Questions from Interviewer

- "What if a celebrity posts during Super Bowl?" (thundering herd)
- "How do you handle a user who follows 10K accounts?"
- "How fresh does the feed need to be?"

---

## Summary

| Aspect              | Decision                                       |
| ------------------- | ---------------------------------------------- |
| Primary use case    | Show personalized feed of posts from follows   |
| Scale               | 500M DAU, 50B feed requests/day                |
| Read:Write ratio    | 100:1 (feed reads vs post creates)             |
| Latency target      | < 200ms p50, < 500ms p99                       |
| Freshness target    | < 5 seconds for friend posts                   |
| Fan-out strategy    | Hybrid (write for regular, read for celebrity) |
| Ranking             | ML-based with engagement signals               |

