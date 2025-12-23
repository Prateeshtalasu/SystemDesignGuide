# Typeahead / Autocomplete - Problem & Requirements

## What is Typeahead/Autocomplete?

Typeahead (also called autocomplete or search suggestions) is a feature that predicts and displays search queries as users type. When you start typing in Google's search box and see suggestions appear, that's typeahead in action.

**Example:**
- User types: "how to"
- System suggests:
  1. "how to tie a tie"
  2. "how to lose weight"
  3. "how to make money online"
  4. "how to screenshot on mac"

### Why Does This Exist?

1. **Faster search**: Users don't need to type complete queries
2. **Spelling assistance**: Correct typos before searching
3. **Query discovery**: Users discover popular/relevant queries
4. **Reduced server load**: Fewer malformed queries to process
5. **Better UX**: Feels responsive and intelligent

### What Breaks Without It?

- Users type more, make more typos
- Search engines process more invalid queries
- Users miss relevant search terms they didn't know
- Overall search experience feels dated

---

## Clarifying Questions (Ask the Interviewer)

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the suggestion source? | Determines data pipeline | Search query logs |
| How many suggestions per query? | Response size | 5-10 suggestions |
| What's the latency requirement? | Critical for UX | < 100ms |
| Do we need personalization? | Complexity increase | Yes, basic personalization |
| Multi-language support? | Data structure choice | English only for now |
| How fresh should suggestions be? | Update frequency | Hourly updates |
| Mobile vs desktop? | UI considerations | Both, same backend |

---

## Functional Requirements

### Core Features (Must Have)

1. **Real-time Suggestions**
   - Return suggestions as user types
   - Update with each keystroke
   - Minimum 2-3 characters before suggestions

2. **Ranked Results**
   - Sort by relevance/popularity
   - Show most likely completions first
   - Consider recency of queries

3. **Prefix Matching**
   - Match beginning of words
   - Support multi-word queries
   - Handle spaces correctly

4. **Fast Response**
   - Return results in < 100ms
   - Handle high concurrent load
   - Graceful degradation under load

### Secondary Features (Nice to Have)

5. **Personalization**
   - Consider user's search history
   - Boost recently searched terms
   - Location-based suggestions

6. **Trending Queries**
   - Highlight currently trending searches
   - Time-decay for popularity

7. **Spell Correction**
   - Suggest corrections for typos
   - "Did you mean..." functionality

8. **Category Hints**
   - Show query categories (e.g., "in Movies")
   - Help users refine searches

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Response latency | < 50ms (p99 < 100ms) | Must feel instant |
| Throughput | 100,000 QPS | High-traffic feature |
| Availability | 99.99% | Core user experience |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| Daily active users | 500 million | Large search engine scale |
| Searches per user per day | 5 | Average usage |
| Keystrokes per search | 10 | Average query length |
| Suggestion requests/day | 25 billion | 500M × 5 × 10 |
| Peak QPS | 500,000 | 3x average during peak hours |

### Data Requirements

| Metric | Value |
|--------|-------|
| Unique queries to store | 5 billion |
| Average query length | 20 characters |
| Suggestion freshness | Updated hourly |
| Historical data | 30 days |

---

## What's Out of Scope

1. **Full-text search**: Only prefix matching, not semantic search
2. **Image/video suggestions**: Text only
3. **Voice input**: Keyboard input only
4. **Complex NLP**: No intent understanding
5. **Advertising integration**: No sponsored suggestions

---

## System Constraints

### Technical Constraints

1. **Prefix-based**: Must match from beginning of query
2. **Case-insensitive**: "How" matches "how"
3. **Unicode support**: Handle international characters
4. **Max query length**: 100 characters

### Business Constraints

1. **Content filtering**: No offensive suggestions
2. **Legal compliance**: Filter illegal content
3. **Privacy**: Don't expose individual user queries

---

## Key Challenges

### 1. Latency Challenge

Users expect suggestions to appear instantly. Any delay > 100ms feels sluggish.

