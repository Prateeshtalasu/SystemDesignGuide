# Back-of-Envelope Calculations

## 0️⃣ Prerequisites

Before diving into back-of-envelope calculations, you should understand:

- **Basic Math**: Comfortable with powers of 10, multiplication, and division
- **System Components**: Familiarity with databases, caches, and networks (covered in Phases 1-6)
- **Requirements Clarification**: Understanding what scale means in system design (covered in Topic 2 of this phase)

Quick refresher: Back-of-envelope calculations are rough estimates done quickly to validate design decisions. They're called "back-of-envelope" because you should be able to do them on a napkin or envelope, without a calculator. The goal is not precision but rather getting the right order of magnitude (10x, 100x, 1000x).

*Reference*: This topic builds on Phase 1, Topic 11 (Back-of-Envelope Calculations) with interview-specific focus.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

System design interviews require you to make decisions about infrastructure: How many servers? How much storage? What database? These decisions must be grounded in reality, not guesswork.

Without estimation skills, you might:

1. **Over-engineer**: Design a distributed system for 100 users
2. **Under-engineer**: Propose a single server for 100 million users
3. **Choose wrong technologies**: Use Redis for 100TB of data (it's in-memory)
4. **Miss bottlenecks**: Not realize your database can't handle the QPS
5. **Lose credibility**: Interviewers notice when numbers don't make sense

### What Breaks Without Estimation

**Scenario 1: The Impossible Cache**

Candidate: "We'll cache all user sessions in Redis."

Interviewer: "How much memory do we need?"

Candidate: "Uh... a lot?"

Reality: 100 million users × 1KB per session = 100GB. A single Redis instance maxes out around 100GB. This is feasible but tight. If the candidate had said "1 billion users," they'd need a Redis cluster.

**Scenario 2: The Overloaded Database**

Candidate: "We'll store all data in PostgreSQL."

Interviewer: "Can it handle the load?"

Candidate: "PostgreSQL is very fast."

Reality: At 100,000 QPS, a single PostgreSQL instance will struggle. You need read replicas, connection pooling, or sharding. The candidate didn't do the math.

**Scenario 3: The Bandwidth Nightmare**

Candidate: "Users upload 4K videos, we store them in S3."

Interviewer: "What's the bandwidth requirement?"

Candidate: "S3 handles it."

Reality: 10,000 concurrent uploads × 10MB/s per upload = 100GB/s = 800 Gbps. That's a massive bandwidth requirement that affects architecture.

### Real Examples of the Problem

**Example 1: Twitter Timeline**

A candidate designed Twitter's timeline without estimating read QPS. They proposed computing the timeline on each read by querying all followed users' tweets. With 200 million DAU reading 100 tweets each, that's 20 billion timeline computations per day, each requiring potentially hundreds of database queries. The design would never work. Quick math would have revealed the need for pre-computation.

**Example 2: Video Streaming Storage**

A candidate designing YouTube storage said "we'll use a distributed file system." When asked about capacity, they hadn't considered that 500 hours of video are uploaded every minute. At 1GB per minute of video, that's 30TB per hour, 720TB per day, 260PB per year. This scale requires specialized storage architecture, not just "a distributed file system."

---

## 2️⃣ Intuition and Mental Model

### The Order of Magnitude Mindset

Back-of-envelope calculations aren't about precision. They're about getting the right order of magnitude.

```
PRECISION VS MAGNITUDE

Precise: 1,847,293 requests per second
Magnitude: ~2 million requests per second

For system design, magnitude is enough.
The difference between 1.8M and 2M doesn't change the architecture.
The difference between 2M and 20M changes everything.
```

Think of it like estimating travel time:
- "About 2 hours" is useful for planning
- "2 hours, 17 minutes, and 43 seconds" is false precision
- "Somewhere between 30 minutes and 8 hours" is useless

### The Powers of 10 Mental Model

Everything in computing scales by powers of 10. Memorize these:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    POWERS OF 10 REFERENCE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  USERS                          TIME                                 │
│  ──────                         ────                                 │
│  1K    = Thousand              1 second                              │
│  1M    = Million               1 minute  ≈ 60 seconds                │
│  1B    = Billion               1 hour    ≈ 3,600 seconds             │
│  1T    = Trillion              1 day     ≈ 86,400 ≈ 100K seconds     │
│                                1 month   ≈ 2.5M seconds              │
│  STORAGE                       1 year    ≈ 30M seconds               │
│  ───────                                                             │
│  1 KB  = 1,000 bytes                                                │
│  1 MB  = 1,000 KB = 1 million bytes                                 │
│  1 GB  = 1,000 MB = 1 billion bytes                                 │
│  1 TB  = 1,000 GB = 1 trillion bytes                                │
│  1 PB  = 1,000 TB                                                   │
│                                                                      │
│  NETWORK                                                            │
│  ───────                                                            │
│  1 Mbps  = 1 million bits per second = 125 KB/s                     │
│  1 Gbps  = 1 billion bits per second = 125 MB/s                     │
│  10 Gbps = typical server NIC                                       │
│  100 Gbps = high-end data center                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Simplification Mindset

Use round numbers and simplify aggressively:

```
SIMPLIFICATION EXAMPLES

Instead of:     Use:
86,400 seconds  100,000 seconds (per day)
2,592,000       2.5 million (per month)
31,536,000      30 million (per year)

Instead of:     Use:
1,024 bytes     1,000 bytes (1 KB)
1,048,576       1,000,000 (1 MB)

The 2.4% error from 1024→1000 doesn't matter.
```

---

## 3️⃣ How It Works Internally

### The Four Core Calculations

Every system design estimation boils down to four calculations:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FOUR CORE CALCULATIONS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. TRAFFIC (QPS)                                                   │
│     How many requests per second?                                   │
│     Formula: (Daily requests) / (Seconds per day)                   │
│                                                                      │
│  2. STORAGE                                                         │
│     How much disk space?                                            │
│     Formula: (Items) × (Size per item) × (Time period)              │
│                                                                      │
│  3. BANDWIDTH                                                       │
│     How much data transfer?                                         │
│     Formula: (QPS) × (Size per request)                             │
│                                                                      │
│  4. MEMORY (Cache)                                                  │
│     How much RAM for caching?                                       │
│     Formula: (Hot items) × (Size per item)                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

Let's explore each in detail.

### Calculation 1: Traffic (QPS)

**Formula**:
```
QPS = (Daily active users × Actions per user per day) / Seconds per day
Peak QPS = QPS × Peak multiplier (typically 2-3x)
```

**Example: Twitter**

```
Given:
- 200 million DAU
- Average user reads 100 tweets per day
- Average user posts 0.5 tweets per day

Read QPS:
= (200M users × 100 reads) / 100K seconds
= 20 billion / 100K
= 200,000 QPS

Write QPS:
= (200M users × 0.5 posts) / 100K seconds
= 100 million / 100K
= 1,000 QPS

Peak (3x):
- Read: 600,000 QPS
- Write: 3,000 QPS

Read/Write ratio: 200:1 (very read-heavy)
```

**Mental shortcut**: 
```
1 million daily actions ≈ 10 QPS
(because 1M / 100K = 10)

So 200M reads/day = 2,000 QPS
Wait, that's wrong. Let me recalculate.

200M users × 100 reads = 20B reads/day
20B / 100K seconds = 200,000 QPS ✓
```

### Calculation 2: Storage

**Formula**:
```
Daily storage = (New items per day) × (Size per item)
Yearly storage = Daily storage × 365
N-year storage = Yearly storage × N
```

**Example: Instagram Photos**

```
Given:
- 500 million photos uploaded per day
- Average photo size: 500 KB (compressed)
- Store for 10 years

Daily storage:
= 500M photos × 500 KB
= 250 TB per day

Yearly storage:
= 250 TB × 365
= 91 PB per year
≈ 100 PB per year (rounded)

10-year storage:
= 100 PB × 10
= 1 EB (Exabyte)
```

**Mental shortcut**:
```
1 million items × 1 KB = 1 GB
1 million items × 1 MB = 1 TB
1 billion items × 1 KB = 1 TB
```

### Calculation 3: Bandwidth

**Formula**:
```
Bandwidth = QPS × Size per request
```

**Example: Video Streaming**

```
Given:
- 1 million concurrent viewers
- 5 Mbps per stream (HD quality)

Bandwidth:
= 1M viewers × 5 Mbps
= 5 Tbps (Terabits per second)
= 625 GB/s

This is massive. Netflix uses CDN edge servers 
distributed globally to handle this.
```

**Example: API Requests**

```
Given:
- 100,000 QPS
- Average response size: 10 KB

Bandwidth:
= 100K QPS × 10 KB
= 1 GB/s
= 8 Gbps

A single 10 Gbps NIC can handle this, but barely.
Need multiple servers for redundancy.
```

### Calculation 4: Memory (Cache Sizing)

**Formula**:
```
Cache size = (Number of hot items) × (Size per item)
```

**The 80/20 Rule**: 20% of items get 80% of traffic. Cache the hot 20%.

**Example: URL Shortener Cache**

```
Given:
- 1 billion total URLs
- Each URL mapping: 500 bytes
- Cache hot 20%

Cache size:
= 1B URLs × 20% × 500 bytes
= 200M × 500 bytes
= 100 GB

This fits in a large Redis instance or small cluster.
```

**Example: Session Cache**

```
Given:
- 100 million DAU
- 10% concurrent at peak
- Session size: 1 KB

Cache size:
= 100M × 10% × 1 KB
= 10M × 1 KB
= 10 GB

Easily fits in a single Redis instance.
```

---

## 4️⃣ Simulation: Complete Estimation for "Design YouTube"

Let's walk through a complete estimation exercise.

### Step 1: Clarify Assumptions

```
ASSUMPTIONS:
- 2 billion monthly active users
- 1 billion daily active users
- Average user watches 5 videos per day
- Average video length: 5 minutes
- 500 hours of video uploaded per minute
- Store videos for 10 years
- Multiple quality levels (360p, 720p, 1080p, 4K)
```

### Step 2: Traffic Calculation

```
VIDEO VIEWS:
Daily views = 1B users × 5 videos = 5 billion views/day
View QPS = 5B / 100K seconds = 50,000 QPS
Peak QPS = 50,000 × 3 = 150,000 QPS

VIDEO UPLOADS:
500 hours/minute = 500 × 60 = 30,000 hours/day
Assuming average video is 5 minutes:
Daily uploads = 30,000 hours × 12 videos/hour = 360,000 videos/day
Upload QPS = 360,000 / 100K = 3.6 QPS
Peak upload QPS ≈ 10 QPS

SEARCH:
Assume 20% of users search once per day
Search QPS = (1B × 20%) / 100K = 2,000 QPS
```

### Step 3: Storage Calculation

```
VIDEO STORAGE:
- 500 hours uploaded per minute
- 1 minute of video ≈ 50 MB (compressed, single quality)
- Store 4 quality levels (average 2x storage)

Per minute: 500 hours × 60 min × 50 MB × 2 = 3 TB/minute
Per day: 3 TB × 60 × 24 = 4.3 PB/day
Per year: 4.3 PB × 365 = 1.6 EB/year
10 years: 16 EB (Exabytes)

METADATA STORAGE:
- 360,000 videos/day
- Metadata per video: 10 KB (title, description, tags, etc.)

Per day: 360K × 10 KB = 3.6 GB/day
Per year: 3.6 GB × 365 = 1.3 TB/year
10 years: 13 TB (negligible compared to video)

THUMBNAIL STORAGE:
- 5 thumbnails per video × 50 KB each
- 360,000 videos/day

Per day: 360K × 5 × 50 KB = 90 GB/day
Per year: 90 GB × 365 = 33 TB/year
10 years: 330 TB
```

### Step 4: Bandwidth Calculation

```
VIDEO STREAMING (OUTBOUND):
- 50,000 view QPS
- Average bitrate: 5 Mbps (mix of qualities)
- Average view duration: 3 minutes (not all watch full video)

Concurrent streams at any moment:
= 50,000 views/second × 180 seconds average
= 9 million concurrent streams

Bandwidth:
= 9M streams × 5 Mbps
= 45 Tbps (Terabits per second)
= 5.6 TB/s

This is why YouTube needs CDN with thousands of edge servers globally.

VIDEO UPLOAD (INBOUND):
- 3.6 upload QPS
- Average upload: 50 MB × 5 minutes = 250 MB

Bandwidth:
= 3.6 × 250 MB
= 900 MB/s
= 7.2 Gbps

Much more manageable than streaming.
```

### Step 5: Cache Sizing

```
VIDEO METADATA CACHE:
- 1 billion total videos
- Hot 1% (viral + recent) = 10 million videos
- Metadata: 10 KB per video

Cache size = 10M × 10 KB = 100 GB

THUMBNAIL CACHE:
- Hot 10 million videos
- 5 thumbnails × 50 KB = 250 KB per video

Cache size = 10M × 250 KB = 2.5 TB

USER SESSION CACHE:
- 100 million concurrent users
- Session: 1 KB

Cache size = 100M × 1 KB = 100 GB
```

### Step 6: Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    YOUTUBE ESTIMATION SUMMARY                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  TRAFFIC                                                            │
│  - View QPS: 50,000 (peak 150,000)                                  │
│  - Upload QPS: 4 (peak 10)                                          │
│  - Search QPS: 2,000                                                │
│                                                                      │
│  STORAGE (10 years)                                                 │
│  - Video: 16 EB (Exabytes)                                          │
│  - Metadata: 13 TB                                                  │
│  - Thumbnails: 330 TB                                               │
│                                                                      │
│  BANDWIDTH                                                          │
│  - Streaming: 45 Tbps (need global CDN)                             │
│  - Upload: 7.2 Gbps (manageable)                                    │
│                                                                      │
│  CACHE                                                              │
│  - Metadata: 100 GB                                                 │
│  - Thumbnails: 2.5 TB                                               │
│  - Sessions: 100 GB                                                 │
│                                                                      │
│  KEY INSIGHTS                                                       │
│  - Storage is the main challenge (Exabytes)                         │
│  - Streaming bandwidth requires CDN                                 │
│  - Read-heavy system (views >> uploads)                             │
│  - Caching is essential for metadata/thumbnails                     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Interview Experiences

**Google L5 (2023)**:
"I was designing a rate limiter. The interviewer asked 'How much memory do you need?' I calculated: 10 million unique IPs × 100 bytes per entry = 1 GB. 'That fits in memory on a single machine,' I said. The interviewer nodded and we moved on. If I had said 'a lot' or guessed wrong by 100x, it would have been a red flag."

**Amazon L6 (2022)**:
"Design a logging system for AWS. I estimated: 1 million services × 1000 logs/second × 1 KB = 1 TB/second of log data. The interviewer said 'Good, you understand the scale. Now how do you handle that?' The estimation set up the entire design discussion."

**Meta E5 (2023)**:
"Design Facebook's news feed. I calculated read QPS: 2 billion DAU × 10 feed refreshes × 20% at peak hour = 4 billion reads in peak hour = 1.1 million QPS. The interviewer asked 'Can a single database handle that?' Obviously not. This led to discussing caching, pre-computation, and sharding."

### Industry Benchmarks to Memorize

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INDUSTRY BENCHMARKS                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  DATABASE PERFORMANCE (single instance)                             │
│  ─────────────────────────────────────                              │
│  PostgreSQL: 10,000-50,000 QPS (depends on query complexity)        │
│  MySQL: 10,000-50,000 QPS                                           │
│  MongoDB: 10,000-100,000 QPS                                        │
│  Cassandra: 10,000-100,000 QPS per node                             │
│  Redis: 100,000-500,000 QPS                                         │
│                                                                      │
│  WEB SERVER PERFORMANCE                                             │
│  ──────────────────────                                             │
│  Single server (simple API): 1,000-10,000 QPS                       │
│  With connection pooling: 10,000-50,000 QPS                         │
│  Nginx (static files): 100,000+ QPS                                 │
│                                                                      │
│  NETWORK                                                            │
│  ───────                                                            │
│  Typical server NIC: 10 Gbps = 1.25 GB/s                            │
│  High-end NIC: 100 Gbps = 12.5 GB/s                                 │
│  AWS same-AZ latency: <1 ms                                         │
│  AWS cross-region latency: 50-200 ms                                │
│                                                                      │
│  STORAGE                                                            │
│  ───────                                                            │
│  SSD IOPS: 10,000-100,000                                           │
│  SSD throughput: 500 MB/s - 3 GB/s                                  │
│  HDD IOPS: 100-200                                                  │
│  HDD throughput: 100-200 MB/s                                       │
│                                                                      │
│  MEMORY                                                             │
│  ──────                                                             │
│  Typical server: 64-256 GB RAM                                      │
│  High-memory instance: 1-4 TB RAM                                   │
│  Redis max practical: 100 GB per instance                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Common Estimation Patterns

**Pattern 1: The 100K Shortcut**

```
1 day ≈ 100,000 seconds

So: Daily volume / 100,000 = QPS

Examples:
- 10 million daily requests = 100 QPS
- 1 billion daily requests = 10,000 QPS
- 100 billion daily requests = 1 million QPS
```

**Pattern 2: The 80/20 Cache Rule**

```
Cache the hot 20% of data.
Or more aggressively, cache the hot 1% for very large datasets.

Examples:
- 1 billion URLs, cache 1% = 10 million URLs
- 100 million users, cache active 10% = 10 million users
```

**Pattern 3: The 3x Peak Multiplier**

```
Peak traffic ≈ 3x average traffic

For critical systems, design for 5-10x headroom.
```

**Pattern 4: The Storage Growth Estimate**

```
5-year storage = Daily growth × 365 × 5 × 2 (for redundancy)
```

---

## 6️⃣ Practice Problems with Solutions

### Problem 1: Design a URL Shortener

**Given**: 100 million new URLs per day, URLs never expire

**Calculate**:

```
TRAFFIC:
Write QPS = 100M / 100K = 1,000 QPS
Assume 10:1 read/write ratio
Read QPS = 10,000 QPS
Peak = 30,000 QPS

STORAGE (5 years):
URL mapping size = 500 bytes (short URL + long URL + metadata)
Daily = 100M × 500 bytes = 50 GB/day
5 years = 50 GB × 365 × 5 = 91 TB ≈ 100 TB

CACHE:
Hot 20% of URLs = 100M × 365 × 5 × 20% = 36 billion × 20% = 7 billion URLs
Too many! Cache hot 1% = 360 million URLs
Cache size = 360M × 500 bytes = 180 GB

BANDWIDTH:
Read = 10,000 QPS × 500 bytes = 5 MB/s = 40 Mbps (trivial)
```

### Problem 2: Design Twitter

**Given**: 500 million tweets per day, 200 million DAU

**Calculate**:

```
TRAFFIC:
Write QPS = 500M / 100K = 5,000 QPS
Read (100 tweets/user/day) = 200M × 100 / 100K = 200,000 QPS
Peak read = 600,000 QPS

STORAGE (5 years):
Tweet size = 500 bytes (280 chars + metadata)
Daily = 500M × 500 bytes = 250 GB/day
5 years = 250 GB × 365 × 5 = 456 TB ≈ 500 TB

TIMELINE CACHE:
200M users × 800 tweets cached × 8 bytes (tweet ID) = 1.28 TB
Or with full tweet: 200M × 800 × 500 bytes = 80 TB (too big, use IDs only)

BANDWIDTH:
Read = 200,000 QPS × 500 bytes = 100 MB/s = 800 Mbps
```

### Problem 3: Design a Chat Application

**Given**: 100 million DAU, 50 messages per user per day

**Calculate**:

```
TRAFFIC:
Messages/day = 100M × 50 = 5 billion
Write QPS = 5B / 100K = 50,000 QPS
Assume 2:1 read/write (read own + recipient)
Read QPS = 100,000 QPS
Peak = 300,000 QPS

STORAGE (5 years):
Message size = 200 bytes (text + metadata)
Daily = 5B × 200 bytes = 1 TB/day
5 years = 1 TB × 365 × 5 = 1.8 PB

CACHE (recent messages):
Hot conversations = 10M concurrent chats
Messages per chat cached = 100
Cache size = 10M × 100 × 200 bytes = 200 GB

BANDWIDTH:
Write = 50,000 QPS × 200 bytes = 10 MB/s = 80 Mbps
WebSocket overhead adds ~2x = 160 Mbps
```

### Problem 4: Design a Video Streaming Service

**Given**: 10 million concurrent viewers, 5 Mbps average stream

**Calculate**:

```
BANDWIDTH:
Streaming = 10M × 5 Mbps = 50 Tbps
Per edge server (10 Gbps) = 50 Tbps / 10 Gbps = 5,000 edge servers minimum
With 50% utilization target = 10,000 edge servers

STORAGE (1 million videos):
Average video = 1 GB (multiple qualities)
Total = 1M × 1 GB = 1 PB
With 3x replication = 3 PB

CDN CACHE:
Hot 10% of videos = 100K videos
Cache size = 100K × 1 GB = 100 TB per region
10 regions = 1 PB total CDN cache
```

### Problem 5: Design an E-commerce Platform

**Given**: 10 million daily orders, 100 million products

**Calculate**:

```
TRAFFIC:
Order QPS = 10M / 100K = 100 QPS
Product view (100 views per order) = 1B / 100K = 10,000 QPS
Search = 5,000 QPS (estimate)
Peak (Black Friday 10x) = 100,000 QPS

STORAGE:
Product data = 100M × 10 KB = 1 TB
Order data (5 years) = 10M × 365 × 5 × 1 KB = 18 TB
Product images = 100M × 10 images × 100 KB = 100 TB

CACHE:
Hot products (1%) = 1M × 10 KB = 10 GB
User sessions = 10M concurrent × 1 KB = 10 GB
Shopping carts = 1M active × 10 KB = 10 GB
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Pitfall 1: False Precision

**Wrong**: "We need exactly 847,293 QPS."

**Right**: "We need roughly 1 million QPS."

False precision wastes time and suggests you don't understand estimation.

### Pitfall 2: Forgetting Peak Traffic

**Wrong**: "Average QPS is 10,000, so we need 10,000 QPS capacity."

**Right**: "Average is 10,000, peak is 3x = 30,000. Design for 50,000 with headroom."

Systems fail at peak, not average.

### Pitfall 3: Ignoring Data Growth

**Wrong**: "We need 100 GB of storage."

**Right**: "We need 100 GB today, growing 10 GB/month. In 2 years, 340 GB."

Always consider growth trajectory.

### Pitfall 4: Confusing Bits and Bytes

**Wrong**: "10 Gbps network = 10 GB/s throughput."

**Right**: "10 Gbps = 1.25 GB/s (8 bits per byte)."

Network speeds are in bits. Storage is in bytes.

### Pitfall 5: Not Showing Work

**Wrong**: "We need about 100 servers."

**Right**: "50,000 QPS ÷ 1,000 QPS per server = 50 servers. With 2x redundancy = 100 servers."

Show your math. It demonstrates your thinking process.

### Pitfall 6: Forgetting Replication

**Wrong**: "100 TB of data needs 100 TB storage."

**Right**: "100 TB with 3x replication = 300 TB raw storage."

Production systems replicate data for durability.

---

## 8️⃣ When NOT to Do Detailed Estimation

### Scenario 1: Obvious Scale

If the scale is obviously small (internal tool for 100 users) or obviously large (global social network), don't spend 5 minutes on math.

**Action**: "This is clearly a small/large scale system. Let me make quick assumptions and move to design."

### Scenario 2: Interviewer Provides Numbers

If the interviewer gives you the numbers, don't recalculate them.

**Interviewer**: "Assume 10,000 QPS and 1 TB storage."

**Action**: "Got it. Let me design for those requirements."

### Scenario 3: Time Pressure

If you're running low on time, do minimal estimation.

**Action**: Quick mental math, state assumptions, move on. "Assuming moderate scale, let's say 10,000 QPS. I can refine this later if needed."

---

## 9️⃣ Interview Follow-up Questions WITH Answers

### Q1: "How did you arrive at that number?"

**Answer**: "Let me walk through my calculation. We have X users doing Y actions per day. That's X × Y total actions. Divided by 100,000 seconds per day gives us Z QPS. I multiplied by 3 for peak traffic. Here's the math on the whiteboard..."

Always be ready to show your work.

### Q2: "What if your estimate is wrong by 10x?"

**Answer**: "If it's 10x higher, we'd need to add horizontal scaling, sharding, and caching layers. The architecture would change from a simple replicated setup to a distributed system. If it's 10x lower, we might be over-engineering, and a simpler single-server approach could work. This is why I prefer to design with 2-3x headroom, not 10x."

### Q3: "How would you validate these estimates?"

**Answer**: "In a real project, I'd validate through load testing with realistic traffic patterns, analyze logs from similar systems if available, start with a smaller pilot deployment and measure actual usage, and build monitoring to track actual vs estimated metrics. Estimates are starting points, not final answers."

### Q4: "Your storage estimate seems high/low. Why?"

**Answer**: "Let me re-examine my assumptions. I assumed [state assumptions]. If [alternative assumption], the number would be [recalculate]. Which assumption better matches your expectations? I'm happy to adjust based on more accurate inputs."

### Q5: "What's the most important number in your estimation?"

**Answer**: "For this system, I'd say [QPS/storage/bandwidth] is most critical because it's the primary bottleneck. If we get this wrong by an order of magnitude, the entire architecture needs to change. The other numbers are important but have more flexibility in how we address them."

---

## 🔟 Practice Problems with Solutions

### Problem 1: Design a URL Shortener

**Given**:
- 100 million URLs shortened per day
- 1 billion total URLs in system
- Each URL mapping: 500 bytes
- Read/write ratio: 100:1 (reads >> writes)

**Calculate**:
1. Write QPS
2. Read QPS (average and peak)
3. Storage for 5 years
4. Cache size (cache hot 20%)

**Solution**:

**1. Write QPS**:
```
Daily writes = 100M URLs/day
Write QPS = 100M / 100K = 1,000 QPS
Peak write QPS = 1,000 × 3 = 3,000 QPS
```

**2. Read QPS**:
```
Read/write ratio = 100:1
Read QPS = 1,000 × 100 = 100,000 QPS
Peak read QPS = 100,000 × 3 = 300,000 QPS
```

**3. Storage (5 years)**:
```
Daily storage = 100M URLs × 500 bytes = 50 GB/day
Yearly storage = 50 GB × 365 = 18.25 TB/year ≈ 20 TB/year
5-year storage = 20 TB × 5 = 100 TB

With replication (3x): 100 TB × 3 = 300 TB
```

**4. Cache size**:
```
Hot URLs = 1B × 20% = 200M URLs
Cache size = 200M × 500 bytes = 100 GB

This fits in a large Redis instance or small Redis cluster.
```

---

### Problem 2: Design a Chat System

**Given**:
- 50 million daily active users
- Average user sends 20 messages per day
- Average message size: 200 bytes
- Average user is in 3 group chats
- Messages stored for 1 year

**Calculate**:
1. Write QPS (message sends)
2. Storage for 1 year
3. Bandwidth for real-time delivery
4. Cache size for active conversations

**Solution**:

**1. Write QPS**:
```
Daily messages = 50M users × 20 messages = 1 billion messages/day
Write QPS = 1B / 100K = 10,000 QPS
Peak write QPS = 10,000 × 3 = 30,000 QPS
```

**2. Storage (1 year)**:
```
Daily storage = 1B messages × 200 bytes = 200 GB/day
Yearly storage = 200 GB × 365 = 73 TB/year ≈ 75 TB/year

With replication (3x): 75 TB × 3 = 225 TB
```

**3. Bandwidth (real-time delivery)**:
```
Peak write QPS = 30,000 QPS
Average message = 200 bytes
Bandwidth = 30,000 × 200 bytes = 6 MB/s = 48 Mbps

This is manageable for a single data center.
But with global distribution, need multiple regions.
```

**4. Cache size (active conversations)**:
```
Active users = 50M DAU
Average 3 conversations per user
Active conversations = 50M × 3 = 150M conversations

Cache recent messages per conversation:
- Last 100 messages per conversation
- 100 messages × 200 bytes = 20 KB per conversation

Cache size = 150M × 20 KB = 3 TB

This is too large for single cache. Need:
- Cache only top 10% active conversations = 300 GB
- Or use distributed cache across multiple instances
```

---

### Problem 3: Design a Social Media Feed

**Given**:
- 200 million daily active users
- Average user follows 200 people
- Average user posts 1 post per day
- Average post size: 1 KB (text + metadata)
- Average user views feed 5 times per day
- Feed shows 20 posts per view

**Calculate**:
1. Write QPS (posts)
2. Read QPS (feed views)
3. Storage for posts (5 years)
4. Bandwidth for feed delivery

**Solution**:

**1. Write QPS**:
```
Daily posts = 200M users × 1 post = 200M posts/day
Write QPS = 200M / 100K = 2,000 QPS
Peak write QPS = 2,000 × 3 = 6,000 QPS
```

**2. Read QPS**:
```
Daily feed views = 200M users × 5 views = 1 billion views/day
Posts per view = 20
Total post reads = 1B × 20 = 20 billion reads/day
Read QPS = 20B / 100K = 200,000 QPS
Peak read QPS = 200,000 × 3 = 600,000 QPS
```

**3. Storage (5 years)**:
```
Daily storage = 200M posts × 1 KB = 200 GB/day
Yearly storage = 200 GB × 365 = 73 TB/year ≈ 75 TB/year
5-year storage = 75 TB × 5 = 375 TB

With replication (3x): 375 TB × 3 = 1.1 PB
```

**4. Bandwidth**:
```
Peak read QPS = 600,000 QPS
Posts per request = 20
Size per post = 1 KB
Response size = 20 × 1 KB = 20 KB per request

Bandwidth = 600K × 20 KB = 12 GB/s = 96 Gbps

This requires multiple servers and CDN for global distribution.
```

---

### Problem 4: Design a File Storage System

**Given**:
- 10 million users
- Average user stores 1,000 files
- Average file size: 5 MB
- 10% of users are active daily
- Active users upload 10 files per day

**Calculate**:
1. Total storage needed
2. Daily upload QPS
3. Daily download QPS (assume users download 5 files/day)
4. Bandwidth for uploads and downloads

**Solution**:

**1. Total storage**:
```
Total files = 10M users × 1,000 files = 10 billion files
Total storage = 10B × 5 MB = 50 PB

With replication (3x): 50 PB × 3 = 150 PB
```

**2. Upload QPS**:
```
Active users = 10M × 10% = 1M users
Daily uploads = 1M × 10 files = 10M files/day
Upload QPS = 10M / 100K = 100 QPS
Peak upload QPS = 100 × 3 = 300 QPS
```

**3. Download QPS**:
```
Daily downloads = 1M × 5 files = 5M files/day
Download QPS = 5M / 100K = 50 QPS
Peak download QPS = 50 × 3 = 150 QPS
```

**4. Bandwidth**:
```
Upload bandwidth:
Peak upload QPS = 300 QPS
File size = 5 MB
Upload bandwidth = 300 × 5 MB = 1.5 GB/s = 12 Gbps

Download bandwidth:
Peak download QPS = 150 QPS
File size = 5 MB
Download bandwidth = 150 × 5 MB = 750 MB/s = 6 Gbps

Total bandwidth = 12 + 6 = 18 Gbps
```

---

### Problem 5: Design a Search Engine

**Given**:
- 1 billion web pages indexed
- Average page size: 50 KB (text content)
- 100 million searches per day
- Average search returns 10 results
- Index stored for 1 year

**Calculate**:
1. Total index storage
2. Search QPS
3. Bandwidth for search results
4. Cache size for popular searches

**Solution**:

**1. Index storage**:
```
Total pages = 1B pages
Size per page = 50 KB
Total index = 1B × 50 KB = 50 TB

With replication (3x): 50 TB × 3 = 150 TB
```

**2. Search QPS**:
```
Daily searches = 100M searches/day
Search QPS = 100M / 100K = 1,000 QPS
Peak search QPS = 1,000 × 3 = 3,000 QPS
```

**3. Bandwidth**:
```
Peak search QPS = 3,000 QPS
Results per search = 10
Metadata per result = 1 KB
Response size = 10 × 1 KB = 10 KB

Bandwidth = 3,000 × 10 KB = 30 MB/s = 240 Mbps

Very manageable bandwidth.
```

**4. Cache size**:
```
Popular searches = 20% of searches (80/20 rule)
Daily popular searches = 100M × 20% = 20M searches
Cache these for 1 day

Cache size = 20M × 10 KB = 200 GB

Fits in Redis cluster.
```

---

### Problem 6: Design a Notification System

**Given**:
- 500 million users
- Average user receives 10 notifications per day
- Notification size: 500 bytes
- 50% of users are active daily
- Notifications delivered in real-time (<100ms latency)

**Calculate**:
1. Notification QPS
2. Storage for notifications (30 days retention)
3. Bandwidth for delivery
4. Cache size for unread notifications

**Solution**:

**1. Notification QPS**:
```
Active users = 500M × 50% = 250M users
Daily notifications = 250M × 10 = 2.5 billion/day
Notification QPS = 2.5B / 100K = 25,000 QPS
Peak notification QPS = 25,000 × 3 = 75,000 QPS
```

**2. Storage (30 days)**:
```
Daily storage = 2.5B × 500 bytes = 1.25 TB/day
30-day storage = 1.25 TB × 30 = 37.5 TB ≈ 40 TB

With replication (3x): 40 TB × 3 = 120 TB
```

**3. Bandwidth**:
```
Peak notification QPS = 75,000 QPS
Notification size = 500 bytes
Bandwidth = 75K × 500 bytes = 37.5 MB/s = 300 Mbps

Manageable, but need global distribution for low latency.
```

**4. Cache size (unread notifications)**:
```
Active users = 250M
Average unread notifications = 5 per user
Cache size = 250M × 5 × 500 bytes = 625 GB

Need Redis cluster for this.
```

---

### Problem 7: Design an E-commerce Product Catalog

**Given**:
- 10 million products
- Average product data: 10 KB (name, description, images metadata)
- 50 million daily active users
- Average user views 20 products per day
- Products updated 1% per day

**Calculate**:
1. Total product storage
2. Read QPS (product views)
3. Write QPS (product updates)
4. Cache size for popular products

**Solution**:

**1. Total storage**:
```
Total products = 10M
Size per product = 10 KB
Total storage = 10M × 10 KB = 100 GB

With replication (3x): 100 GB × 3 = 300 GB
```

**2. Read QPS**:
```
Daily product views = 50M users × 20 views = 1 billion views/day
Read QPS = 1B / 100K = 10,000 QPS
Peak read QPS = 10,000 × 3 = 30,000 QPS
```

**3. Write QPS**:
```
Products updated = 10M × 1% = 100K products/day
Write QPS = 100K / 100K = 1 QPS
Peak write QPS = 1 × 3 = 3 QPS

Very write-light system.
```

**4. Cache size**:
```
Popular products = 20% (80/20 rule)
Popular products = 10M × 20% = 2M products
Cache size = 2M × 10 KB = 20 GB

Easily fits in single Redis instance.
```

---

### Problem 8: Design a Real-time Analytics Dashboard

**Given**:
- 1 million events per second (peak)
- Average event size: 1 KB
- Events stored for 7 days
- Dashboard serves 10,000 concurrent users
- Each user refreshes dashboard every 10 seconds

**Calculate**:
1. Storage for 7 days
2. Dashboard read QPS
3. Bandwidth for event ingestion
4. Bandwidth for dashboard delivery

**Solution**:

**1. Storage (7 days)**:
```
Peak events = 1M events/sec
Average events = 1M / 3 = 333K events/sec (assuming 3x peak multiplier)
Daily events = 333K × 100K seconds = 33.3 billion events/day

Event size = 1 KB
Daily storage = 33.3B × 1 KB = 33.3 TB/day
7-day storage = 33.3 TB × 7 = 233 TB ≈ 250 TB

With replication (3x): 250 TB × 3 = 750 TB
```

**2. Dashboard read QPS**:
```
Concurrent users = 10,000
Refresh rate = every 10 seconds
Read QPS = 10,000 / 10 = 1,000 QPS
```

**3. Event ingestion bandwidth**:
```
Peak events = 1M events/sec
Event size = 1 KB
Ingestion bandwidth = 1M × 1 KB = 1 GB/s = 8 Gbps

Requires multiple ingestion servers.
```

**4. Dashboard delivery bandwidth**:
```
Read QPS = 1,000 QPS
Average dashboard response = 100 KB (aggregated data)
Bandwidth = 1,000 × 100 KB = 100 MB/s = 800 Mbps

Manageable bandwidth.
```

---

### Problem 9: Design a Gaming Leaderboard

**Given**:
- 10 million players
- Average player plays 10 games per day
- Each game generates 1 score update
- Leaderboard shows top 1,000 players
- Scores stored for 30 days

**Calculate**:
1. Score update QPS
2. Leaderboard read QPS (assume 1M reads/day)
3. Storage for scores (30 days)
4. Cache size for leaderboard

**Solution**:

**1. Score update QPS**:
```
Daily games = 10M players × 10 games = 100M games/day
Score updates = 100M (1 per game)
Update QPS = 100M / 100K = 1,000 QPS
Peak update QPS = 1,000 × 3 = 3,000 QPS
```

**2. Leaderboard read QPS**:
```
Daily reads = 1M reads/day
Read QPS = 1M / 100K = 10 QPS
Peak read QPS = 10 × 3 = 30 QPS
```

**3. Storage (30 days)**:
```
Daily scores = 100M scores/day
Score size = 100 bytes (player_id, score, timestamp)
Daily storage = 100M × 100 bytes = 10 GB/day
30-day storage = 10 GB × 30 = 300 GB

With replication (3x): 300 GB × 3 = 900 GB
```

**4. Cache size**:
```
Leaderboard = top 1,000 players
Score entry = 100 bytes
Cache size = 1,000 × 100 bytes = 100 KB

Tiny cache! Can store in application memory.
```

---

### Problem 10: Design a Video Streaming Service

**Given**:
- 50 million daily active users
- Average user watches 5 videos per day
- Average video length: 10 minutes
- Average bitrate: 5 Mbps (HD quality)
- Videos stored in 4 quality levels (360p, 720p, 1080p, 4K)
- Average storage per minute: 50 MB (single quality)

**Calculate**:
1. Video view QPS
2. Storage for videos (assume 1M total videos, average 10 min)
3. Streaming bandwidth (outbound)
4. Cache size for popular videos

**Solution**:

**1. Video view QPS**:
```
Daily views = 50M users × 5 videos = 250M views/day
View QPS = 250M / 100K = 2,500 QPS
Peak view QPS = 2,500 × 3 = 7,500 QPS
```

**2. Storage**:
```
Total videos = 1M videos
Average length = 10 minutes
Storage per video (single quality) = 10 min × 50 MB = 500 MB
Storage per video (4 qualities) = 500 MB × 2 (average) = 1 GB

Total storage = 1M × 1 GB = 1 PB

With replication (3x): 1 PB × 3 = 3 PB
```

**3. Streaming bandwidth**:
```
Peak view QPS = 7,500 QPS
Average view duration = 5 minutes (not all watch full video)
Concurrent streams = 7,500 × 300 seconds = 2.25M concurrent streams

Bitrate = 5 Mbps
Bandwidth = 2.25M × 5 Mbps = 11.25 Tbps

This requires massive CDN with thousands of edge servers globally.
```

**4. Cache size**:
```
Popular videos = 1% (viral + recent)
Popular videos = 1M × 1% = 10K videos
Cache metadata per video = 10 KB
Cache size = 10K × 10 KB = 100 MB

Very small cache for metadata. Video content served from CDN.
```

---

### Problem 11: Design a Distributed Logging System

**Given**:
- 1,000 application servers
- Each server generates 1,000 log entries per second
- Average log entry: 1 KB
- Logs stored for 90 days
- 100 analysts query logs (10 queries per analyst per day)

**Calculate**:
1. Total log ingestion QPS
2. Storage for 90 days
3. Query QPS
4. Bandwidth for log ingestion

**Solution**:

**1. Log ingestion QPS**:
```
Servers = 1,000
Logs per server = 1,000 logs/sec
Total ingestion = 1,000 × 1,000 = 1M logs/sec = 1M QPS
```

**2. Storage (90 days)**:
```
Daily logs = 1M logs/sec × 100K seconds = 100 billion logs/day
Log size = 1 KB
Daily storage = 100B × 1 KB = 100 TB/day
90-day storage = 100 TB × 90 = 9 PB

With replication (3x): 9 PB × 3 = 27 PB
```

**3. Query QPS**:
```
Analysts = 100
Queries per analyst = 10 queries/day
Daily queries = 100 × 10 = 1,000 queries/day
Query QPS = 1,000 / 100K = 0.01 QPS

Very low query rate (analytical workload).
```

**4. Ingestion bandwidth**:
```
Ingestion QPS = 1M logs/sec
Log size = 1 KB
Bandwidth = 1M × 1 KB = 1 GB/s = 8 Gbps

Requires distributed ingestion across multiple servers.
```

---

### Problem 12: Design a Recommendation System

**Given**:
- 100 million users
- 10 million items (products, videos, etc.)
- Average user-item interaction: 1 KB
- 50 million daily active users
- Average user has 100 interactions per day
- Recommendations computed daily for all users

**Calculate**:
1. Interaction write QPS
2. Storage for interactions (1 year)
3. Recommendation computation load
4. Cache size for recommendations

**Solution**:

**1. Interaction write QPS**:
```
Active users = 50M
Interactions per user = 100/day
Daily interactions = 50M × 100 = 5 billion/day
Write QPS = 5B / 100K = 50,000 QPS
Peak write QPS = 50K × 3 = 150,000 QPS
```

**2. Storage (1 year)**:
```
Daily interactions = 5B
Interaction size = 1 KB
Daily storage = 5B × 1 KB = 5 TB/day
Yearly storage = 5 TB × 365 = 1.8 PB/year ≈ 2 PB/year

With replication (3x): 2 PB × 3 = 6 PB
```

**3. Recommendation computation**:
```
Users = 100M
Recommendations computed daily
Recommendations per user = 100
Total recommendations = 100M × 100 = 10 billion/day

If computed over 1 hour:
Computation rate = 10B / 3,600 seconds = 2.8M recommendations/sec

Requires distributed computation (Spark, Hadoop, etc.)
```

**4. Cache size**:
```
Active users = 50M
Recommendations per user = 100
Recommendation size = 1 KB (item IDs + scores)
Cache size = 50M × 100 × 1 KB = 5 TB

Too large for single cache. Need distributed cache or 
cache only top 10 recommendations per user = 500 GB
```

---

### Problem 13: Design a Payment Processing System

**Given**:
- 10 million transactions per day
- Average transaction: 2 KB
- Transactions stored for 7 years (compliance)
- 1 million daily active users
- Average user makes 2 payments per day

**Calculate**:
1. Transaction QPS
2. Storage for 7 years
3. Bandwidth for transaction processing
4. Cache size for recent transactions

**Solution**:

**1. Transaction QPS**:
```
Daily transactions = 10M transactions/day
Transaction QPS = 10M / 100K = 100 QPS
Peak transaction QPS = 100 × 3 = 300 QPS
```

**2. Storage (7 years)**:
```
Daily transactions = 10M
Transaction size = 2 KB
Daily storage = 10M × 2 KB = 20 GB/day
Yearly storage = 20 GB × 365 = 7.3 TB/year
7-year storage = 7.3 TB × 7 = 51 TB

With replication (3x): 51 TB × 3 = 153 TB
```

**3. Bandwidth**:
```
Peak transaction QPS = 300 QPS
Transaction size = 2 KB
Bandwidth = 300 × 2 KB = 600 KB/s = 4.8 Mbps

Very manageable bandwidth.
```

**4. Cache size**:
```
Recent transactions = last 24 hours
Daily transactions = 10M
Transaction size = 2 KB
Cache size = 10M × 2 KB = 20 GB

Fits in Redis instance.
```

---

### Problem 14: Design a Content Delivery Network (CDN)

**Given**:
- 1 billion requests per day
- Average response size: 500 KB
- 80% cache hit rate
- 20% requests go to origin
- Global distribution across 50 edge locations

**Calculate**:
1. Total request QPS
2. Origin request QPS (cache misses)
3. Total bandwidth (edge + origin)
4. Storage per edge location (cache 1% of content)

**Solution**:

**1. Total request QPS**:
```
Daily requests = 1B requests/day
Request QPS = 1B / 100K = 10,000 QPS
Peak request QPS = 10K × 3 = 30,000 QPS
```

**2. Origin request QPS**:
```
Cache hit rate = 80%
Cache miss rate = 20%
Origin QPS = 10K × 20% = 2,000 QPS
Peak origin QPS = 2K × 3 = 6,000 QPS
```

**3. Total bandwidth**:
```
Edge bandwidth (serving cached content):
Peak edge QPS = 30K × 80% = 24,000 QPS
Response size = 500 KB
Edge bandwidth = 24K × 500 KB = 12 GB/s = 96 Gbps

Origin bandwidth (cache misses):
Peak origin QPS = 6,000 QPS
Response size = 500 KB
Origin bandwidth = 6K × 500 KB = 3 GB/s = 24 Gbps

Total bandwidth = 96 + 24 = 120 Gbps
```

**4. Storage per edge**:
```
Total content = Assume 1M unique items
Cache 1% = 1M × 1% = 10K items
Item size = 500 KB
Storage per edge = 10K × 500 KB = 5 GB

Very small per edge, but 50 edges × 5 GB = 250 GB total cache
```

---

### Problem 15: Design a Distributed File System

**Given**:
- 100 million files
- Average file size: 10 MB
- 10,000 concurrent users
- Average user uploads 5 files per day
- Average user downloads 10 files per day
- Files stored for 5 years

**Calculate**:
1. Total storage
2. Upload QPS
3. Download QPS
4. Bandwidth requirements

**Solution**:

**1. Total storage**:
```
Total files = 100M
Average file size = 10 MB
Total storage = 100M × 10 MB = 1 PB

With replication (3x): 1 PB × 3 = 3 PB
```

**2. Upload QPS**:
```
Concurrent users = 10K
Uploads per user = 5 files/day
Daily uploads = 10K × 5 = 50K files/day
Upload QPS = 50K / 100K = 0.5 QPS
Peak upload QPS = 0.5 × 3 = 1.5 QPS
```

**3. Download QPS**:
```
Downloads per user = 10 files/day
Daily downloads = 10K × 10 = 100K files/day
Download QPS = 100K / 100K = 1 QPS
Peak download QPS = 1 × 3 = 3 QPS
```

**4. Bandwidth**:
```
Upload bandwidth:
Peak upload QPS = 1.5 QPS
File size = 10 MB
Upload bandwidth = 1.5 × 10 MB = 15 MB/s = 120 Mbps

Download bandwidth:
Peak download QPS = 3 QPS
File size = 10 MB
Download bandwidth = 3 × 10 MB = 30 MB/s = 240 Mbps

Total bandwidth = 120 + 240 = 360 Mbps
```

---

## Practice Tips

1. **Start with assumptions**: Always state your assumptions clearly
2. **Show your work**: Write out calculations step by step
3. **Use round numbers**: 100K seconds per day, 3x peak multiplier
4. **Include replication**: Multiply storage by 3 for redundancy
5. **Consider peak traffic**: Always calculate peak (3x average)
6. **Check reasonableness**: Does your answer make sense?
7. **Identify bottlenecks**: Which number is the constraint?

---

## 🔟 One Clean Mental Summary

Back-of-envelope calculations are about getting the right order of magnitude, not precision. Master four calculations: QPS (traffic), storage, bandwidth, and cache sizing. Use round numbers and simplify aggressively. One day is 100,000 seconds. One million daily actions equals 10 QPS.

The goal is to anchor your design in reality. When you say "we need a Redis cluster," you should know it's because 500 GB of cache doesn't fit in a single instance. When you say "we need CDN," you should know it's because 50 Tbps can't come from a single data center.

Show your work. The calculation process demonstrates your thinking more than the final number. And always remember: peak traffic is 3x average, systems fail at peak, and storage needs replication.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────┐
│              BACK-OF-ENVELOPE CHEAT SHEET                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  TIME CONVERSIONS                                                   │
│  1 day = 86,400 sec ≈ 100,000 sec (use 100K)                       │
│  1 month ≈ 2.5 million seconds                                      │
│  1 year ≈ 30 million seconds                                        │
│                                                                      │
│  QPS FORMULA                                                        │
│  QPS = Daily actions / 100,000                                      │
│  Peak QPS = QPS × 3                                                 │
│                                                                      │
│  STORAGE FORMULA                                                    │
│  Daily = Items × Size per item                                      │
│  Yearly = Daily × 365                                               │
│  With replication = Raw × 3                                         │
│                                                                      │
│  QUICK MULTIPLIERS                                                  │
│  1M daily actions = 10 QPS                                          │
│  1M items × 1 KB = 1 GB                                             │
│  1M items × 1 MB = 1 TB                                             │
│                                                                      │
│  TYPICAL SIZES                                                      │
│  Tweet/message: 200-500 bytes                                       │
│  User profile: 1-10 KB                                              │
│  Image thumbnail: 10-50 KB                                          │
│  Full image: 100-500 KB                                             │
│  Video (1 min): 10-100 MB                                           │
│                                                                      │
│  SYSTEM LIMITS                                                      │
│  Single DB: 10-50K QPS                                              │
│  Redis: 100-500K QPS                                                │
│  Web server: 1-10K QPS                                              │
│  Network NIC: 10 Gbps = 1.25 GB/s                                   │
│                                                                      │
│  CACHE SIZING                                                       │
│  Cache hot 20% (or 1% for huge datasets)                            │
│  Redis instance: max ~100 GB                                        │
│                                                                      │
│  REMEMBER                                                           │
│  □ Show your work                                                   │
│  □ Use round numbers                                                │
│  □ Include peak traffic (3x)                                        │
│  □ Include replication (3x)                                         │
│  □ Bits vs Bytes (8:1)                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

