# Web Crawler - Problem & Requirements

## What is a Web Crawler?

A Web Crawler (also called a spider or bot) is a system that systematically browses the internet to discover and download web pages. It starts with a set of seed URLs, downloads those pages, extracts links from them, and adds new URLs to a queue for future crawling.

**Example:**

```
Start: https://en.wikipedia.org
  ↓ Download page, extract links
Found: /wiki/Computer_science, /wiki/Internet, /wiki/Web_browser
  ↓ Add to queue, download each
Found: /wiki/Algorithm, /wiki/Programming, ...
  ↓ Continue recursively
```

### Why Does This Exist?

1. **Search Engine Indexing**: Google, Bing need to discover and index web pages
2. **Web Archiving**: Internet Archive preserves historical web content
3. **Data Mining**: Extract structured data from websites
4. **Price Monitoring**: Track product prices across e-commerce sites
5. **SEO Analysis**: Analyze website structure and content
6. **Security Scanning**: Detect vulnerabilities across web properties

### What Breaks Without It?

- Search engines can't find new content
- Web archives become stale
- Price comparison sites have outdated data
- Security vulnerabilities go undetected
- New websites remain undiscoverable

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question                                  | Why It Matters                           | Assumed Answer                                     |
| ----------------------------------------- | ---------------------------------------- | -------------------------------------------------- |
| What's the scale (pages to crawl)?        | Determines infrastructure size           | 1 billion pages/month                              |
| What's the crawl frequency?               | Affects freshness and resource usage     | Re-crawl important pages daily, others weekly      |
| Do we need to render JavaScript?          | Adds significant complexity              | No, HTML only for now                              |
| What content types?                       | Affects parsing logic                    | HTML, PDF, plain text                              |
| Do we need to respect robots.txt?         | Legal and ethical requirement            | Yes, mandatory                                     |
| Is this for a search engine or specific use case? | Affects breadth vs depth        | General-purpose for search engine                  |
| What's the budget for bandwidth?          | Major cost driver                        | 10 Gbps sustained                                  |
| Do we need to store raw HTML?             | Storage requirements                     | Yes, for re-indexing                               |

---

## Functional Requirements

### Core Features (Must Have)

1. **URL Discovery**
   - Start from seed URLs
   - Extract links from downloaded pages
   - Discover new URLs continuously

2. **Page Download**
   - Fetch web pages via HTTP/HTTPS
   - Handle redirects (301, 302, 307)
   - Support various content types (HTML, PDF)

3. **Politeness**
   - Respect robots.txt directives
   - Implement crawl delays per domain
   - Limit concurrent connections per domain

4. **URL Frontier Management**
   - Maintain queue of URLs to crawl
   - Prioritize important/fresh URLs
   - Avoid re-crawling unchanged pages

5. **Duplicate Detection**
   - Detect duplicate URLs (normalization)
   - Detect duplicate content (fingerprinting)
   - Handle URL variations (www vs non-www)

6. **Content Storage**
   - Store downloaded pages
   - Store metadata (headers, timestamps)
   - Support efficient retrieval

### Secondary Features (Nice to Have)

7. **Freshness Management**
   - Track page change frequency
   - Prioritize frequently changing pages
   - Adaptive re-crawl scheduling

8. **Quality Scoring**
   - Estimate page importance
   - Prioritize high-quality domains
   - Deprioritize spam sites

9. **Distributed Coordination**
   - Coordinate across multiple crawlers
   - Avoid duplicate work
   - Handle crawler failures

---

## Non-Functional Requirements

### Performance

| Metric                 | Target            | Rationale                               |
| ---------------------- | ----------------- | --------------------------------------- |
| Crawl rate             | 1,000 pages/sec   | 1B pages/month ≈ 385 pages/sec, 3x buffer |
| Page download latency  | < 10 seconds      | Timeout for slow servers                |
| URL frontier size      | 100M+ URLs        | Queue for continuous crawling           |
| Throughput             | 10 Gbps           | Bandwidth for page downloads            |

### Scale

| Metric                 | Value             | Calculation                             |
| ---------------------- | ----------------- | --------------------------------------- |
| Pages/month            | 1 billion         | Given assumption                        |
| Pages/day              | 33 million        | 1B / 30 days                            |
| Pages/second           | 385               | 33M / 86,400 seconds                    |
| Unique domains         | 100 million       | Estimated web diversity                 |

### Reliability

- **No duplicate crawls**: Same URL shouldn't be crawled twice in short period
- **Graceful degradation**: Continue crawling if some components fail
- **Resumable**: Recover from crashes without losing progress

### Compliance