**Why it's hard:**
- Millions of queries to search through
- Must happen on every keystroke
- Global user base (network latency)

### 2. Scale Challenge

With 500K QPS, traditional database queries won't work.

**Why it's hard:**
- Can't query database on every keystroke
- Need in-memory data structures
- Must distribute across many servers

### 3. Freshness Challenge

New trending topics should appear quickly.

**Why it's hard:**
- Can't rebuild entire index for each update
- Need incremental updates
- Balance freshness vs stability

### 4. Ranking Challenge

Must show most relevant suggestions first.

**Why it's hard:**
- Popularity changes over time
- Different users want different results
- Must compute rankings quickly

---

## Comparison with Other Systems

| Aspect | URL Shortener | Pastebin | Typeahead |
|--------|---------------|----------|-----------|
| Read:Write ratio | 100:1 | 10:1 | 10000:1 (read-heavy) |
| Latency requirement | < 50ms | < 100ms | < 50ms |
| Data structure | Key-value | Key-value | Trie/Prefix tree |
| Main challenge | Throughput | Storage | Latency + Scale |
| Update frequency | Real-time | Real-time | Batch (hourly) |

---

## User Stories

### Story 1: User Searches for Information
```
As a user,
I want to see search suggestions as I type,
So that I can find what I'm looking for faster.

Acceptance Criteria:
- Suggestions appear within 100ms
- At least 5 relevant suggestions shown
- Suggestions update with each keystroke
- I can click a suggestion to search
```

### Story 2: User Discovers Trending Topics
```
As a user,
I want to see trending searches,
So that I can discover what's popular.

Acceptance Criteria:
- Trending queries are highlighted
- Suggestions reflect current events
- Updates within 1 hour of trend starting
```

### Story 3: User Gets Personalized Suggestions
```
As a logged-in user,
I want suggestions based on my history,
So that I can quickly re-find previous searches.

Acceptance Criteria:
- My recent searches appear in suggestions
- Personalized results ranked higher
- Can clear my search history
```

---

## Data Flow Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATA COLLECTION                                  │
│                                                                          │
│   Search Logs ──────> Aggregation ──────> Filtering ──────> Ranking     │
│   (billions/day)      (count queries)    (remove bad)      (by popularity)│
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         INDEX BUILDING                                   │
│                                                                          │
│   Ranked Queries ──────> Build Trie ──────> Distribute to Servers       │
│   (top 5 billion)        (in-memory)        (replicate globally)        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         QUERY SERVING                                    │
│                                                                          │
│   User Types ──────> Trie Lookup ──────> Rank Results ──────> Return    │
│   "how to"           O(prefix length)    (top 10)            (< 50ms)   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| P50 latency | < 20ms | Percentile monitoring |
| P99 latency | < 100ms | Percentile monitoring |
| Suggestion click rate | > 30% | Clicks on suggestions / total searches |
| Query completion rate | > 50% | Searches using suggestions |
| System availability | 99.99% | Uptime monitoring |

---

## Interview Tips

### What Interviewers Look For

1. **Data structure knowledge**: Trie understanding is essential
2. **Scale awareness**: This is a very high-QPS system
3. **Latency focus**: Every millisecond matters
4. **Trade-off analysis**: Freshness vs consistency vs latency

### Common Mistakes

1. **Using database queries**: Too slow for real-time suggestions
2. **Ignoring caching**: Must cache aggressively
3. **Single server design**: Won't handle the scale
4. **Complex ranking at query time**: Pre-compute rankings

### Good Follow-up Questions

- "How would you handle a sudden trending topic?"
- "What if a user types very fast?"
- "How do you handle offensive suggestions?"

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary use case | Real-time search suggestions |
| Scale | 500K QPS peak |
| Latency target | < 50ms p50, < 100ms p99 |
| Data structure | Trie (prefix tree) |
| Update frequency | Hourly batch updates |
| Suggestions per query | 5-10 |
| Personalization | Basic (recent searches) |
| Main challenge | Latency at massive scale |

