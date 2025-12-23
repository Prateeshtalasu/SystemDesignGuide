# 🎯 LSM Tree (Log-Structured Merge Tree)

## 0️⃣ Prerequisites

Before diving into LSM Trees, you need to understand:

### Write-Ahead Log (WAL)
A **WAL** is an append-only file where all writes are recorded before being applied. It ensures durability: if the system crashes, you can replay the log.

```java
// Every write goes to WAL first
wal.append("PUT key1 value1");
wal.append("PUT key2 value2");
wal.append("DELETE key1");
// Then apply to in-memory structure
```

### B-Trees (Previous Topic)
**B-Trees** are the traditional database index structure. They're optimized for reads but have write amplification issues.

### Sequential vs Random I/O
- **Sequential I/O**: Reading/writing consecutive disk blocks (fast, ~100 MB/s on HDD)
- **Random I/O**: Reading/writing scattered disk blocks (slow, ~1 MB/s on HDD)

```
Sequential: [Block 1][Block 2][Block 3][Block 4]  → Fast
Random:     [Block 7]...[Block 2]...[Block 9]... → Slow
```

### Sorted String Table (SSTable)
An **SSTable** is an immutable file containing sorted key-value pairs. Once written, it's never modified.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Core Problem: B-Tree Write Amplification

B-Trees are optimized for reads, but writes are expensive:

```
Insert key into B-Tree:
1. Read the leaf page from disk (1 random read)
2. Modify the page in memory
3. Write the page back to disk (1 random write)
4. If page splits, write multiple pages

For each write: multiple random I/O operations!
```

**Write amplification**: The ratio of actual bytes written to disk vs. bytes in the original write.

```
User writes: 100 bytes
B-Tree writes: 4KB page (minimum)
Write amplification: 40x!
```

### The LSM Tree Solution

LSM Trees optimize for writes by converting random writes to sequential writes:

```
LSM Write Path:
1. Append to WAL (sequential write)
2. Insert into in-memory MemTable
3. When MemTable is full, flush to disk as SSTable (sequential write)
4. Background compaction merges SSTables

All disk writes are sequential!
```

### Real-World Pain Points

**Scenario 1: Time-Series Data**
IoT sensors sending millions of data points per second. B-Tree can't keep up with random writes.

**Scenario 2: Write-Heavy Workloads**
Social media feeds, activity logs, event sourcing. Writes vastly outnumber reads.

**Scenario 3: SSD Optimization**
SSDs have limited write endurance. Lower write amplification = longer SSD life.

### What Breaks Without LSM Trees?

| B-Tree | LSM Tree |
|--------|----------|
| Random writes (slow) | Sequential writes (fast) |
| High write amplification | Lower write amplification |
| ~10K writes/sec | ~100K writes/sec |
| Good read performance | Slightly slower reads |
| In-place updates | Append-only, merge later |

---

## 2️⃣ Intuition and Mental Model

### The Inbox Analogy

Imagine you're organizing mail:

**B-Tree approach** (filing cabinet):
- Each letter goes directly into the correct folder
- Finding a letter is fast
- But filing each letter requires opening the cabinet, finding the folder, inserting...

**LSM Tree approach** (inbox + periodic organization):
- New mail goes into your inbox (fast!)
- Periodically, sort inbox and merge into filing cabinet
- Finding a letter requires checking inbox first, then cabinet

```
┌─────────────────────────────────────────────────────────────────┐
│                    LSM TREE MENTAL MODEL                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Write Path (fast):                                             │
│  1. Drop letter in inbox (MemTable)                             │
│  2. When inbox full, sort and file (flush to SSTable)           │
│                                                                  │
│  Read Path (check multiple places):                             │
│  1. Check inbox (MemTable)                                      │
│  2. Check recent files (Level 0 SSTables)                       │
│  3. Check older files (Level 1, 2, ... SSTables)               │
│                                                                  │
│  Background: Merge and compact files periodically               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Key Insight

```
┌─────────────────────────────────────────────────────────────────┐
│                    LSM TREE KEY INSIGHT                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "Trade read performance for write performance"                 │
│                                                                  │
│  Writes: O(1) amortized (just append to MemTable)              │
│  Reads: O(log n) × number of levels (check each level)         │
│                                                                  │
│  The tradeoff is tunable:                                       │
│  - More levels = better write performance, worse reads          │
│  - Fewer levels = worse write performance, better reads         │
│  - Bloom filters reduce read amplification                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3️⃣ How It Works Internally

