# ⚡ JVM Tuning & Performance

---

## 0️⃣ Prerequisites

Before diving into JVM Tuning, you need to understand:

- **Java Memory Model**: Heap, Stack, Metaspace (covered in `07-java-memory-model.md`)
- **Garbage Collection**: GC algorithms, generations (covered in `08-garbage-collection.md`)
- **Basic Linux Commands**: `top`, `ps`, file operations

---

## 1️⃣ JVM Memory Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    JVM MEMORY ARCHITECTURE                               │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                         JVM PROCESS                              │   │
│   │                                                                  │   │
│   │  ┌──────────────────────────────────────────────────────────┐   │   │
│   │  │                     HEAP (-Xmx, -Xms)                     │   │   │
│   │  │  ┌────────────────────────────────────────────────────┐  │   │   │
│   │  │  │           YOUNG GENERATION (-Xmn)                  │  │   │   │
│   │  │  │  ┌──────────┬───────────┬───────────┐             │  │   │   │
│   │  │  │  │  EDEN    │ Survivor0 │ Survivor1 │             │  │   │   │
│   │  │  │  └──────────┴───────────┴───────────┘             │  │   │   │
│   │  │  └────────────────────────────────────────────────────┘  │   │   │
│   │  │  ┌────────────────────────────────────────────────────┐  │   │   │
│   │  │  │              OLD GENERATION                        │  │   │   │
│   │  │  │         (Long-lived objects)                       │  │   │   │
│   │  │  └────────────────────────────────────────────────────┘  │   │   │
│   │  └──────────────────────────────────────────────────────────┘   │   │
│   │                                                                  │   │
│   │  ┌──────────────────────────────────────────────────────────┐   │   │
│   │  │                    METASPACE                              │   │   │
│   │  │  Class metadata, method bytecode, static variables       │   │   │
│   │  │  (-XX:MetaspaceSize, -XX:MaxMetaspaceSize)               │   │   │
│   │  └──────────────────────────────────────────────────────────┘   │   │
│   │                                                                  │   │
│   │  ┌──────────────────────────────────────────────────────────┐   │   │
│   │  │                  NATIVE MEMORY                            │   │   │
│   │  │  Thread stacks (-Xss), JNI, Direct buffers               │   │   │
│   │  └──────────────────────────────────────────────────────────┘   │   │
│   │                                                                  │   │
│   │  ┌──────────────────────────────────────────────────────────┐   │   │
│   │  │                  CODE CACHE                               │   │   │
│   │  │  JIT compiled code (-XX:ReservedCodeCacheSize)           │   │   │
│   │  └──────────────────────────────────────────────────────────┘   │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2️⃣ Essential JVM Flags

### Memory Settings

```bash
# Heap size
-Xms4g              # Initial heap size (4GB)
-Xmx4g              # Maximum heap size (4GB)
                    # Best practice: Set -Xms = -Xmx to avoid resizing

# Young generation
-Xmn1g              # Young generation size
-XX:NewRatio=2      # Old:Young ratio (Old = 2x Young)
-XX:SurvivorRatio=8 # Eden:Survivor ratio (Eden = 8x each Survivor)

# Metaspace
-XX:MetaspaceSize=256m      # Initial metaspace size
-XX:MaxMetaspaceSize=512m   # Maximum metaspace size

# Thread stack
-Xss512k            # Thread stack size (default ~1MB)

# Direct memory
-XX:MaxDirectMemorySize=256m  # For NIO direct buffers
```

### GC Selection

```bash
# Serial GC (single-threaded, small heaps)
-XX:+UseSerialGC

# Parallel GC (throughput-focused, Java 8 default)
-XX:+UseParallelGC

# G1 GC (balanced, Java 9+ default)
-XX:+UseG1GC

# ZGC (low latency, Java 15+)
-XX:+UseZGC

# Shenandoah (low latency, OpenJDK)
-XX:+UseShenandoahGC
```

### G1 GC Tuning

