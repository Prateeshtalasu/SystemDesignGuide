# Back-of-Envelope Calculations

## 0️⃣ Prerequisites

Before diving into back-of-envelope calculations, you should understand:

- **Basic arithmetic**: Multiplication, division, and working with large numbers
- **Powers of 10**: Understanding scientific notation (10^3, 10^6, 10^9)
- **Core Metrics** (covered in Topic 03): Latency, throughput, and bandwidth concepts
- **Scalability** (covered in Topic 04): Understanding why systems need to handle varying loads

**Quick refresher**: In system design, we deal with massive numbers. A single server might handle thousands of requests per second. A database might store billions of records. Back-of-envelope calculations help us quickly estimate if our design can handle the expected load.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Specific Pain Point

Imagine you're in a system design interview. The interviewer asks: "Design a URL shortener like bit.ly." You start drawing boxes and arrows, but then they ask:

> "How many servers do you need?"
> "How much storage will you require?"
> "Can a single database handle this load?"

Without quick estimation skills, you're guessing blindly.

### What Systems Looked Like Before This

Before engineers developed estimation skills:

1. **Over-provisioning**: Companies bought 10x more servers than needed "just to be safe"
2. **Under-provisioning**: Systems crashed during peak traffic because no one calculated capacity
3. **Wasted interviews**: Candidates drew beautiful architectures that couldn't actually work at scale
4. **Failed launches**: Products launched without understanding their resource requirements

### What Breaks Without It

**Without estimation skills:**

- You propose a design that needs 1 TB of RAM per server (impossible)
- You suggest a single database for 1 billion users (will crash)
- You underestimate storage and run out of disk space in production
- You can't justify your design decisions with numbers
- Interviewers lose confidence in your engineering judgment

### Real Examples of the Problem

**Twitter's Early Days (2008-2010)**:
Twitter famously showed the "Fail Whale" error page constantly. Part of the problem was inadequate capacity planning. Engineers didn't properly estimate how many tweets per second the system needed to handle during peak events (like the World Cup or elections).

**Pokemon GO Launch (2016)**:
The game's servers crashed globally on launch day. Niantic estimated 5x their expected traffic, but actual traffic was 50x. Better estimation could have prepared them for viral growth.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Kitchen Analogy

Think of system design like planning a restaurant kitchen:

**Before opening, you need to estimate:**
- How many customers per hour? (requests per second)
- How many dishes can one chef prepare? (server throughput)
- How big should the refrigerator be? (storage)
- How wide should the kitchen door be? (bandwidth)

You don't need exact numbers. You need to know:
- "We need roughly 5 chefs, not 1 and not 100"
- "We need a walk-in freezer, not a mini-fridge"

**This is back-of-envelope calculation**: Quick, rough estimates that are within the right order of magnitude.

### The "Order of Magnitude" Concept

In engineering, being within 2-3x of the actual number is often good enough for initial design.

**Why?** Because:
- If you need 10 servers and estimate 15, you're fine
- If you need 10 servers and estimate 1000, your design is fundamentally wrong

The goal is to avoid being off by 10x or 100x.

---

## 3️⃣ How It Works Internally

### The Estimation Framework

Every back-of-envelope calculation follows this pattern:

```
1. Identify what you're estimating (storage, bandwidth, servers, etc.)
2. Break it down into smaller, estimable pieces
3. Use known constants and reference numbers
4. Multiply/divide to get your answer
5. Sanity check: Does this make sense?
```

### Essential Numbers to Memorize

These are the building blocks of all estimations:

#### Time Units
```
1 second = 1,000 milliseconds (ms)
1 millisecond = 1,000 microseconds (μs)
1 microsecond = 1,000 nanoseconds (ns)

Seconds in a day: 86,400 ≈ 100,000 (10^5) for estimation
Seconds in a month: ~2.5 million ≈ 2.5 × 10^6
Seconds in a year: ~31 million ≈ 3 × 10^7
```

