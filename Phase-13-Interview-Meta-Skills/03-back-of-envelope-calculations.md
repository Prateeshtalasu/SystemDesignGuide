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

