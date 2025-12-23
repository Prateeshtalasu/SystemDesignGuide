# 🎯 PHASE 5.5: PRACTICAL DATA STRUCTURES (Week 5)

**Goal**: Advanced DS used in production systems

**Learning Objectives**:
- Understand probabilistic data structures and their trade-offs
- Implement data structures used in real systems
- Know when to use approximate vs exact algorithms
- Apply these structures to solve real system design problems

**Estimated Time**: 12-15 hours

---

## Topics:

1. **Bloom Filters**
   - How it works (hash functions + bit array)
   - False positive rate calculation
   - Use cases (cache existence check, avoiding DB hits)
   - Google BigTable usage
   - Counting Bloom filters
   - Scalable Bloom filters

2. **HyperLogLog**
   - Cardinality estimation algorithm
   - Use cases (unique visitors, unique IPs)
   - Redis PFADD/PFCOUNT
   - Accuracy vs memory trade-off
   - HyperLogLog++

3. **Count-Min Sketch**
   - Frequency estimation
   - Use cases (trending topics, heavy hitters)
   - Trade-offs vs exact counting
   - Top-K with Count-Min Sketch

4. **Trie (Prefix Tree)**
   - Autocomplete systems
   - IP routing tables (longest prefix match)
   - Implementation in Java
   - Compressed tries (Radix tree)
   - Ternary search tree

5. **Skip List**
   - Redis sorted sets internals
   - vs Balanced trees (why skip lists win)
   - Probabilistic data structure
   - Concurrent skip lists
   - Implementation walkthrough

6. **Consistent Hashing**
   - Ring-based approach
   - Virtual nodes (vnodes)
   - Use cases (cache distribution, load balancing)
   - DynamoDB partitioning
   - Jump consistent hashing
   - Bounded load consistent hashing

7. **Segment Trees & Fenwick Trees**
   - Range query problems
   - Use cases (range sum, min, max queries)
   - Time complexity analysis
   - Lazy propagation
   - 2D segment trees

8. **Merkle Trees**
   - Hash tree structure
   - Use cases (blockchain, Git, Cassandra anti-entropy)
   - Efficient data verification
   - Merkle DAG (IPFS)

9. **Ring Buffer (Circular Buffer)**
   - Fixed-size buffer
   - Use cases (logging, streaming, producer-consumer)
   - Lock-free ring buffers
   - LMAX Disruptor pattern

10. **Quadtree**
    - Spatial indexing
    - Use cases (Uber driver locations, collision detection)
    - Point quadtree vs region quadtree
    - K-d trees comparison
    - Implementation for geo-queries

11. **R-Tree**
    - Bounding box indexing
    - Use cases (spatial databases, GIS)
    - R*-tree optimization
    - PostGIS usage

12. **Geohash**
    - Location encoding
    - Use cases (nearby search, location-based services)
    - Geohash precision levels
    - Geohash + Redis for geo-queries
    - Edge cases and limitations

13. **LSM Tree (Log-Structured Merge Tree)**
    - Write-optimized data structure
    - Use cases (Cassandra, RocksDB, LevelDB)
    - Compaction strategies (size-tiered, leveled)
    - Read amplification vs write amplification
    - SSTable format

14. **B+ Tree Deep Dive**
    - Why databases use B+ trees
    - Internal nodes vs leaf nodes
    - Range queries efficiency
    - B+ tree vs B-tree
    - Page splits and merges

15. **Inverted Index**
    - Full-text search
    - Use cases (Elasticsearch, search engines)
    - TF-IDF scoring
    - Posting lists
    - Index compression

---

## Cross-References:
- **Consistent Hashing**: Used in Phase 4 (Caching) and Phase 9 (Distributed Cache HLD)
- **LSM Tree**: Related to Phase 3 (Database internals)
- **Geohash/Quadtree**: Used in Phase 9 (Uber, Google Maps HLD)

---

## Practice Problems:
1. Implement a Bloom filter with configurable false positive rate
2. Design a trending topics system using Count-Min Sketch
3. Implement consistent hashing with virtual nodes
4. Build a spatial index using Quadtree for nearby driver search

---

## Common Interview Questions:
- "How does a Bloom filter work? Can it have false negatives?"
- "Why does Redis use skip lists instead of balanced trees?"
- "How would you find nearby drivers efficiently?"
- "Explain how consistent hashing handles node failures"

---

## Deliverable
Can implement Bloom filter and explain Redis internals