#### Data Size Units
```
1 Byte = 8 bits
1 KB (Kilobyte) = 1,000 bytes ≈ 10^3 bytes
1 MB (Megabyte) = 1,000 KB ≈ 10^6 bytes
1 GB (Gigabyte) = 1,000 MB ≈ 10^9 bytes
1 TB (Terabyte) = 1,000 GB ≈ 10^12 bytes
1 PB (Petabyte) = 1,000 TB ≈ 10^15 bytes
```

#### Typical Data Sizes
```
1 character (ASCII) = 1 byte
1 character (Unicode/UTF-8) = 1-4 bytes (use 2-3 for estimation)
Average English word = 5 characters
1 tweet (280 chars) ≈ 300 bytes (with metadata)
1 web page ≈ 2 MB (with images, scripts)
1 photo (compressed JPEG) ≈ 200 KB - 2 MB
1 minute of video (720p) ≈ 50-100 MB
1 minute of video (4K) ≈ 300-500 MB
```

#### Latency Numbers (Critical for Interviews)
```
L1 cache reference: 0.5 ns
L2 cache reference: 7 ns
RAM reference: 100 ns
SSD random read: 150 μs (150,000 ns)
HDD seek: 10 ms (10,000,000 ns)
Network round trip (same datacenter): 0.5 ms
Network round trip (cross-country): 30-50 ms
Network round trip (cross-continent): 100-150 ms
```

#### Throughput Numbers
```
SSD sequential read: 500 MB/s - 3 GB/s
HDD sequential read: 100-200 MB/s
1 Gbps network: 125 MB/s theoretical
10 Gbps network: 1.25 GB/s theoretical
Typical web server: 1,000-10,000 requests/second
Database (simple queries): 10,000-100,000 QPS
Database (complex queries): 100-1,000 QPS
Redis (in-memory): 100,000+ operations/second
```

### The Powers of 2 Table

Memorize these for quick calculations:

```
2^10 = 1,024 ≈ 1 thousand (1 KB)
2^20 = 1,048,576 ≈ 1 million (1 MB)
2^30 = 1,073,741,824 ≈ 1 billion (1 GB)
2^40 ≈ 1 trillion (1 TB)

Quick trick: 2^10 ≈ 10^3
Therefore: 2^n ≈ 10^(n/3.33)
```

---

## 4️⃣ Simulation-First Explanation

Let's walk through a complete estimation for a real interview question.

### Problem: Design a URL Shortener (like bit.ly)

**Step 1: Clarify Requirements**

Ask the interviewer:
- How many URLs shortened per day? Let's assume 100 million new URLs/day
- How long do we keep URLs? Let's assume 10 years
- What's the read:write ratio? Let's assume 100:1 (reads are more common)

**Step 2: Estimate Traffic**

```
Write traffic:
- 100 million new URLs per day
- Per second: 100,000,000 / 86,400 ≈ 100,000,000 / 100,000 = 1,000 writes/second

Read traffic:
- 100:1 ratio
- 1,000 × 100 = 100,000 reads/second
```

**Step 3: Estimate Storage**

```
Each URL record contains:
- Short URL: 7 characters = 7 bytes
- Long URL: average 100 characters = 100 bytes
- Created timestamp: 8 bytes
- User ID (optional): 8 bytes
- Total per record: ~125 bytes, round to 150 bytes for safety

Daily storage:
- 100 million URLs × 150 bytes = 15 billion bytes = 15 GB/day

10-year storage:
- 15 GB × 365 days × 10 years = 54,750 GB ≈ 55 TB
```

**Step 4: Estimate Bandwidth**

```
Incoming (writes):
- 1,000 requests/second × 150 bytes = 150 KB/s
- Negligible

Outgoing (reads):
- 100,000 requests/second × 150 bytes = 15 MB/s
- This is well within a single server's network capacity
```

**Step 5: Estimate Server Count**

