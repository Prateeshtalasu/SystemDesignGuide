# 🎯 PHASE 5.5: PRACTICAL DATA STRUCTURES (Week 5)

**Goal**: Advanced DS used in production systems

## Topics:

1. **Bloom Filters**
   - How it works (hash functions + bit array)
   - False positive rate calculation
   - Use cases (cache existence check, avoiding DB hits)
   - Google BigTable usage

2. **HyperLogLog**
   - Cardinality estimation
   - Use cases (unique visitors, unique IPs)
   - Redis PFADD/PFCOUNT

3. **Count-Min Sketch**
   - Frequency estimation
   - Use cases (trending topics, heavy hitters)
   - Trade-offs vs exact counting

4. **Trie (Prefix Tree)**
   - Autocomplete systems
   - IP routing tables
   - Implementation

5. **Skip List**
   - Redis sorted sets internals
   - vs Balanced trees
   - Probabilistic data structure

6. **Consistent Hashing**
   - Ring-based approach
   - Virtual nodes
   - Use cases (cache distribution, load balancing)
   - DynamoDB partitioning

7. **Segment Trees & Fenwick Trees**
   - Range query problems
   - Use cases (range sum, min, max queries)
   - Time complexity analysis

8. **Merkle Trees**
   - Hash tree structure
   - Use cases (blockchain, Git, Cassandra)
   - Efficient data verification

9. **Ring Buffer (Circular Buffer)**
   - Fixed-size buffer
   - Use cases (logging, streaming)
   - Implementation considerations

## Deliverable
Can implement Bloom filter and explain Redis internals

