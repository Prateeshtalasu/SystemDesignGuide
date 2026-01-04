# Key-Value Store - Production Deep Dives: Core

## STEP 6: ASYNC, MESSAGING & EVENT FLOW

### Is Async Required?

**Answer: Not Required for Core Operations**

**Why:**
- Key-value operations are synchronous (GET/SET)
- Replication is async (acceptable)
- Persistence is async (background)

**What Uses Async:**
- Replication: Async stream of commands
- Persistence: Background RDB/AOF writes
- Pub/Sub: Async message delivery

**Replication Flow:**
```
Master receives SET command
  ↓
Master writes to memory
  ↓
Master sends to replicas (async)
  ↓
Replicas apply command
  ↓
Replication complete (eventual consistency)
```

**Persistence Flow:**
```
Master receives SET command
  ↓
Master writes to memory
  ↓
Master appends to AOF (async, based on appendfsync)
  ↓
Background process syncs to disk
```

---

## STEP 7: CACHING

### Internal Caching

**Key-Value Store IS a Cache:**
- All data in memory
- Fast lookups (hash table)
- Eviction policies (LRU, LFU)

**No External Cache Needed:**
- Data already in memory
- Sub-millisecond latency
- No need for additional caching layer

**Optimizations:**
- Key encoding (small integers, short strings)
- Data structure optimization (ziplist for small collections)
- Memory pooling (reduce allocations)

---

## STEP 8: SEARCH

**Not Required:**
- Key-value store is accessed by key (not searched)
- No full-text search
- No complex queries

**If Needed:**
- Use external search engine (Elasticsearch)
- Index keys/values separately
- Query search engine, then fetch from key-value store

---

## Summary

**Async:**
- Replication: Async (eventual consistency)
- Persistence: Async (background writes)
- Core operations: Synchronous

**Caching:**
- Key-value store is the cache (in-memory)
- No external cache needed

**Search:**
- Not required (key-based access)