```bash
# Basic G1 settings
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200        # Target max pause time (default 200ms)
-XX:G1HeapRegionSize=16m        # Region size (1-32MB, auto-calculated)

# Concurrent marking
-XX:InitiatingHeapOccupancyPercent=45  # Start marking when heap 45% full
-XX:G1ReservePercent=10                 # Reserve for promotion failures

# Reference processing
-XX:+ParallelRefProcEnabled     # Parallel reference processing

# String deduplication
-XX:+UseStringDeduplication     # Deduplicate String objects
```

### ZGC Tuning

```bash
# Basic ZGC settings (minimal tuning needed)
-XX:+UseZGC
-Xmx16g                         # Set appropriate heap size

# Soft heap limit (Java 13+)
-XX:SoftMaxHeapSize=8g          # GC tries to keep heap under this

# Concurrent threads
-XX:ConcGCThreads=4             # Number of concurrent GC threads
```

### GC Logging

```bash
# Java 9+ unified logging
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=10m

# Detailed GC logging
-Xlog:gc*=debug:file=gc-debug.log:time,uptime:filecount=3,filesize=20m

# GC pause times only
-Xlog:gc:file=gc.log:time

# Safepoint logging (when threads stop)
-Xlog:safepoint:file=safepoint.log:time
```

### Performance Flags

```bash
# JIT compilation
-XX:+TieredCompilation          # Enable tiered compilation (default)
-XX:TieredStopAtLevel=4         # Full optimization (default)
-XX:CompileThreshold=10000      # Invocations before compilation

# Code cache
-XX:ReservedCodeCacheSize=256m  # Size for JIT compiled code
-XX:InitialCodeCacheSize=64m    # Initial code cache size

# Inlining
-XX:MaxInlineSize=35            # Max bytecode size for inlining
-XX:FreqInlineSize=325          # Max size for hot methods

# Escape analysis
-XX:+DoEscapeAnalysis           # Enable escape analysis (default)

# Compressed pointers (64-bit JVM, heap < 32GB)
-XX:+UseCompressedOops          # Compress object pointers (default)
-XX:+UseCompressedClassPointers # Compress class pointers (default)
```

---

## 3️⃣ JIT Compilation

### How JIT Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    JIT COMPILATION TIERS                                 │
│                                                                          │
│   BYTECODE                                                              │
│       │                                                                  │
│       ▼                                                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     INTERPRETER                                  │   │
│   │   Executes bytecode directly (slow, but no compilation delay)   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│       │                                                                  │
│       │ Method called frequently (profiling)                            │
│       ▼                                                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                  C1 COMPILER (Client)                            │   │
│   │   Quick compilation, basic optimizations                        │   │
│   │   - Inlining                                                    │   │
│   │   - Simple dead code elimination                                │   │
│   │   - Basic loop optimizations                                    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│       │                                                                  │
│       │ Method still hot (more profiling data)                         │
│       ▼                                                                  │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                  C2 COMPILER (Server)                            │   │
│   │   Aggressive optimizations (takes longer to compile)            │   │
│   │   - Advanced inlining                                           │   │
│   │   - Escape analysis                                             │   │
│   │   - Loop unrolling                                              │   │
│   │   - Vectorization (SIMD)                                        │   │
│   │   - Dead code elimination                                       │   │
│   │   - Speculative optimizations                                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   TIERED COMPILATION (Default):                                         │
│   Level 0: Interpreter                                                  │
│   Level 1: C1 with full optimization (no profiling)                    │
│   Level 2: C1 with invocation counters                                 │
│   Level 3: C1 with full profiling                                      │
│   Level 4: C2 with full optimization                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key JIT Optimizations