```
Assumptions:
- A single web server can handle 10,000 requests/second
- We need 100,000 reads/second

Servers needed:
- 100,000 / 10,000 = 10 servers minimum
- With redundancy (2x): 20 servers
- With headroom for spikes (1.5x): 30 servers
```

**Step 6: Sanity Check**

- 55 TB storage over 10 years? Reasonable, fits on a few database servers
- 30 application servers? Reasonable for a service of this scale
- 100,000 QPS? Challenging but achievable with caching

---

## 5️⃣ How Engineers Actually Use This in Production

### At FAANG Companies

**Google**:
Google engineers use a tool called "Capacity Planning" that automates many estimations. But during design reviews, engineers are expected to provide back-of-envelope calculations to justify their proposals.

**Amazon**:
Amazon's "Working Backwards" process includes capacity estimation. Before building a new service, teams must estimate:
- Expected traffic at launch
- Traffic growth over 1, 2, 5 years
- Infrastructure costs

**Netflix**:
Netflix engineers use what they call "Capacity Analysis" before major events. For example, before a big show release (like Stranger Things), they estimate:
- How many concurrent streams?
- How much bandwidth per region?
- How many encoding servers needed?

### Real Production Workflow

```
1. Product Manager: "We're launching in India, expecting 10 million users"

2. Engineer does estimation:
   - 10 million users
   - 10% daily active = 1 million DAU
   - Average 5 requests per session
   - 5 million requests/day
   - Peak is 10x average = 50 million requests in peak hour
   - 50,000,000 / 3,600 = ~14,000 requests/second peak

3. Engineer proposes:
   - 5 application servers (with 3,000 RPS each)
   - 2 database replicas
   - CDN for static content
   - Redis cache for hot data

4. Team reviews and adjusts based on experience
```

### Tools That Help

While back-of-envelope is manual, tools help validate:

- **Grafana/Prometheus**: Monitor actual traffic to compare with estimates
- **Load testing tools**: Verify servers can handle estimated load
- **Cloud cost calculators**: AWS, GCP, Azure calculators help estimate costs

---

## 6️⃣ How to Implement or Apply It

### Java Utility for Quick Calculations

Here's a utility class that helps with common estimations:

