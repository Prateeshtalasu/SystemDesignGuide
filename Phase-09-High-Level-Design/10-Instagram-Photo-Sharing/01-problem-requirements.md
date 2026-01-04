# Instagram / Photo Sharing - Problem & Requirements

## Problem Statement

Design a photo sharing platform that allows users to upload, share, and discover photos. The system should support a personalized feed, stories (ephemeral content), image processing, and social features like likes and comments. The platform must handle billions of photos and serve millions of concurrent users globally.

---

## Functional Requirements

### Core Features

| Feature                  | Description                                           | Priority |
| ------------------------ | ----------------------------------------------------- | -------- |
| **Photo Upload**         | Upload photos with filters and captions               | P0       |
| **Photo Feed**           | Personalized feed of photos from followed users       | P0       |
| **Follow/Unfollow**      | Follow other users to see their content               | P0       |
| **Like/Comment**         | Engage with photos                                    | P0       |
| **User Profile**         | View user's photos and information                    | P0       |
| **Search**               | Search users, hashtags, locations                     | P0       |

### Extended Features

| Feature                  | Description                                           | Priority |
| ------------------------ | ----------------------------------------------------- | -------- |
| **Stories**              | Ephemeral photos/videos (24-hour lifespan)            | P1       |
| **Direct Messages**      | Private messaging between users                       | P1       |
| **Explore/Discover**     | Discover new content and users                        | P1       |
| **Notifications**        | Alerts for likes, comments, follows                   | P1       |
| **Hashtags**             | Tag and discover content by topic                     | P1       |
| **Location Tags**        | Tag photos with location                              | P2       |
| **Video Posts**          | Short video content                                   | P2       |

---

## Non-Functional Requirements

| Requirement      | Target                | Rationale                                      |
| ---------------- | --------------------- | ---------------------------------------------- |
| **Latency**      | < 200ms feed load     | Fast, responsive experience                    |
| **Throughput**   | 100K uploads/second   | Peak upload during events                      |
| **Availability** | 99.99%                | Social platform must be always available       |
| **Durability**   | 99.999999999%         | Photos must never be lost                      |
| **Scalability**  | 2B users, 500M DAU    | Instagram scale                                |
| **Consistency**  | Eventual (feed)       | Feed can be slightly stale                     |

---

## User Stories

### Content Creator

1. **As a user**, I want to upload photos with filters so I can share visually appealing content.

2. **As a user**, I want to add captions and hashtags so my content is discoverable.

3. **As a user**, I want to see who liked and commented on my photos so I can engage with my audience.

### Content Consumer

1. **As a user**, I want a personalized feed so I see content from people I follow.

2. **As a user**, I want to discover new content so I can find interesting accounts to follow.

3. **As a user**, I want to view stories so I can see ephemeral updates from friends.

4. **As a user**, I want to search by hashtag so I can find content on specific topics.

---

## Constraints & Assumptions

### Constraints

1. **Image Size**: Max 10MB per photo
2. **Storage**: Billions of photos, petabytes of storage
3. **Mobile-first**: Optimize for mobile experience
4. **Real-time**: Feed should feel current

### Assumptions

1. Average photo size: 2MB (original), 200KB (optimized)
2. Average user follows 200 accounts
3. Average user posts 1 photo/week
4. 80% of traffic is feed consumption
5. Stories viewed within 24 hours of posting

---

## Out of Scope

1. Reels/long-form video
2. Shopping/e-commerce features
3. Advertising platform
4. Content moderation ML
5. IGTV

---

## Key Challenges

### 1. Feed Generation

**Challenge:** Generate personalized feed for 500M daily users

**Solution:**
- Hybrid fan-out (write for regular users, read for celebrities)
- Pre-computed feeds for active users
- Ranking algorithm for relevance

### 2. Image Processing

**Challenge:** Process millions of uploads with filters and resizing