```java
// 1. INLINING - Replace method call with method body
// Before inlining:
public int calculate(int x) {
    return square(x) + 1;
}
private int square(int x) {
    return x * x;
}

// After inlining:
public int calculate(int x) {
    return x * x + 1;  // No method call overhead
}

// 2. ESCAPE ANALYSIS - Allocate on stack if object doesn't escape
public int sum() {
    Point p = new Point(1, 2);  // Object doesn't escape method
    return p.x + p.y;
}
// JIT can allocate Point on stack or even eliminate it entirely

// 3. LOOP UNROLLING
// Before:
for (int i = 0; i < 4; i++) {
    sum += array[i];
}
// After:
sum += array[0];
sum += array[1];
sum += array[2];
sum += array[3];

// 4. DEAD CODE ELIMINATION
public int example(boolean flag) {
    int x = 10;
    if (flag) {
        return x;
    }
    int y = 20;  // Dead code if flag is always true
    return y;
}
```

---

## 4️⃣ Profiling Tools

### JFR (Java Flight Recorder)

```bash
# Start recording with JVM flags
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

# Start recording on running JVM
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Dump recording
jcmd <pid> JFR.dump filename=dump.jfr

# Stop recording
jcmd <pid> JFR.stop

# Analyze with JDK Mission Control (jmc)
jmc recording.jfr
```

### jcmd (JVM Diagnostic Command)

```bash
# List running Java processes
jcmd

# Get VM info
jcmd <pid> VM.info

# Get system properties
jcmd <pid> VM.system_properties

# Get VM flags
jcmd <pid> VM.flags

# Trigger GC
jcmd <pid> GC.run

# Heap dump
jcmd <pid> GC.heap_dump filename.hprof

# Thread dump
jcmd <pid> Thread.print

# Class histogram
jcmd <pid> GC.class_histogram
```

### jstat (JVM Statistics)

```bash
# GC statistics every 1 second
jstat -gc <pid> 1000

# GC cause
jstat -gccause <pid> 1000

# Class loading statistics
jstat -class <pid> 1000

# Compilation statistics
jstat -compiler <pid> 1000

# Output columns for -gc:
# S0C, S1C: Survivor space capacity
# S0U, S1U: Survivor space used
# EC, EU: Eden capacity/used
# OC, OU: Old capacity/used
# MC, MU: Metaspace capacity/used
# YGC, YGCT: Young GC count/time
# FGC, FGCT: Full GC count/time
# GCT: Total GC time
```

### async-profiler

```bash
# CPU profiling
./profiler.sh -d 30 -f cpu.html <pid>

# Allocation profiling
./profiler.sh -d 30 -e alloc -f alloc.html <pid>

# Wall-clock profiling (includes waiting time)
./profiler.sh -d 30 -e wall -f wall.html <pid>

# Lock contention profiling
./profiler.sh -d 30 -e lock -f lock.html <pid>

# Generate flame graph
./profiler.sh -d 30 -f flamegraph.svg <pid>
```

### VisualVM

```bash
# Start VisualVM
jvisualvm

# Features:
# - Monitor heap, CPU, threads in real-time
# - Take heap dumps and analyze
# - Profile CPU and memory
# - View thread dumps
# - Monitor GC activity
```

---

## 5️⃣ Flame Graphs

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    READING FLAME GRAPHS                                  │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                     main()                                       │   │
│   │  ┌────────────────────────────┬─────────────────────────────┐   │   │
│   │  │      processOrders()       │      generateReport()        │   │   │
│   │  │  ┌───────────┬──────────┐  │  ┌──────────────────────┐   │   │   │
│   │  │  │validateOrd│ saveOrder│  │  │   queryDatabase()    │   │   │   │
│   │  │  │  ┌──────┐ │  ┌─────┐ │  │  │  ┌────────────────┐  │   │   │   │
│   │  │  │  │check │ │  │write│ │  │  │  │  executeSQL()  │  │   │   │   │
│   │  │  │  └──────┘ │  └─────┘ │  │  │  └────────────────┘  │   │   │   │
│   │  │  └───────────┴──────────┘  │  └──────────────────────┘   │   │   │
│   │  └────────────────────────────┴─────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   HOW TO READ:                                                          │
│   - X-axis: Percentage of samples (width = time spent)                 │
│   - Y-axis: Stack depth (bottom = entry point, top = leaf function)    │
│   - Wide boxes = Hot spots (where time is spent)                       │
│   - Narrow boxes = Quick functions                                      │
│                                                                          │
│   WHAT TO LOOK FOR:                                                     │
│   - Wide plateaus at top = Time spent in that specific function        │
│   - Wide boxes anywhere = Significant time consumers                   │
│   - Deep stacks = Many nested calls                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6️⃣ Common Performance Issues