### Architecture Overview

```
                    ┌─────────────────┐
                    │    MemTable     │ ← In-memory, sorted
                    │   (Red-Black    │   (writes go here)
                    │    Tree/SkipList)│
                    └────────┬────────┘
                             │ flush when full
                             ▼
              ┌──────────────────────────────┐
              │         Level 0              │ ← Recently flushed
              │  ┌──────┐ ┌──────┐ ┌──────┐ │   (may overlap)
              │  │SST 1 │ │SST 2 │ │SST 3 │ │
              │  └──────┘ └──────┘ └──────┘ │
              └──────────────┬───────────────┘
                             │ compaction
                             ▼
              ┌──────────────────────────────┐
              │         Level 1              │ ← Compacted
              │  ┌─────────────────────────┐ │   (no overlap)
              │  │      SSTable            │ │
              │  └─────────────────────────┘ │
              └──────────────┬───────────────┘
                             │ compaction
                             ▼
              ┌──────────────────────────────┐
              │         Level 2              │ ← Larger, older
              │  ┌─────────────────────────┐ │
              │  │      SSTable            │ │
              │  └─────────────────────────┘ │
              └──────────────────────────────┘
```

### Write Path

```
1. Write to WAL (durability)
2. Insert into MemTable (in-memory sorted structure)
3. When MemTable reaches threshold:
   a. Convert MemTable to immutable
   b. Create new MemTable for new writes
   c. Flush immutable MemTable to Level 0 SSTable
4. Background compaction merges SSTables
```

### Read Path

```
1. Check MemTable (most recent data)
2. Check immutable MemTable (if exists)
3. For each level (0, 1, 2, ...):
   a. Use Bloom filter to check if key might exist
   b. If Bloom filter says "maybe", search SSTable
   c. If found, return value
4. If not found in any level, key doesn't exist
```

### SSTable Format

```
┌────────────────────────────────────────────────────────────┐
│                      SSTable File                          │
├────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────┐ │
│  │                    Data Blocks                        │ │
│  │  ┌─────────────────────────────────────────────────┐ │ │
│  │  │ Block 1: key1→val1, key2→val2, key3→val3, ...  │ │ │
│  │  ├─────────────────────────────────────────────────┤ │ │
│  │  │ Block 2: key100→val100, key101→val101, ...     │ │ │
│  │  ├─────────────────────────────────────────────────┤ │ │
│  │  │ Block N: ...                                    │ │ │
│  │  └─────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────┐ │
│  │                    Index Block                        │ │
│  │  Block 1: first_key=key1, offset=0                   │ │
│  │  Block 2: first_key=key100, offset=4096              │ │
│  │  ...                                                  │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────┐ │
│  │                   Bloom Filter                        │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────┐ │
│  │                     Footer                            │ │
│  │  index_offset, bloom_offset, metadata                │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### Compaction Strategies

**Size-Tiered Compaction (STCS)**:
- Merge SSTables of similar size
- Simple, good for write-heavy workloads
- Can have high space amplification

```
Level 0: [1MB] [1MB] [1MB] [1MB]
              ↓ merge
Level 1: [4MB]
              ↓ merge with existing
Level 1: [4MB] [4MB] [4MB] [4MB]
              ↓ merge
Level 2: [16MB]
```

**Leveled Compaction (LCS)**:
- Each level has size limit (e.g., Level N = 10^N × base)
- SSTables within a level don't overlap
- Better read performance, more write amplification

```
Level 0: [a-m] [d-z] (may overlap)
              ↓ compact
Level 1: [a-c] [d-f] [g-i] [j-l] [m-o] [p-r] [s-u] [v-z]
         (no overlap, sorted)
```

---

## 4️⃣ Simulation: Step-by-Step Walkthrough

### Write Operations

```
Initial state:
MemTable: empty
Level 0: empty
Level 1: empty

Write(A, 1):
MemTable: {A: 1}

