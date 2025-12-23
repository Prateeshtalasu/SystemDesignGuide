# Search Engine - Problem & Requirements

## What is a Search Engine?

A Search Engine is a system that crawls the web, indexes content, and allows users to find relevant information by entering queries. When a user types a search query, the system returns ranked results from billions of web pages in milliseconds.

**Example:**

- User Query: `"best pizza restaurants in New York"`
- System Response: Ranked list of 10 web pages with titles, snippets, and URLs

### Why Does This Exist?

1. **Information Discovery**: The web has billions of pages; users need help finding relevant content
2. **Relevance Ranking**: Not all pages are equally useful; ranking surfaces the best results
3. **Speed**: Users expect results in under 500ms despite searching billions of documents
4. **Freshness**: News and content change constantly; search must reflect current information
5. **Monetization**: Search ads are a multi-billion dollar industry

### What Breaks Without It?

- Users can't discover new websites or content
- Information becomes siloed and inaccessible
- E-commerce, news, and knowledge sharing collapse
- Internet navigation becomes impossible at scale

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question                                  | Why It Matters                           | Assumed Answer                                     |
| ----------------------------------------- | ---------------------------------------- | -------------------------------------------------- |
| What's the scale (pages indexed)?         | Determines storage and compute needs     | 10 billion web pages                               |
| What's the expected QPS?                  | Affects infrastructure sizing            | 100,000 queries/second                             |
| Do we need real-time indexing?            | Affects crawler and index pipeline       | Near real-time (minutes for news, days for others) |
| What types of content?                    | Text only vs images, videos              | Text-based web pages only                          |
| Do we need personalization?               | Adds complexity to ranking               | Basic personalization (location, language)         |
| What's the latency target?                | Critical for user experience             | < 500ms for query results                          |
| Do we need spell correction?              | Improves user experience                 | Yes, basic spell correction                        |
| Is this a vertical or horizontal search?  | Affects scope                            | Horizontal (general web search)                    |

---

## Functional Requirements

### Core Features (Must Have)

1. **Web Crawling**
   - Discover and download web pages from the internet
   - Respect robots.txt and crawl politeness
   - Handle different content types (HTML, PDF, etc.)

2. **Indexing**
   - Parse and extract text content from pages
   - Build inverted index for fast keyword lookup
   - Store document metadata (title, URL, timestamp)

3. **Query Processing**
   - Parse user queries into searchable terms
   - Handle multi-word queries and phrases
   - Support boolean operators (AND, OR, NOT)

4. **Ranking**
   - Score documents based on relevance
   - Consider page quality (PageRank-like algorithm)
   - Factor in freshness for time-sensitive queries

5. **Results Display**
   - Return top 10 results per page
   - Show title, URL, and snippet
   - Support pagination for more results

### Secondary Features (Nice to Have)

6. **Spell Correction**
   - Suggest "Did you mean..." for misspelled queries
   - Auto-correct obvious typos

7. **Query Suggestions**
   - Autocomplete as user types
   - Show popular related queries

8. **Filters**
   - Filter by date, domain, content type
   - Safe search filtering

9. **Caching**
   - Cache popular query results
   - Reduce latency for common searches

---

## Non-Functional Requirements

### Performance

| Metric                 | Target            | Rationale                               |
| ---------------------- | ----------------- | --------------------------------------- |
| Query latency          | < 200ms (p50)     | Users expect instant results            |
| Query latency          | < 500ms (p99)     | Worst case still acceptable             |
| Indexing latency       | < 1 hour (news)   | News needs freshness                    |
| Indexing latency       | < 1 week (other)  | General content can be less fresh       |
| Availability           | 99.99%            | Search is critical infrastructure       |

### Scale

| Metric                 | Value             | Calculation                             |
| ---------------------- | ----------------- | --------------------------------------- |
| Pages indexed          | 10 billion        | Given assumption                        |
| Queries per second     | 100,000           | Given assumption                        |
| Index size             | ~100 TB           | 10B pages × 10KB avg compressed         |
| Daily queries          | 8.6 billion       | 100K × 86,400 seconds                   |

### Reliability

- **No data loss**: Index must be durable and recoverable
- **Graceful degradation**: Return partial results if some shards fail
- **Eventual consistency**: Index updates don't need to be instant

### Security