```java
package com.systemdesign.estimation;

/**
 * Utility class for back-of-envelope calculations in system design.
 * All methods use approximations suitable for quick estimation.
 */
public class EstimationUtils {

    // Time constants (in seconds)
    public static final long SECONDS_PER_MINUTE = 60;
    public static final long SECONDS_PER_HOUR = 3_600;
    public static final long SECONDS_PER_DAY = 86_400;      // Use 100,000 for quick math
    public static final long SECONDS_PER_MONTH = 2_592_000; // ~2.5 million
    public static final long SECONDS_PER_YEAR = 31_536_000; // ~31 million

    // Size constants (in bytes)
    public static final long KB = 1_000L;
    public static final long MB = 1_000_000L;
    public static final long GB = 1_000_000_000L;
    public static final long TB = 1_000_000_000_000L;
    public static final long PB = 1_000_000_000_000_000L;

    /**
     * Converts daily requests to requests per second.
     * Uses approximation: 1 day ≈ 100,000 seconds for easy mental math.
     *
     * @param dailyRequests number of requests per day
     * @return approximate requests per second
     */
    public static long dailyToPerSecond(long dailyRequests) {
        // Exact: dailyRequests / 86,400
        // Approximation for mental math: dailyRequests / 100,000
        return dailyRequests / SECONDS_PER_DAY;
    }

    /**
     * Calculates storage needed for a given number of records.
     *
     * @param recordCount number of records
     * @param bytesPerRecord size of each record in bytes
     * @return total storage in bytes
     */
    public static long calculateStorage(long recordCount, long bytesPerRecord) {
        return recordCount * bytesPerRecord;
    }

    /**
     * Formats bytes into human-readable format.
     *
     * @param bytes number of bytes
     * @return formatted string (e.g., "1.5 TB")
     */
    public static String formatBytes(long bytes) {
        if (bytes >= PB) {
            return String.format("%.2f PB", bytes / (double) PB);
        } else if (bytes >= TB) {
            return String.format("%.2f TB", bytes / (double) TB);
        } else if (bytes >= GB) {
            return String.format("%.2f GB", bytes / (double) GB);
        } else if (bytes >= MB) {
            return String.format("%.2f MB", bytes / (double) MB);
        } else if (bytes >= KB) {
            return String.format("%.2f KB", bytes / (double) KB);
        } else {
            return bytes + " bytes";
        }
    }

    /**
     * Estimates number of servers needed given traffic and capacity.
     *
     * @param requestsPerSecond expected requests per second
     * @param serverCapacity requests per second a single server can handle
     * @param redundancyFactor multiplier for redundancy (e.g., 2.0 for 2x)
     * @return number of servers needed
     */
    public static int estimateServers(
            long requestsPerSecond,
            long serverCapacity,
            double redundancyFactor) {

        double serversNeeded = (double) requestsPerSecond / serverCapacity;
        return (int) Math.ceil(serversNeeded * redundancyFactor);
    }

    /**
     * Calculates bandwidth needed for given traffic.
     *
     * @param requestsPerSecond number of requests per second
     * @param bytesPerRequest average size of each request/response
     * @return bandwidth in bytes per second
     */
    public static long calculateBandwidth(long requestsPerSecond, long bytesPerRequest) {
        return requestsPerSecond * bytesPerRequest;
    }

    /**
     * Estimates QPS (Queries Per Second) from daily active users.
     *
     * @param dailyActiveUsers number of daily active users
     * @param actionsPerUser average actions per user per day
     * @param peakMultiplier multiplier for peak traffic (e.g., 3.0 for 3x average)
     * @return peak QPS estimate
     */
    public static long estimatePeakQPS(
            long dailyActiveUsers,
            int actionsPerUser,
            double peakMultiplier) {

        long dailyRequests = dailyActiveUsers * actionsPerUser;
        long averageQPS = dailyToPerSecond(dailyRequests);
        return (long) (averageQPS * peakMultiplier);
    }
}
```

### Example Usage: URL Shortener Estimation

```java
package com.systemdesign.estimation;

/**
 * Example: Back-of-envelope calculation for URL Shortener.
 */
public class URLShortenerEstimation {

    public static void main(String[] args) {
        System.out.println("=== URL Shortener Capacity Estimation ===\n");

        // Given requirements
        long newUrlsPerDay = 100_000_000L;  // 100 million
        int retentionYears = 10;
        int readWriteRatio = 100;

        // Step 1: Traffic estimation
        long writesPerSecond = EstimationUtils.dailyToPerSecond(newUrlsPerDay);
        long readsPerSecond = writesPerSecond * readWriteRatio;

        System.out.println("Traffic Estimation:");
        System.out.println("  Writes per second: " + writesPerSecond);
        System.out.println("  Reads per second: " + readsPerSecond);
        System.out.println();

        // Step 2: Storage estimation
        long bytesPerRecord = 150;  // Short URL + Long URL + metadata
        long recordsPerYear = newUrlsPerDay * 365;
        long totalRecords = recordsPerYear * retentionYears;
        long totalStorage = EstimationUtils.calculateStorage(totalRecords, bytesPerRecord);

        System.out.println("Storage Estimation:");
        System.out.println("  Bytes per record: " + bytesPerRecord);
        System.out.println("  Total records (10 years): " + totalRecords);
        System.out.println("  Total storage: " + EstimationUtils.formatBytes(totalStorage));
        System.out.println();

        // Step 3: Bandwidth estimation
        long readBandwidth = EstimationUtils.calculateBandwidth(readsPerSecond, bytesPerRecord);
        long writeBandwidth = EstimationUtils.calculateBandwidth(writesPerSecond, bytesPerRecord);

        System.out.println("Bandwidth Estimation:");
        System.out.println("  Read bandwidth: " + EstimationUtils.formatBytes(readBandwidth) + "/s");
        System.out.println("  Write bandwidth: " + EstimationUtils.formatBytes(writeBandwidth) + "/s");
        System.out.println();

        // Step 4: Server estimation
        long serverCapacity = 10_000;  // Requests per second per server
        int servers = EstimationUtils.estimateServers(readsPerSecond, serverCapacity, 2.0);

        System.out.println("Server Estimation:");
        System.out.println("  Server capacity: " + serverCapacity + " RPS");
        System.out.println("  Servers needed (with 2x redundancy): " + servers);
        System.out.println();

        // Step 5: Summary
        System.out.println("=== Summary ===");
        System.out.println("For a URL shortener with 100M new URLs/day:");
        System.out.println("  - Peak read traffic: ~" + readsPerSecond + " RPS");
        System.out.println("  - Storage (10 years): ~" + EstimationUtils.formatBytes(totalStorage));
        System.out.println("  - Application servers: ~" + servers);
        System.out.println("  - This is feasible with standard infrastructure");
    }
}
```