### Memory Issues

```java
// 1. MEMORY LEAK - Objects retained but not used
public class EventBus {
    private static List<EventListener> listeners = new ArrayList<>();
    
    public void addListener(EventListener listener) {
        listeners.add(listener);  // Never removed!
    }
}

// Fix: Remove listeners when done
public void removeListener(EventListener listener) {
    listeners.remove(listener);
}

// 2. LARGE OBJECT ALLOCATION - Pressure on GC
public void processFile(String path) {
    byte[] content = Files.readAllBytes(path);  // 1GB file = 1GB allocation
}

// Fix: Stream processing
public void processFile(String path) {
    try (BufferedReader reader = Files.newBufferedReader(path)) {
        reader.lines().forEach(this::processLine);
    }
}

// 3. STRING CONCATENATION IN LOOPS
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i;  // Creates new String each iteration!
}

// Fix: Use StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i);
}
String result = sb.toString();

// 4. AUTOBOXING IN HOT PATHS
public long sum(List<Long> numbers) {
    long sum = 0;
    for (Long n : numbers) {
        sum += n;  // Unboxing on every iteration
    }
    return sum;
}

// Fix: Use primitive streams
public long sum(List<Long> numbers) {
    return numbers.stream().mapToLong(Long::longValue).sum();
}
```

### CPU Issues

```java
// 1. INEFFICIENT ALGORITHMS
// O(n²) when O(n) is possible
public boolean hasDuplicates(List<String> items) {
    for (int i = 0; i < items.size(); i++) {
        for (int j = i + 1; j < items.size(); j++) {
            if (items.get(i).equals(items.get(j))) {
                return true;
            }
        }
    }
    return false;
}

// Fix: Use Set - O(n)
public boolean hasDuplicates(List<String> items) {
    return items.size() != new HashSet<>(items).size();
}

// 2. REGEX COMPILATION IN LOOPS
public List<String> filterEmails(List<String> inputs) {
    return inputs.stream()
        .filter(s -> s.matches(".*@.*\\..*"))  // Compiles regex each time!
        .toList();
}

// Fix: Compile once
private static final Pattern EMAIL_PATTERN = Pattern.compile(".*@.*\\..*");

public List<String> filterEmails(List<String> inputs) {
    return inputs.stream()
        .filter(s -> EMAIL_PATTERN.matcher(s).matches())
        .toList();
}

// 3. EXCESSIVE LOGGING
public void processItem(Item item) {
    log.debug("Processing item: " + item.toDetailedString());  // Always builds string!
}

// Fix: Use lazy evaluation
public void processItem(Item item) {
    if (log.isDebugEnabled()) {
        log.debug("Processing item: {}", item.toDetailedString());
    }
    // Or with SLF4J 2.0+
    log.atDebug().log(() -> "Processing item: " + item.toDetailedString());
}
```

### Lock Contention

```java
// 1. OVER-SYNCHRONIZED
public class Counter {
    private int count = 0;
    
    public synchronized void increment() {  // Blocks all threads
        count++;
    }
    
    public synchronized int getCount() {  // Even reads block!
        return count;
    }
}

// Fix: Use atomic types
public class Counter {
    private final AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();  // Lock-free
    }
    
    public int getCount() {
        return count.get();  // No blocking
    }
}

// 2. LOCK HELD DURING I/O
public synchronized void processAndSave(Data data) {
    // Lock held during entire operation!
    Data processed = process(data);
    database.save(processed);  // I/O while holding lock
}

// Fix: Minimize lock scope
public void processAndSave(Data data) {
    Data processed;
    synchronized(this) {
        processed = process(data);  // Only lock for processing
    }
    database.save(processed);  // I/O outside lock
}
```