- Prevent query injection attacks
- Rate limiting to prevent abuse
- Filter malicious/spam content from results

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Image/Video Search**: Only text-based web pages
2. **Voice Search**: No speech-to-text processing
3. **Ads System**: No sponsored results or bidding
4. **Knowledge Graph**: No structured data extraction
5. **Local Search**: No location-based business results
6. **Shopping Search**: No product-specific features
7. **News Aggregation**: No dedicated news vertical

---

## System Constraints

### Technical Constraints

1. **Index Size**: Must handle 100+ TB of index data
   - Requires distributed storage across many machines
   - Must fit hot data in memory for fast access

2. **Query Latency**: Sub-500ms response time
   - Must query multiple shards in parallel
   - Caching is critical for popular queries

3. **Crawl Rate**: Must respect website limits
   - Politeness delays between requests
   - robots.txt compliance

### Business Constraints

1. **Global Service**: Must serve users worldwide
2. **Cost-Effective**: Storage and compute costs scale with index size
3. **Legal Compliance**: Must respect DMCA takedown requests

---

## Success Metrics

How do we know if the system is working well?

| Metric                    | Target      | How to Measure                          |
| ------------------------- | ----------- | --------------------------------------- |
| Query success rate        | > 99.9%     | Queries returning results / total       |
| P50 query latency         | < 200ms     | Percentile from monitoring              |
| P99 query latency         | < 500ms     | Percentile from monitoring              |
| Index freshness           | < 1 hour    | Time from page change to index update   |
| Click-through rate        | > 30%       | Users clicking on results               |
| Zero-result rate          | < 5%        | Queries with no results                 |

---

## User Stories

### Story 1: User Searches for Information

```
As a user,
I want to search for "how to make pizza dough",
So that I can find recipes and instructions.

Acceptance Criteria:
- Results appear in under 500ms
- Top results are relevant recipe pages
- Each result shows title, URL, and snippet
- I can click through to the actual page
```

### Story 2: User Searches for Recent News

```
As a user,
I want to search for "latest stock market news",
So that I can see current financial information.

Acceptance Criteria:
- Results include pages from the last few hours
- News articles are ranked by freshness and relevance
- I can filter by date if needed
```

### Story 3: User Misspells Query

```
As a user,
I want to search for "resturant" (misspelled),
So that I still get relevant restaurant results.

Acceptance Criteria:
- System suggests "Did you mean: restaurant?"
- Results still show for the corrected query
- Original query results shown if user prefers
```

---

## Core Components Overview

### 1. Web Crawler
- Discovers URLs and downloads web pages
- Maintains URL frontier (queue of URLs to crawl)
- Respects robots.txt and crawl delays

### 2. Document Processor
- Parses HTML and extracts text
- Removes boilerplate (ads, navigation)
- Extracts metadata (title, description, links)

### 3. Indexer
- Builds inverted index (word → document list)
- Computes document statistics (term frequency)
- Stores document metadata

### 4. Query Processor
- Parses and tokenizes user queries
- Expands queries (synonyms, spell correction)
- Identifies query intent

### 5. Ranking Engine
- Scores documents for relevance
- Combines multiple signals (TF-IDF, PageRank, freshness)
- Applies personalization factors

### 6. Results Server
- Fetches snippets and metadata
- Formats results for display
- Handles pagination

---

## Interview Tips

### What Interviewers Look For

1. **Scale awareness**: Can you handle 10B documents and 100K QPS?
2. **Ranking understanding**: How do you determine relevance?
3. **Trade-off thinking**: Freshness vs completeness, latency vs accuracy

### Common Mistakes

1. **Ignoring scale**: This is a distributed systems problem, not a single-server solution
2. **Oversimplifying ranking**: "Just use TF-IDF" isn't enough for production
3. **Forgetting the crawler**: Many candidates jump straight to indexing

### Good Follow-up Questions from Interviewer

- "How would you handle a celebrity's tweet going viral?" (freshness)
- "What if a shard goes down during a query?" (fault tolerance)
- "How do you prevent spam sites from ranking high?" (quality)

---

## Summary

| Aspect              | Decision                                       |
| ------------------- | ---------------------------------------------- |
| Primary use case    | Search billions of web pages by keyword        |
| Scale               | 10B pages, 100K QPS                            |
| Read:Write ratio    | 1000:1 (queries vs index updates)              |
| Latency target      | < 200ms p50, < 500ms p99                       |
| Availability target | 99.99%                                         |
| Index structure     | Inverted index with sharding                   |
| Ranking approach    | TF-IDF + PageRank + freshness signals          |