**Output:**
```
=== URL Shortener Capacity Estimation ===

Traffic Estimation:
  Writes per second: 1157
  Reads per second: 115700

Storage Estimation:
  Bytes per record: 150
  Total records (10 years): 365000000000
  Total storage: 54.75 TB

Bandwidth Estimation:
  Read bandwidth: 17.36 MB/s
  Write bandwidth: 173.55 KB/s

Server Estimation:
  Server capacity: 10000 RPS
  Servers needed (with 2x redundancy): 24

=== Summary ===
For a URL shortener with 100M new URLs/day:
  - Peak read traffic: ~115700 RPS
  - Storage (10 years): ~54.75 TB
  - Application servers: ~24
  - This is feasible with standard infrastructure
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

#### 1. Forgetting Peak vs Average

**Wrong:**
```
100 million requests/day ÷ 86,400 seconds = 1,157 RPS
"We need servers for 1,157 RPS"
```

**Right:**
```
Average: 1,157 RPS
Peak (during busy hours): 3x to 10x average
Design for: 3,500 - 11,570 RPS
```

Traffic is never evenly distributed. Design for peak, not average.

#### 2. Ignoring Metadata Overhead

**Wrong:**
```
Storing 1 million tweets
280 characters × 1 million = 280 MB
```

**Right:**
```
Each tweet has:
- Text: 280 bytes
- User ID: 8 bytes
- Timestamp: 8 bytes
- Tweet ID: 8 bytes
- Indexes, replication overhead: 2x
Total: ~600 bytes × 1 million = 600 MB
```

Always account for metadata, indexes, and replication.

#### 3. Confusing Bits and Bytes

**Wrong:**
```
1 Gbps network = 1 GB/s throughput
```

**Right:**
```
1 Gbps = 1 Gigabit per second
1 Gigabit = 1/8 Gigabyte
1 Gbps = 125 MB/s (theoretical max)
Practical: ~100 MB/s due to overhead
```

Network speeds are in bits. Storage is in bytes. Always convert.

#### 4. Not Accounting for Growth

**Wrong:**
```
We have 1 million users today.
Design for 1 million users.
```

**Right:**
```
We have 1 million users today.
Expected growth: 2x per year
Design for 3 years: 1M × 2^3 = 8 million users
Add 50% buffer: 12 million users
```

### Performance Gotchas

#### Database Limitations

```
MySQL single instance: 
- Simple queries: 10,000-50,000 QPS
- Complex joins: 100-1,000 QPS
- Write-heavy: 1,000-10,000 QPS