---

## 7️⃣ Production JVM Configuration Examples

### Web Application (Spring Boot)

```bash
java \
  -Xms4g -Xmx4g \                          # Fixed heap size
  -XX:+UseG1GC \                           # G1 GC
  -XX:MaxGCPauseMillis=200 \               # Target pause time
  -XX:+UseStringDeduplication \            # Deduplicate strings
  -XX:+ParallelRefProcEnabled \            # Parallel reference processing
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=20m \
  -XX:+HeapDumpOnOutOfMemoryError \        # Dump heap on OOM
  -XX:HeapDumpPath=/var/log/app/ \         # Heap dump location
  -XX:+ExitOnOutOfMemoryError \            # Exit on OOM (let orchestrator restart)
  -jar app.jar
```

### Low-Latency Application

```bash
java \
  -Xms8g -Xmx8g \                          # Fixed heap
  -XX:+UseZGC \                            # ZGC for low latency
  -XX:SoftMaxHeapSize=6g \                 # Soft limit
  -XX:+AlwaysPreTouch \                    # Pre-touch heap pages
  -XX:+UseNUMA \                           # NUMA-aware allocation
  -XX:+DisableExplicitGC \                 # Ignore System.gc()
  -Xlog:gc*:file=/var/log/app/gc.log:time \
  -jar app.jar
```

### Batch Processing (Throughput)

```bash
java \
  -Xms16g -Xmx16g \                        # Large heap
  -XX:+UseParallelGC \                     # Parallel GC for throughput
  -XX:ParallelGCThreads=8 \                # GC threads
  -XX:+UseAdaptiveSizePolicy \             # Adaptive sizing
  -XX:MaxGCPauseMillis=1000 \              # Longer pauses OK
  -Xlog:gc*:file=/var/log/app/gc.log:time \
  -jar batch-job.jar
```

### Container (Kubernetes/Docker)

```bash
java \
  -XX:+UseContainerSupport \               # Respect container limits (default)
  -XX:MaxRAMPercentage=75.0 \              # Use 75% of container memory
  -XX:InitialRAMPercentage=75.0 \          # Start at 75%
  -XX:+UseG1GC \                           # G1 GC
  -XX:MaxGCPauseMillis=200 \               # Target pause
  -Xlog:gc*:file=/var/log/gc.log:time \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/ \
  -jar app.jar
```

---

## 8️⃣ Troubleshooting Checklist

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PERFORMANCE TROUBLESHOOTING                           │
│                                                                          │
│   HIGH CPU USAGE                                                        │
│   ─────────────                                                         │
│   □ Check thread dump for busy threads                                  │
│   □ Profile with async-profiler (CPU mode)                              │
│   □ Look for infinite loops, inefficient algorithms                     │
│   □ Check for excessive GC (jstat -gc)                                  │
│                                                                          │
│   HIGH MEMORY USAGE                                                     │
│   ─────────────────                                                     │
│   □ Take heap dump (jcmd <pid> GC.heap_dump)                           │
│   □ Analyze with Eclipse MAT or VisualVM                               │
│   □ Look for memory leaks (retained objects)                           │
│   □ Check GC logs for increasing heap after Full GC                    │
│                                                                          │
│   LONG GC PAUSES                                                        │
│   ──────────────                                                        │
│   □ Check GC logs for pause times                                       │
│   □ Consider switching GC (G1 → ZGC for lower latency)                 │
│   □ Tune GC parameters (-XX:MaxGCPauseMillis)                          │
│   □ Reduce allocation rate (profile allocations)                       │
│                                                                          │
│   SLOW RESPONSE TIMES                                                   │
│   ───────────────────                                                   │
│   □ Profile with JFR or async-profiler                                 │
│   □ Check for lock contention (async-profiler lock mode)               │
│   □ Review database queries (slow query log)                           │
│   □ Check external service latencies                                   │
│                                                                          │
│   OUTOFMEMORYERROR                                                      │
│   ────────────────                                                      │
│   □ Heap space: Increase -Xmx or fix memory leak                       │
│   □ Metaspace: Increase -XX:MaxMetaspaceSize                           │
│   □ Direct buffer: Increase -XX:MaxDirectMemorySize                    │
│   □ Unable to create thread: Reduce -Xss or increase ulimit           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between -Xms and -Xmx?**