Write(C, 3):
MemTable: {A: 1, C: 3}

Write(B, 2):
MemTable: {A: 1, B: 2, C: 3}  (sorted)

Write(D, 4):
MemTable: {A: 1, B: 2, C: 3, D: 4}  (at threshold)

Write(E, 5):  // Triggers flush
1. MemTable becomes immutable
2. New MemTable created
3. Flush immutable to Level 0

MemTable: {E: 5}
Level 0: [SST1: A→1, B→2, C→3, D→4]
```

### Read Operations

```
State:
MemTable: {E: 5, F: 6}
Level 0: [SST1: A→1, B→2, C→3, D→4]
         [SST2: C→30, G→7]  (C was updated!)
Level 1: [SST3: H→8, I→9, J→10]

Read(E):
1. Check MemTable: Found! Return 5

Read(C):
1. Check MemTable: Not found
2. Check Level 0, SST2 (newer): Found C→30! Return 30
   (Don't check SST1, we already found it in newer SSTable)

Read(H):
1. Check MemTable: Not found
2. Check Level 0: Bloom filter says "no" for both → skip
3. Check Level 1, SST3: Found! Return 8

Read(Z):
1. Check MemTable: Not found
2. Check Level 0: Bloom filters say "no" → skip
3. Check Level 1: Bloom filter says "no" → skip
4. Not found, return null
```

### Compaction

```
Before compaction:
Level 0: [SST1: A→1, B→2] [SST2: B→20, C→3] [SST3: A→10, D→4]
Level 1: empty

Compaction (merge all Level 0 to Level 1):
1. Read all SSTables
2. Merge-sort, keeping newest value for each key:
   - A: SST3 has newest (A→10)
   - B: SST2 has newest (B→20)
   - C: SST2 (C→3)
   - D: SST3 (D→4)
3. Write merged SSTable to Level 1

After compaction:
Level 0: empty
Level 1: [SST4: A→10, B→20, C→3, D→4]
```

---

## 5️⃣ How Engineers Use This in Production

### Apache Cassandra

Cassandra uses LSM Trees for its storage engine:

```
┌─────────────────────────────────────────────────────────────────┐
│                    CASSANDRA STORAGE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Write Path:                                                    │
│  1. Write to CommitLog (WAL)                                   │
│  2. Write to MemTable                                          │
│  3. When MemTable full → flush to SSTable                      │
│                                                                  │
│  Compaction Strategies:                                         │
│  - SizeTieredCompactionStrategy (default)                      │
│  - LeveledCompactionStrategy                                   │
│  - TimeWindowCompactionStrategy (time-series)                  │
│                                                                  │
│  Configuration:                                                 │
│  CREATE TABLE users (                                          │
│    id UUID PRIMARY KEY,                                        │
│    name TEXT                                                   │
│  ) WITH compaction = {                                         │
│    'class': 'LeveledCompactionStrategy'                        │
│  };                                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### RocksDB (Facebook)

RocksDB is an embedded LSM-based key-value store:

```cpp
// RocksDB usage
#include "rocksdb/db.h"

rocksdb::DB* db;
rocksdb::Options options;
options.create_if_missing = true;

// Compaction settings
options.compaction_style = rocksdb::kCompactionStyleLevel;
options.level0_file_num_compaction_trigger = 4;
options.max_bytes_for_level_base = 256 * 1024 * 1024;  // 256 MB

rocksdb::DB::Open(options, "/tmp/testdb", &db);

// Write
db->Put(rocksdb::WriteOptions(), "key1", "value1");

// Read
std::string value;
db->Get(rocksdb::ReadOptions(), "key1", &value);
```

### LevelDB (Google)

LevelDB is the original LSM implementation that inspired RocksDB:

```
LevelDB Structure:
├── CURRENT          # Points to current MANIFEST
├── MANIFEST-000001  # Database metadata
├── LOG              # Write-ahead log
├── 000003.log       # Current WAL
├── 000004.ldb       # SSTable files
├── 000005.ldb
└── 000006.ldb
```

### InfluxDB (Time-Series)

InfluxDB uses a variant called TSM (Time-Structured Merge Tree):

```
TSM optimizations for time-series:
1. Data naturally sorted by time
2. Compression optimized for timestamps
3. Time-based compaction windows
4. Automatic data expiration (retention policies)
```

---

## 6️⃣ Implementation in Java

### MemTable Implementation

```java
import java.util.concurrent.ConcurrentSkipListMap;
import java.util.concurrent.atomic.AtomicLong;

/**
 * In-memory sorted structure for LSM Tree.
 * Uses ConcurrentSkipListMap for thread-safe sorted access.
 */
public class MemTable {
    
    private final ConcurrentSkipListMap<String, byte[]> data;
    private final AtomicLong size;
    private final long maxSize;
    
    public MemTable(long maxSize) {
        this.data = new ConcurrentSkipListMap<>();
        this.size = new AtomicLong(0);
        this.maxSize = maxSize;
    }
    
    /**
     * Puts a key-value pair into the MemTable.
     * @return true if MemTable is now full and should be flushed
     */
    public boolean put(String key, byte[] value) {
        byte[] oldValue = data.put(key, value);
        
        // Update size estimate
        long delta = key.length() + value.length;
        if (oldValue != null) {
            delta -= (key.length() + oldValue.length);
        }
        size.addAndGet(delta);
        
        return size.get() >= maxSize;
    }
    
    /**
     * Gets a value by key.
     */
    public byte[] get(String key) {
        return data.get(key);
    }
    
    /**
     * Deletes a key (tombstone marker).
     */
    public boolean delete(String key) {
        return put(key, null);  // null = tombstone
    }
    
    /**
     * Returns an iterator over all entries in sorted order.
     */
    public Iterable<java.util.Map.Entry<String, byte[]>> entries() {
        return data.entrySet();
    }
    
    public long getSize() {
        return size.get();
    }
    
    public boolean isEmpty() {
        return data.isEmpty();
    }
}
```

### SSTable Implementation

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;

/**
 * Sorted String Table - immutable on-disk sorted key-value file.
 */
public class SSTable {
    
    private final Path filePath;
    private final List<IndexEntry> index;
    private final BloomFilter bloomFilter;
    
    private static class IndexEntry {
        String firstKey;
        long offset;
        int length;
        
        IndexEntry(String firstKey, long offset, int length) {
            this.firstKey = firstKey;
            this.offset = offset;
            this.length = length;
        }
    }
    
    /**
     * Creates an SSTable from a MemTable.
     */
    public static SSTable create(Path filePath, MemTable memTable) throws IOException {
        List<IndexEntry> index = new ArrayList<>();
        BloomFilter bloom = new BloomFilter(10000, 0.01);
        
        try (DataOutputStream out = new DataOutputStream(
                new BufferedOutputStream(Files.newOutputStream(filePath)))) {
            
            long offset = 0;
            int blockSize = 0;
            String blockFirstKey = null;
            long blockOffset = 0;
            
            for (Map.Entry<String, byte[]> entry : memTable.entries()) {
                String key = entry.getKey();
                byte[] value = entry.getValue();
                
                // Add to bloom filter
                bloom.add(key);
                
                // Start new block if needed
                if (blockFirstKey == null) {
                    blockFirstKey = key;
                    blockOffset = offset;
                }
                
                // Write entry
                out.writeUTF(key);
                if (value == null) {
                    out.writeInt(-1);  // Tombstone
                } else {
                    out.writeInt(value.length);
                    out.write(value);
                    offset += 4 + value.length;
                }
                offset += 2 + key.length();  // UTF length prefix + string
                blockSize++;
                
                // Block full, create index entry
                if (blockSize >= 100) {
                    index.add(new IndexEntry(blockFirstKey, blockOffset, 
                                            (int)(offset - blockOffset)));
                    blockFirstKey = null;
                    blockSize = 0;
                }
            }
            
            // Final block
            if (blockFirstKey != null) {
                index.add(new IndexEntry(blockFirstKey, blockOffset, 
                                        (int)(offset - blockOffset)));
            }
        }
        
        return new SSTable(filePath, index, bloom);
    }
    
    private SSTable(Path filePath, List<IndexEntry> index, BloomFilter bloomFilter) {
        this.filePath = filePath;
        this.index = index;
        this.bloomFilter = bloomFilter;
    }
    
    /**
     * Gets a value by key.
     */
    public byte[] get(String key) throws IOException {
        // Check bloom filter first
        if (!bloomFilter.mightContain(key)) {
            return null;
        }
        
        // Binary search index to find block
        int blockIdx = findBlock(key);
        if (blockIdx < 0) {
            return null;
        }
        
        // Search within block
        IndexEntry block = index.get(blockIdx);
        try (RandomAccessFile raf = new RandomAccessFile(filePath.toFile(), "r")) {
            raf.seek(block.offset);
            DataInputStream in = new DataInputStream(
                new BufferedInputStream(new FileInputStream(raf.getFD())));
            
            long bytesRead = 0;
            while (bytesRead < block.length) {
                String k = in.readUTF();
                int valueLen = in.readInt();
                
                if (k.equals(key)) {
                    if (valueLen == -1) {
                        return null;  // Tombstone
                    }
                    byte[] value = new byte[valueLen];
                    in.readFully(value);
                    return value;
                }
                
                if (k.compareTo(key) > 0) {
                    return null;  // Passed the key
                }
                
                if (valueLen > 0) {
                    in.skipBytes(valueLen);
                }
                
                bytesRead += 2 + k.length() + 4 + Math.max(0, valueLen);
            }
        }
        
        return null;
    }
    
    private int findBlock(String key) {
        int lo = 0, hi = index.size() - 1;
        int result = -1;
        
        while (lo <= hi) {
            int mid = (lo + hi) / 2;
            String firstKey = index.get(mid).firstKey;
            
            if (firstKey.compareTo(key) <= 0) {
                result = mid;
                lo = mid + 1;
            } else {
                hi = mid - 1;
            }
        }
        
        return result;
    }
    
    public Path getFilePath() {
        return filePath;
    }
}
```

### LSM Tree Implementation

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

/**
 * Log-Structured Merge Tree implementation.
 */
public class LSMTree {
    
    private final Path directory;
    private final long memTableMaxSize;
    
    private volatile MemTable activeMemTable;
    private volatile MemTable immutableMemTable;
    private final List<List<SSTable>> levels;
    
    private final ExecutorService compactionExecutor;
    private final AtomicInteger sstableCounter;
    
    public LSMTree(Path directory, long memTableMaxSize) throws IOException {
        this.directory = directory;
        this.memTableMaxSize = memTableMaxSize;
        this.activeMemTable = new MemTable(memTableMaxSize);
        this.levels = new CopyOnWriteArrayList<>();
        this.levels.add(new CopyOnWriteArrayList<>());  // Level 0
        this.compactionExecutor = Executors.newSingleThreadExecutor();
        this.sstableCounter = new AtomicInteger(0);
        
        Files.createDirectories(directory);
    }
    
    /**
     * Puts a key-value pair.
     */
    public synchronized void put(String key, byte[] value) throws IOException {
        boolean shouldFlush = activeMemTable.put(key, value);
        
        if (shouldFlush) {
            flush();
        }
    }
    
    /**
     * Gets a value by key.
     */
    public byte[] get(String key) throws IOException {
        // Check active MemTable
        byte[] value = activeMemTable.get(key);
        if (value != null) {
            return value;
        }
        
        // Check immutable MemTable
        MemTable immutable = immutableMemTable;
        if (immutable != null) {
            value = immutable.get(key);
            if (value != null) {
                return value;
            }
        }
        
        // Check SSTables level by level
        for (List<SSTable> level : levels) {
            // Search in reverse order (newest first)
            for (int i = level.size() - 1; i >= 0; i--) {
                value = level.get(i).get(key);
                if (value != null) {
                    return value;
                }
            }
        }
        
        return null;
    }
    
    /**
     * Deletes a key.
     */
    public void delete(String key) throws IOException {
        put(key, null);  // Tombstone
    }
    
    /**
     * Flushes the active MemTable to disk.
     */
    private void flush() throws IOException {
        // Make current MemTable immutable
        immutableMemTable = activeMemTable;
        activeMemTable = new MemTable(memTableMaxSize);
        
        // Flush immutable MemTable to SSTable
        Path sstPath = directory.resolve("sst_" + sstableCounter.incrementAndGet() + ".db");
        SSTable sst = SSTable.create(sstPath, immutableMemTable);
        
        // Add to Level 0
        levels.get(0).add(sst);
        
        // Clear immutable MemTable
        immutableMemTable = null;
        
        // Trigger compaction if needed
        if (levels.get(0).size() >= 4) {
            scheduleCompaction();
        }
    }
    
    /**
     * Schedules background compaction.
     */
    private void scheduleCompaction() {
        compactionExecutor.submit(() -> {
            try {
                compact();
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
    
    /**
     * Compacts Level 0 SSTables into Level 1.
     */
    private synchronized void compact() throws IOException {
        List<SSTable> level0 = levels.get(0);
        if (level0.size() < 4) {
            return;
        }
        
        // Ensure Level 1 exists
        while (levels.size() < 2) {
            levels.add(new CopyOnWriteArrayList<>());
        }
        
        // Merge all Level 0 SSTables
        // (Simplified: in production, would merge with overlapping Level 1 SSTables)
        MemTable merged = new MemTable(Long.MAX_VALUE);
        
        for (SSTable sst : level0) {
            // Read all entries from SSTable and merge
            // (Simplified: would use iterators in production)
        }
        
        // Create new SSTable in Level 1
        Path sstPath = directory.resolve("sst_" + sstableCounter.incrementAndGet() + ".db");
        SSTable newSst = SSTable.create(sstPath, merged);
        levels.get(1).add(newSst);
        
        // Remove old Level 0 SSTables
        for (SSTable sst : level0) {
            Files.deleteIfExists(sst.getFilePath());
        }
        level0.clear();
    }
    
    /**
     * Closes the LSM Tree.
     */
    public void close() throws IOException {
        compactionExecutor.shutdown();
        // Flush remaining MemTable
        if (!activeMemTable.isEmpty()) {
            flush();
        }
    }
}
```

### Simple Bloom Filter for SSTable

```java
import java.util.BitSet;

/**
 * Simple Bloom Filter for SSTable.
 */
public class BloomFilter {
    
    private final BitSet bits;
    private final int size;
    private final int numHashes;
    
    public BloomFilter(int expectedElements, double falsePositiveRate) {
        this.size = optimalSize(expectedElements, falsePositiveRate);
        this.numHashes = optimalHashes(size, expectedElements);
        this.bits = new BitSet(size);
    }
    
    public void add(String element) {
        for (int i = 0; i < numHashes; i++) {
            int hash = hash(element, i);
            bits.set(Math.abs(hash % size));
        }
    }
    
    public boolean mightContain(String element) {
        for (int i = 0; i < numHashes; i++) {
            int hash = hash(element, i);
            if (!bits.get(Math.abs(hash % size))) {
                return false;
            }
        }
        return true;
    }
    
    private int hash(String element, int seed) {
        int hash = seed;
        for (char c : element.toCharArray()) {
            hash = hash * 31 + c;
        }
        return hash;
    }
    
    private static int optimalSize(int n, double p) {
        return (int) Math.ceil(-n * Math.log(p) / (Math.log(2) * Math.log(2)));
    }
    
    private static int optimalHashes(int m, int n) {
        return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
    }
}
```

---

## 7️⃣ Amplification Factors

### Write Amplification

**Definition**: Ratio of bytes written to storage vs. bytes written by user.

```
User writes: 1 MB
With compaction: Data rewritten multiple times as it moves through levels

Size-Tiered: ~10-30x write amplification
Leveled: ~10x write amplification (but more predictable)
```

### Read Amplification

**Definition**: Number of disk reads required to find a key.

```
Worst case: Check every level
- MemTable: 1 memory access
- Level 0: Up to 4 SSTables (may overlap)
- Level 1+: 1 SSTable per level (no overlap)

With Bloom filters: Most "not found" cases skip disk reads
```

### Space Amplification

**Definition**: Ratio of disk space used vs. actual data size.

```
Size-Tiered: High space amplification (old versions kept until compaction)
Leveled: Lower space amplification (~10%)
```

### Tuning Trade-offs

```
┌─────────────────────────────────────────────────────────────────┐
│                    AMPLIFICATION TRADE-OFFS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Size-Tiered Compaction:                                        │
│  + Lower write amplification                                    │
│  + Good for write-heavy workloads                               │
│  - Higher space amplification                                   │
│  - Higher read amplification                                    │
│                                                                  │
│  Leveled Compaction:                                            │
│  + Lower space amplification                                    │
│  + Lower read amplification                                     │
│  + Predictable performance                                      │
│  - Higher write amplification                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Pitfalls

**1. Ignoring read amplification**

```java
// BAD: Assuming LSM is always better
// For read-heavy workloads, B+ Tree may be faster

// GOOD: Profile your workload
// Write-heavy (>80% writes): LSM Tree
// Read-heavy (>80% reads): B+ Tree
// Balanced: Depends on latency requirements
```

**2. Wrong compaction strategy**

```java
// BAD: Using Size-Tiered for read-heavy workload
// High read amplification, unpredictable latency

// BAD: Using Leveled for write-heavy workload
// High write amplification, SSD wear

// GOOD: Match strategy to workload
// Size-Tiered: Write-heavy, can tolerate space overhead
// Leveled: Read-heavy, need predictable performance
// Time-Window: Time-series data with TTL
```

**3. MemTable too small or too large**

```java
// BAD: MemTable too small (1MB)
// Frequent flushes, many small SSTables, high compaction load

// BAD: MemTable too large (1GB)
// Long recovery time after crash, high memory usage

// GOOD: Balance based on write rate and recovery time
// Typically 64MB - 256MB
options.setWriteBufferSize(128 * 1024 * 1024);  // 128MB
```

**4. Not tuning Bloom filters**

```java
// BAD: Default Bloom filter settings
// May have too high false positive rate

// GOOD: Tune based on workload
// More bits = lower false positive = more memory
options.setBloomFilterBitsPerKey(10);  // ~1% false positive
```

### Performance Gotchas

**1. Compaction storms**

```java
// Problem: Many SSTables trigger compaction simultaneously
// Solution: Rate limit compaction, spread over time
options.setMaxBackgroundCompactions(4);
options.setRateLimiter(new RateLimiter(100 * 1024 * 1024));  // 100MB/s
```

**2. Write stalls**

```java
// Problem: Too many Level 0 files, writes blocked
// Solution: Tune L0 thresholds
options.setLevel0SlowdownWritesTrigger(20);
options.setLevel0StopWritesTrigger(36);
```

**3. Space amplification spikes**

```java
// Problem: During compaction, both old and new files exist
// Solution: Ensure enough disk space (2-3x data size)
```

---

## 9️⃣ When NOT to Use LSM Trees

### Anti-Patterns

**1. Read-heavy workloads**

```java
// WRONG: LSM for 95% reads, 5% writes
// Read amplification hurts performance

// RIGHT: Use B+ Tree (traditional database)
// Or use aggressive compaction with SSD
```

**2. Small datasets that fit in memory**

```java
// WRONG: LSM for 100MB dataset
// Overhead not worth it

// RIGHT: Use in-memory data structure
// HashMap, TreeMap, etc.
```

**3. When you need immediate consistency**

```java
// WRONG: LSM when every read must see latest write
// Compaction delay can cause stale reads from older SSTables

// RIGHT: Use B+ Tree or ensure read-your-writes consistency
```

**4. Random read patterns**

```java
// WRONG: LSM for random point queries
// Must check multiple levels

// RIGHT: B+ Tree with good caching
// Or add more Bloom filter bits
```

### Better Alternatives

| Use Case | Better Alternative |
|----------|-------------------|
| Read-heavy | B+ Tree (PostgreSQL, MySQL) |
| Small data | In-memory structures |
| Random reads | B+ Tree with cache |
| OLAP workloads | Column stores |
| Strong consistency | Traditional RDBMS |

---

## 🔟 Interview Follow-Up Questions with Answers

### L4 (Entry-Level) Questions

**Q1: What is an LSM Tree and why is it used?**

**Answer**: An LSM Tree (Log-Structured Merge Tree) is a data structure optimized for write-heavy workloads. It buffers writes in memory (MemTable), then flushes them to sorted files on disk (SSTables) when the buffer is full. Background compaction merges and sorts these files. It's used because it converts random writes to sequential writes, which are much faster on both HDDs and SSDs. Systems like Cassandra, RocksDB, and LevelDB use LSM Trees.

**Q2: How does an LSM Tree handle reads?**

**Answer**: Reads check multiple places in order:
1. Active MemTable (newest data, in memory)
2. Immutable MemTable (if being flushed)
3. Level 0 SSTables (recent flushes, may overlap)
4. Level 1, 2, ... SSTables (older data, no overlap within level)

Bloom filters help skip SSTables that definitely don't contain the key. Once found, return immediately (newer data shadows older).

### L5 (Senior) Questions

**Q3: Compare Size-Tiered and Leveled compaction strategies.**

**Answer**:

| Aspect | Size-Tiered | Leveled |
|--------|-------------|---------|
| Trigger | Similar-sized SSTables | Level size limit |
| Write amp | Lower (~10x) | Higher (~10-30x) |
| Read amp | Higher | Lower |
| Space amp | Higher (~2x) | Lower (~10%) |
| Predictability | Less predictable | More predictable |
| Best for | Write-heavy | Read-heavy, SSD |

Size-Tiered: Merge SSTables of similar size. Simple, good write throughput, but unpredictable compaction and space usage.

Leveled: Each level has a size limit. SSTables within a level don't overlap. More writes during compaction, but better read performance and space efficiency.

**Q4: How do Bloom filters help LSM Tree read performance?**

**Answer**: Bloom filters are probabilistic data structures that can tell if a key is definitely NOT in an SSTable. Each SSTable has its own Bloom filter. When reading:

1. Before searching an SSTable, check its Bloom filter
2. If Bloom filter says "no" → skip this SSTable entirely (no disk I/O)
3. If Bloom filter says "maybe" → search the SSTable

With 1% false positive rate, 99% of "not found" cases skip disk reads. This dramatically reduces read amplification, especially for keys that don't exist.

### L6 (Staff) Questions

**Q5: Design an LSM-based storage system for a time-series database.**

**Answer**:

```
Architecture:

1. Data Model:
   - Key: (metric_id, timestamp)
   - Value: measurement value
   - Natural ordering by time

2. Write Path:
   - Buffer writes in MemTable
   - Partition by time window (e.g., 1 hour)
   - Each partition has its own LSM tree

3. Compaction Strategy:
   - Time-Window Compaction:
     - Compact within time windows
     - Never merge across windows
     - Old windows become read-only
   - Benefits:
     - Queries hit fewer SSTables
     - Old data easily expired (delete whole files)

4. Optimizations:
   - Column-oriented storage within SSTable
   - Delta encoding for timestamps
   - Compression (timestamps compress well)
   - Bloom filters per time range

5. Retention:
   - TTL per time window
   - Delete entire SSTable files when expired
   - No compaction needed for deletion

6. Query Optimization:
   - Time-range queries hit specific partitions
   - Skip older partitions with Bloom filters
   - Parallel scan of relevant SSTables
```

---

## 1️⃣1️⃣ One Clean Mental Summary

An LSM Tree optimizes for writes by buffering data in memory (MemTable), then flushing to immutable sorted files (SSTables) on disk. All writes are sequential, avoiding the random I/O penalty of B-Trees. Reads check the MemTable first, then SSTables from newest to oldest, using Bloom filters to skip files that don't contain the key. Background compaction merges SSTables to reduce read amplification and reclaim space. The trade-off is read performance for write performance, tunable via compaction strategy. LSM Trees power write-heavy systems like Cassandra, RocksDB, and LevelDB.

---

## Summary

LSM Trees are essential for:
- **Write-heavy workloads**: Time-series, logging, event sourcing
- **NoSQL databases**: Cassandra, HBase, RocksDB
- **Embedded storage**: LevelDB, RocksDB
- **SSD optimization**: Sequential writes extend SSD life

Key takeaways:
1. Writes go to MemTable, flush to SSTables
2. All disk writes are sequential
3. Reads check MemTable, then SSTables by level
4. Bloom filters reduce read amplification
5. Compaction strategies trade write vs read amplification
6. Size-Tiered for writes, Leveled for reads