If your estimation shows 100,000 QPS, you need:
- Sharding (multiple databases)
- Caching (Redis/Memcached)
- Read replicas
```

#### Network Bandwidth

```
Cross-datacenter replication:
- Latency: 50-100ms
- Bandwidth: Often limited to 1-10 Gbps

If you're replicating 100 MB/s of data:
- 1 Gbps link is at 80% capacity
- You need to account for burst traffic
```

---

## 8️⃣ When NOT to Use This

### Anti-Patterns

#### 1. Over-Precision

**Don't do this:**
```
"We need exactly 17.3 servers"
"Storage will be 47.832 TB"
```

Back-of-envelope is for order of magnitude. Saying "15-20 servers" or "~50 TB" is appropriate.

#### 2. Premature Optimization

If you're building an MVP for 1,000 users, you don't need to calculate for 1 billion users. Start simple, measure, then scale.

#### 3. Ignoring Context

Not every interview or design needs detailed calculations. If the interviewer wants to focus on API design or data modeling, don't spend 10 minutes on capacity estimation.

### When Simple Heuristics Are Better

For small-scale systems:
- Under 1,000 users: A single server is probably fine
- Under 100 GB data: A single database is probably fine
- Under 100 RPS: No need for complex load balancing

---

## 9️⃣ Comparison with Alternatives

### Back-of-Envelope vs Detailed Capacity Planning

| Aspect | Back-of-Envelope | Detailed Planning |
|--------|------------------|-------------------|
| Time | 5-10 minutes | Days to weeks |
| Accuracy | Within 2-5x | Within 10-20% |
| Use case | Interviews, initial design | Production deployment |
| Tools | Mental math, calculator | Spreadsheets, simulators |
| When to use | Early design phase | Before major launches |

### Back-of-Envelope vs Load Testing

| Aspect | Back-of-Envelope | Load Testing |
|--------|------------------|--------------|
| Timing | Before building | After building |
| Cost | Free | Requires infrastructure |
| Accuracy | Theoretical | Empirical |
| Catches | Order of magnitude errors | Actual bottlenecks |
| Limitation | Doesn't find code bugs | Expensive to run |

**Best practice**: Use back-of-envelope first to validate design, then load test to verify.

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q1: How would you estimate the storage needed for a photo-sharing app with 10 million users?**

**Answer:**
```
Assumptions:
- 10 million users
- 20% are active uploaders
- Active users upload 2 photos/week
- Average photo size: 500 KB (compressed)

Weekly uploads:
- 10M × 0.2 × 2 = 4 million photos/week

Weekly storage:
- 4M × 500 KB = 2 TB/week

Yearly storage:
- 2 TB × 52 = 104 TB/year

With replication (3x):
- 312 TB/year

This is feasible with cloud storage (S3, GCS).
```

**Q2: What's the difference between QPS and RPS?**

**Answer:**
- **QPS (Queries Per Second)**: Usually refers to database queries
- **RPS (Requests Per Second)**: Usually refers to HTTP requests

One HTTP request might generate multiple database queries. For example:
- 1 page load (1 RPS) might query user data, posts, and comments (3 QPS)

In estimation, clarify which you're calculating.

### L5 (Senior) Questions

**Q3: You're designing a real-time chat system. How do you estimate the number of WebSocket connections needed?**

**Answer:**
```
Assumptions:
- 50 million registered users
- 10% concurrent at peak = 5 million connections
- Each server handles 50,000 WebSocket connections

Servers needed:
- 5M / 50K = 100 servers

Memory per connection:
- ~10 KB per WebSocket connection
- 50K connections × 10 KB = 500 MB per server
- This is manageable

Bandwidth consideration:
- Average message: 200 bytes
- 1 message/second per active user
- 5M × 200 bytes = 1 GB/s total
- Per server: 10 MB/s (well within limits)
```

**Q4: How do you estimate if a single Redis instance can handle your caching needs?**

**Answer:**
```
Redis single instance limits:
- Operations: 100,000+ ops/second
- Memory: Up to ~100 GB practical limit
- Network: 1-10 Gbps