- **robots.txt**: Must respect crawl directives
- **Crawl-delay**: Honor delay requests from websites
- **Legal**: Comply with terms of service where applicable

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **JavaScript Rendering**: No headless browser execution
2. **Login/Authentication**: No crawling behind login walls
3. **CAPTCHA Solving**: Skip pages requiring CAPTCHAs
4. **Deep Web**: Only publicly accessible pages
5. **Real-time Crawling**: Not event-driven, batch-oriented
6. **Content Extraction**: Just download, indexing is separate
7. **Image/Video Download**: Text content only

---

## System Constraints

### Technical Constraints

1. **Bandwidth**: Limited by network capacity and cost
   - 10 Gbps = 1.25 GB/s = 108 TB/day

2. **Politeness**: Can't overwhelm target servers
   - Max 1 request/second per domain
   - Must respect robots.txt

3. **DNS**: DNS lookups can be bottleneck
   - Cache DNS responses
   - Use local DNS resolver

4. **Storage**: Must store billions of pages
   - Compressed storage required
   - Efficient retrieval for re-indexing

### Business Constraints

1. **Cost**: Bandwidth and storage are expensive
2. **Legal**: Must not violate terms of service
3. **Reputation**: Aggressive crawling damages brand

---

## Success Metrics

How do we know if the crawler is working well?

| Metric                    | Target      | How to Measure                          |
| ------------------------- | ----------- | --------------------------------------- |
| Crawl rate                | 1,000/sec   | Pages downloaded per second             |
| Success rate              | > 95%       | Successful downloads / total attempts   |
| Freshness                 | < 7 days    | Average age of crawled pages            |
| Coverage                  | > 90%       | % of known URLs crawled                 |
| Duplicate rate            | < 5%        | Duplicate pages / total pages           |
| robots.txt compliance     | 100%        | No violations                           |

---

## User Stories

### Story 1: Initial Crawl of New Domain

```
As a crawler,
I want to discover and download all pages from a new domain,
So that the search engine can index the content.

Acceptance Criteria:
- Fetch robots.txt first
- Respect disallow rules
- Discover all linked pages
- Store pages with metadata
- Complete within reasonable time
```

### Story 2: Re-crawl for Freshness

```
As a crawler,
I want to re-crawl pages that may have changed,
So that the search index stays current.

Acceptance Criteria:
- Prioritize frequently changing pages
- Check Last-Modified headers
- Skip unchanged pages (304 response)
- Update stored content if changed
```

### Story 3: Handle Slow/Failing Servers

```
As a crawler,
I want to handle servers that are slow or failing,
So that crawling continues smoothly.

Acceptance Criteria:
- Timeout after 10 seconds
- Retry failed requests (max 3 times)
- Back off from consistently failing domains
- Don't block queue on slow servers
```

---

## Core Components Overview

### 1. URL Frontier
- Queue of URLs to crawl
- Priority-based scheduling
- Per-domain rate limiting

### 2. Fetcher
- Downloads web pages
- Handles HTTP/HTTPS
- Manages connections and timeouts

### 3. DNS Resolver
- Resolves domain names to IPs
- Caches DNS responses
- Handles DNS failures

### 4. Parser
- Extracts links from HTML
- Normalizes URLs
- Extracts metadata

### 5. Duplicate Detector
- URL deduplication
- Content fingerprinting
- Near-duplicate detection

### 6. Robots.txt Handler
- Fetches and parses robots.txt
- Caches rules per domain
- Enforces crawl restrictions

### 7. Document Store
- Stores downloaded pages
- Handles compression
- Supports retrieval

### 8. Coordinator
- Distributes work across crawlers
- Handles failures
- Monitors progress

---

## Interview Tips

### What Interviewers Look For

1. **Politeness understanding**: Do you know about robots.txt and crawl delays?
2. **Scale awareness**: Can you handle billions of pages?
3. **Duplicate handling**: URL normalization, content fingerprinting

### Common Mistakes

1. **Ignoring politeness**: This is a legal and ethical requirement
2. **Single-threaded thinking**: Must be massively parallel
3. **Forgetting DNS**: DNS can be a major bottleneck
4. **No duplicate detection**: Wastes resources on same content

### Good Follow-up Questions from Interviewer

- "How do you handle a site that returns different content each time?"
- "What if a site has infinite pages (calendar, search results)?"
- "How do you prioritize which pages to crawl first?"

---

## Summary

| Aspect              | Decision                                       |
| ------------------- | ---------------------------------------------- |
| Primary use case    | Discover and download web pages for indexing   |
| Scale               | 1 billion pages/month                          |
| Crawl rate          | 1,000 pages/second                             |
| Politeness          | robots.txt compliance, 1 req/sec per domain    |
| Content types       | HTML, PDF, plain text                          |
| Storage             | Compressed raw HTML + metadata                 |
| Deduplication       | URL normalization + content fingerprinting     |