**Solution:**
- Distributed image processing pipeline
- Multiple resolution generation
- CDN for delivery

### 3. Stories (Ephemeral Content)

**Challenge:** Content that expires after 24 hours

**Solution:**
- Separate storage with TTL
- Efficient cleanup jobs
- Quick access for active stories

### 4. Celebrity Problem

**Challenge:** Users with millions of followers

**Solution:**
- Fan-out on read for celebrities
- Dedicated infrastructure
- Caching celebrity content

---

## Success Metrics

| Metric              | Definition                              | Target        |
| ------------------- | --------------------------------------- | ------------- |
| **Feed Load Time**  | Time to display first photo             | < 500ms       |
| **Upload Success**  | % of uploads completing                 | > 99.9%       |
| **Engagement Rate** | Likes + Comments / Impressions          | > 3%          |
| **Story Views**     | % of followers viewing stories          | > 30%         |
| **DAU/MAU Ratio**   | Daily to monthly active users           | > 50%         |

---

## Photo Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          PHOTO PROCESSING PIPELINE                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

Upload                  Processing                              Storage
  │                         │                                      │
  ▼                         ▼                                      ▼
┌─────────┐           ┌─────────────┐                        ┌─────────────┐
│ Original│           │   Image     │                        │  Processed  │
│  Photo  │──────────>│  Processor  │───────────────────────>│   Images    │
│  (5MB)  │           │             │                        │             │
└─────────┘           └─────────────┘                        └─────────────┘
                            │                                      │
                            │                                      │
              ┌─────────────┼─────────────┐                       │
              ▼             ▼             ▼                       │
        ┌─────────┐   ┌─────────┐   ┌─────────┐                  │
        │ Thumb   │   │ Medium  │   │ Large   │                  │
        │ 150x150 │   │ 640x640 │   │ 1080px  │──────────────────┘
        │ (10KB)  │   │ (100KB) │   │ (300KB) │
        └─────────┘   └─────────┘   └─────────┘
              │             │             │
              └─────────────┴─────────────┘
                            │
                            ▼
                    ┌─────────────┐
                    │     CDN     │
                    │ Distribution│
                    └─────────────┘

Also generates:
- Blurhash placeholder (for loading)
- EXIF extraction (location, camera)
- Content hash (deduplication)
```

---

## Feed Types

### Home Feed

```
Content from users you follow
Ranked by:
- Recency
- Engagement (likes, comments)
- Relationship (interaction history)
- Content type preference
```

### Explore Feed

```
Content you might like
Based on:
- Similar users' interests
- Trending content
- Content similar to what you've liked
- Location-based recommendations
```

### Stories Tray

```
Stories from users you follow
Ordered by:
- Unseen stories first
- Recency
- Relationship strength
```

---

## Data Model Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          CORE ENTITIES                                               │
└─────────────────────────────────────────────────────────────────────────────────────┘

User
├── user_id
├── username
├── profile_photo_url
├── bio
├── follower_count
├── following_count
└── post_count

Post (Photo)
├── post_id
├── user_id
├── image_urls (multiple sizes)
├── caption
├── location
├── hashtags
├── like_count
├── comment_count
├── created_at
└── is_archived

Story
├── story_id
├── user_id
├── media_url
├── media_type (photo/video)
├── created_at
├── expires_at
└── view_count

Follow
├── follower_id
├── following_id
└── created_at

Like
├── user_id
├── post_id
└── created_at

Comment
├── comment_id
├── post_id
├── user_id
├── text
└── created_at
```

---

## Summary

| Aspect              | Decision                                              |
| ------------------- | ----------------------------------------------------- |
| Feed strategy       | Hybrid fan-out (write for regular, read for celebrities)|
| Image storage       | Object storage with CDN                               |
| Image processing    | Multiple resolutions, lazy filter application         |
| Stories             | Separate storage with 24-hour TTL                     |
| Scale               | 2B users, 500M DAU, 100K uploads/second               |