Check 1: Operations
- If you need 50,000 cache reads/second → Single instance OK
- If you need 500,000 → Need Redis Cluster

Check 2: Memory
- 10 million cached items × 1 KB each = 10 GB → Single instance OK
- 1 billion items × 1 KB = 1 TB → Need sharding

Check 3: Network
- 50,000 ops × 1 KB = 50 MB/s → Single instance OK
```

### L6 (Staff) Questions

**Q5: How would you estimate the cost of running a service at scale?**

**Answer:**
```
Cost components:
1. Compute: $0.05/hour per vCPU (varies by cloud)
2. Storage: $0.02/GB/month for SSD
3. Network: $0.01-0.10/GB egress
4. Managed services: Database, cache, etc.

Example: URL Shortener at scale
- 30 servers × 4 vCPU × $0.05 × 720 hours = $4,320/month (compute)
- 55 TB × $0.02 = $1,100/month (storage)
- 15 MB/s × 2.6M seconds × $0.05/GB = ~$2,000/month (network)
- Database, monitoring, etc.: ~$2,000/month

Total: ~$10,000/month

This seems reasonable for a service handling 100M URLs/day.
```

**Q6: How do you handle uncertainty in your estimates?**

**Answer:**
```
1. State assumptions explicitly
   "Assuming 100:1 read/write ratio..."

2. Provide ranges, not point estimates
   "We need 20-40 servers depending on traffic patterns"

3. Identify the most uncertain variables
   "The biggest unknown is user growth rate"

4. Design for flexibility
   "Start with 20 servers, auto-scale to 60"

5. Plan for measurement
   "We'll validate with load testing before launch"
```

---

## 1️⃣1️⃣ One Clean Mental Summary

Back-of-envelope calculations are quick, rough estimates that help you validate system designs before building them. The goal is not precision but avoiding order-of-magnitude errors. Memorize key numbers (seconds in a day ≈ 100,000, 1 GB = 10^9 bytes, typical server handles 10,000 RPS), break problems into smaller pieces, and always sanity-check your results. In interviews, showing your estimation process demonstrates engineering maturity and helps justify your design decisions. Start with traffic, then storage, then bandwidth, then servers. Always account for peak traffic (3-10x average), metadata overhead, and growth over time.

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                ESTIMATION CHEAT SHEET                        │
├─────────────────────────────────────────────────────────────┤
│ TIME                                                         │
│   1 day ≈ 100,000 seconds (use 10^5)                        │
│   1 month ≈ 2.5 million seconds                             │
│   1 year ≈ 30 million seconds                               │
├─────────────────────────────────────────────────────────────┤
│ DATA                                                         │
│   1 KB = 10^3 bytes    1 MB = 10^6 bytes                    │
│   1 GB = 10^9 bytes    1 TB = 10^12 bytes                   │
├─────────────────────────────────────────────────────────────┤
│ TYPICAL SIZES                                                │
│   Tweet: 300 bytes     Photo: 500 KB                        │
│   Web page: 2 MB       1 min video (720p): 100 MB           │
├─────────────────────────────────────────────────────────────┤
│ LATENCY                                                      │
│   RAM: 100 ns          SSD: 150 μs                          │
│   Same datacenter: 0.5 ms   Cross-country: 50 ms            │
├─────────────────────────────────────────────────────────────┤
│ THROUGHPUT                                                   │
│   Web server: 1,000-10,000 RPS                              │
│   Database: 10,000-100,000 simple QPS                       │
│   Redis: 100,000+ ops/sec                                   │
├─────────────────────────────────────────────────────────────┤
│ FORMULA                                                      │
│   Daily → Per second: divide by 100,000                     │
│   Peak traffic: 3-10x average                               │
│   Storage: records × bytes_per_record × replication_factor  │
│   Servers: peak_RPS / server_capacity × redundancy          │
└─────────────────────────────────────────────────────────────┘
```