A: `-Xms` sets the initial heap size, `-Xmx` sets the maximum heap size.

Example: `-Xms2g -Xmx4g` starts with 2GB heap, can grow to 4GB.

Best practice: Set them equal (`-Xms4g -Xmx4g`) to avoid the overhead of heap resizing during runtime.

**Q: What is JIT compilation?**

A: JIT (Just-In-Time) compilation converts frequently executed bytecode into native machine code at runtime for better performance.

The JVM starts by interpreting bytecode, then identifies "hot" methods (called frequently) and compiles them. Java uses tiered compilation: C1 compiler for quick compilation with basic optimizations, C2 compiler for aggressive optimizations on very hot code.

### L5 (Mid-Level) Questions

**Q: How would you diagnose a memory leak?**

A: Step-by-step approach:

1. **Confirm the leak**: Monitor heap usage after Full GC. If it keeps increasing, there's likely a leak.

2. **Take heap dumps**: `jcmd <pid> GC.heap_dump heap.hprof`

3. **Analyze with tools**: Use Eclipse MAT or VisualVM
   - Look at dominator tree (what's retaining memory)
   - Find objects with unexpected retention
   - Check GC roots keeping objects alive

4. **Common culprits**:
   - Static collections that grow
   - Listeners not removed
   - ThreadLocal not cleaned up
   - Caches without eviction

**Q: Explain the difference between G1 and ZGC.**

A: Both are low-pause collectors, but with different approaches:

**G1**:
- Divides heap into regions
- Collects regions with most garbage first
- Pause times: 10-200ms typical
- Good for: General purpose, heaps 4GB-64GB

**ZGC**:
- Uses colored pointers and load barriers
- Almost all work is concurrent
- Pause times: <1ms regardless of heap size
- Good for: Low-latency requirements, very large heaps

Choose G1 for general purpose, ZGC when you need sub-millisecond pauses.

### L6 (Senior) Questions

**Q: How would you tune JVM for a latency-sensitive trading application?**

A: Key considerations:

1. **GC Selection**: ZGC or Shenandoah for sub-ms pauses

2. **Memory**:
   - Fixed heap size (`-Xms = -Xmx`)
   - `-XX:+AlwaysPreTouch` to pre-fault pages
   - Size heap to minimize GC frequency

3. **Reduce allocations**:
   - Object pooling
   - Primitive arrays over object arrays
   - Avoid autoboxing

4. **JIT warmup**:
   - Warm up before accepting traffic
   - Consider `-XX:+TieredCompilation`

5. **OS tuning**:
   - Pin JVM to specific CPUs
   - Use huge pages (`-XX:+UseLargePages`)
   - Disable NUMA balancing if single-socket

6. **Monitoring**:
   - JFR for continuous profiling
   - Alert on GC pause times
   - Monitor allocation rate

---

## 🔟 One Clean Mental Summary

JVM tuning is about balancing **throughput** (total work done), **latency** (pause times), and **footprint** (memory usage). Start with defaults and measure before tuning. Key flags: `-Xms`/`-Xmx` for heap size (set equal), `-XX:+UseG1GC` or `-XX:+UseZGC` for GC selection. Always enable GC logging (`-Xlog:gc*`). Use profiling tools: **JFR** for production profiling, **async-profiler** for CPU/allocation analysis, **jstat** for GC stats, **jcmd** for diagnostics. Flame graphs show where time is spent (wide = hot). Common issues: memory leaks (heap grows after GC), lock contention (threads waiting), inefficient algorithms (CPU spikes). For containers, use `-XX:MaxRAMPercentage` instead of fixed heap sizes. The goal isn't to use every tuning flag - it's to identify bottlenecks, make targeted changes, and measure the impact.

