# 🧶 Project Loom (Java 21+)

---

## 0️⃣ Prerequisites

Before diving into Project Loom, you need to understand:

- **Java Concurrency**: Threads, ExecutorService, thread pools (covered in `06-java-concurrency.md`)
- **Blocking I/O**: How threads wait for network, disk, database operations
- **Thread Costs**: Memory overhead (~1MB per thread), context switching

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Thread-Per-Request Problem

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THE SCALABILITY PROBLEM                               │
│                                                                          │
│   TRADITIONAL THREAD-PER-REQUEST MODEL:                                 │
│   ─────────────────────────────────────                                 │
│                                                                          │
│   Request 1 ──► Thread 1 ──► [DB Query 100ms] ──► Response             │
│   Request 2 ──► Thread 2 ──► [DB Query 100ms] ──► Response             │
│   Request 3 ──► Thread 3 ──► [DB Query 100ms] ──► Response             │
│   ...                                                                    │
│   Request 1000 ──► Thread 1000 ──► [DB Query 100ms] ──► Response       │
│                                                                          │
│   PROBLEM:                                                              │
│   - Each thread uses ~1MB of stack memory                               │
│   - 1000 threads = 1GB just for stacks!                                │
│   - Most threads are BLOCKED waiting for I/O                           │
│   - OS scheduler overhead with many threads                            │
│   - Can't scale beyond ~10,000 concurrent requests                     │
│                                                                          │
│   Thread 1: [Working 1ms][Waiting 99ms][Working 1ms][Waiting 99ms]...  │
│             ↑ 99% of time doing nothing!                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Solutions Before Loom

```java
// Solution 1: Thread Pools (limits threads, but limits concurrency)
ExecutorService executor = Executors.newFixedThreadPool(200);
// Can only handle 200 concurrent requests!

// Solution 2: Async/Reactive (complex, different programming model)
Mono.fromCallable(() -> fetchUser(id))
    .flatMap(user -> fetchOrders(user))
    .flatMap(orders -> processOrders(orders))
    .subscribe(result -> sendResponse(result));
// Callback hell, hard to debug, stack traces useless

// Solution 3: Event loops (Node.js style - single thread)
// Limited by single thread for CPU work
```

### What Loom Provides

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VIRTUAL THREADS SOLUTION                              │
│                                                                          │
│   VIRTUAL THREADS (Project Loom):                                       │
│   ───────────────────────────────                                       │
│                                                                          │
│   Request 1 ──► VThread 1 ──► [DB Query] ──► Response                  │
│   Request 2 ──► VThread 2 ──► [DB Query] ──► Response                  │
│   ...                                                                    │
│   Request 1,000,000 ──► VThread 1M ──► [DB Query] ──► Response         │
│                                                                          │
│   HOW IT WORKS:                                                         │
│   - Virtual threads are CHEAP (~1KB vs ~1MB)                           │
│   - Millions of virtual threads possible                               │
│   - When virtual thread blocks, it's UNMOUNTED from carrier thread     │
│   - Carrier thread can run other virtual threads                       │
│   - Same simple blocking code, massive scalability                     │
│                                                                          │
│   Carrier Thread 1: [VT1][VT5][VT9][VT13]...                          │
│   Carrier Thread 2: [VT2][VT6][VT10][VT14]...                         │
│   Carrier Thread 3: [VT3][VT7][VT11][VT15]...                         │
│   Carrier Thread 4: [VT4][VT8][VT12][VT16]...                         │
│                                                                          │
│   Virtual threads are multiplexed onto few carrier threads!            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2️⃣ Virtual Threads

### Creating Virtual Threads

```java
// Method 1: Thread.startVirtualThread()
Thread vThread = Thread.startVirtualThread(() -> {
    System.out.println("Running in virtual thread: " + Thread.currentThread());
});

// Method 2: Thread.ofVirtual()
Thread vThread = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> {
        System.out.println("Running in: " + Thread.currentThread());
    });

// Method 3: Virtual thread executor (recommended for most cases)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // Each task gets its own virtual thread
        return fetchDataFromDatabase();
    });
}

// Method 4: Virtual thread factory
ThreadFactory factory = Thread.ofVirtual().name("worker-", 0).factory();
Thread vThread = factory.newThread(() -> doWork());
vThread.start();
```

### Virtual vs Platform Threads

```java
// Check if thread is virtual
Thread.currentThread().isVirtual();  // true or false

// Platform thread (traditional OS thread)
Thread platformThread = Thread.ofPlatform()
    .name("platform-thread")
    .start(() -> {
        System.out.println("Platform thread: " + Thread.currentThread());
        System.out.println("Is virtual: " + Thread.currentThread().isVirtual());  // false
    });

// Virtual thread
Thread virtualThread = Thread.ofVirtual()
    .name("virtual-thread")
    .start(() -> {
        System.out.println("Virtual thread: " + Thread.currentThread());
        System.out.println("Is virtual: " + Thread.currentThread().isVirtual());  // true
    });
```

### The Power of Virtual Threads

```java
// This would crash with platform threads (memory exhaustion)
// But works fine with virtual threads!

public class VirtualThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        
        Instant start = Instant.now();
        
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            // Submit 100,000 tasks!
            IntStream.range(0, 100_000).forEach(i -> {
                executor.submit(() -> {
                    Thread.sleep(Duration.ofSeconds(1));  // Simulate I/O
                    return i;
                });
            });
        }  // Waits for all tasks to complete
        
        Instant end = Instant.now();
        System.out.println("Duration: " + Duration.between(start, end).toSeconds() + "s");
        // With virtual threads: ~1 second (all run concurrently)
        // With 100 platform threads: ~1000 seconds (sequential batches)
    }
}
```

---

## 3️⃣ How Virtual Threads Work

### Carrier Threads and Mounting

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VIRTUAL THREAD MECHANICS                              │
│                                                                          │
│   CARRIER THREADS (Platform threads managed by JVM):                    │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Carrier 1   │  Carrier 2   │  Carrier 3   │  Carrier 4        │   │
│   │  (OS Thread) │  (OS Thread) │  (OS Thread) │  (OS Thread)      │   │
│   └──────┬───────┴──────┬───────┴──────┬───────┴──────┬────────────┘   │
│          │              │              │              │                 │
│   VIRTUAL THREADS (Lightweight, managed by JVM):                       │
│                                                                          │
│   Time T1:                                                              │
│   ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐               │
│   │ VT1  │       │ VT2  │       │ VT3  │       │ VT4  │               │
│   │MOUNTED       │MOUNTED       │MOUNTED       │MOUNTED               │
│   └──────┘       └──────┘       └──────┘       └──────┘               │
│                                                                          │
│   VT1 blocks on I/O...                                                  │
│                                                                          │
│   Time T2:                                                              │
│   ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐               │
│   │ VT5  │       │ VT2  │       │ VT3  │       │ VT4  │               │
│   │MOUNTED       │MOUNTED       │MOUNTED       │MOUNTED               │
│   └──────┘       └──────┘       └──────┘       └──────┘               │
│                                                                          │
│   ┌──────┐                                                              │
│   │ VT1  │ UNMOUNTED (waiting for I/O, not using carrier)              │
│   └──────┘                                                              │
│                                                                          │
│   VT1's I/O completes...                                                │
│                                                                          │
│   Time T3:                                                              │
│   ┌──────┐       ┌──────┐       ┌──────┐       ┌──────┐               │
│   │ VT1  │       │ VT2  │       │ VT6  │       │ VT4  │               │
│   │MOUNTED       │MOUNTED       │MOUNTED       │MOUNTED               │
│   └──────┘       └──────┘       └──────┘       └──────┘               │
│                                                                          │
│   VT1 is remounted and continues execution!                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Blocking Operations

```java
// These operations cause virtual thread to UNMOUNT (good!):
// - Thread.sleep()
// - Blocking I/O (sockets, files)
// - BlockingQueue operations
// - Lock.lock() (with some caveats)
// - Object.wait()

// Example: Virtual thread unmounts during sleep
Thread.ofVirtual().start(() -> {
    System.out.println("Before sleep on carrier: " + carrierThread());
    Thread.sleep(1000);  // Virtual thread unmounts!
    System.out.println("After sleep on carrier: " + carrierThread());
    // Might be different carrier thread!
});

private static String carrierThread() {
    return Thread.currentThread().toString();
}
```

### Pinning (When Virtual Threads Can't Unmount)

```java
// PINNING: Virtual thread is "pinned" to carrier thread and can't unmount

// Cause 1: synchronized blocks
synchronized (lock) {
    Thread.sleep(1000);  // Virtual thread is PINNED! Can't unmount.
}

// Cause 2: Native methods / JNI

// SOLUTION: Use ReentrantLock instead of synchronized
private final ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    Thread.sleep(1000);  // Virtual thread CAN unmount
} finally {
    lock.unlock();
}

// Detect pinning with JVM flag:
// -Djdk.tracePinnedThreads=full
// or
// -Djdk.tracePinnedThreads=short
```

---

## 4️⃣ Structured Concurrency (Preview in Java 21)

### The Problem with Unstructured Concurrency

```java
// Unstructured: Tasks have independent lifecycles
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

Future<User> userFuture = executor.submit(() -> fetchUser(userId));
Future<List<Order>> ordersFuture = executor.submit(() -> fetchOrders(userId));

// Problems:
// 1. If fetchUser fails, fetchOrders continues running (wasted work)
// 2. If we cancel, both tasks might not be cancelled
// 3. Exceptions can be lost
// 4. Hard to reason about lifecycle
```

### Structured Concurrency with StructuredTaskScope

```java
// Structured: Tasks are scoped to a block
// When scope ends, all tasks are complete (or cancelled)

import java.util.concurrent.StructuredTaskScope;

public record UserWithOrders(User user, List<Order> orders) { }

public UserWithOrders fetchUserWithOrders(String userId) throws Exception {
    
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        
        // Fork subtasks
        Subtask<User> userTask = scope.fork(() -> fetchUser(userId));
        Subtask<List<Order>> ordersTask = scope.fork(() -> fetchOrders(userId));
        
        // Wait for all tasks to complete (or first failure)
        scope.join();
        
        // Propagate any exception
        scope.throwIfFailed();
        
        // All tasks succeeded - get results
        return new UserWithOrders(userTask.get(), ordersTask.get());
    }
    // Scope ensures all tasks are complete before exiting
}
```

### ShutdownOnFailure vs ShutdownOnSuccess

```java
// ShutdownOnFailure: Cancel all if ANY task fails
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var task1 = scope.fork(() -> fetchFromServiceA());
    var task2 = scope.fork(() -> fetchFromServiceB());
    
    scope.join();
    scope.throwIfFailed();  // Throws if any failed
    
    return combine(task1.get(), task2.get());
}

// ShutdownOnSuccess: Cancel remaining when FIRST task succeeds
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    scope.fork(() -> fetchFromMirror1());
    scope.fork(() -> fetchFromMirror2());
    scope.fork(() -> fetchFromMirror3());
    
    scope.join();
    
    return scope.result();  // Returns first successful result
}
```

### Custom StructuredTaskScope

```java
// Custom scope that collects all results
public class CollectingScope<T> extends StructuredTaskScope<T> {
    private final List<T> results = new CopyOnWriteArrayList<>();
    private final List<Throwable> errors = new CopyOnWriteArrayList<>();
    
    @Override
    protected void handleComplete(Subtask<? extends T> subtask) {
        switch (subtask.state()) {
            case SUCCESS -> results.add(subtask.get());
            case FAILED -> errors.add(subtask.exception());
        }
    }
    
    public List<T> results() {
        return List.copyOf(results);
    }
    
    public List<Throwable> errors() {
        return List.copyOf(errors);
    }
}

// Usage
try (var scope = new CollectingScope<String>()) {
    scope.fork(() -> fetchFromService1());
    scope.fork(() -> fetchFromService2());
    scope.fork(() -> fetchFromService3());
    
    scope.join();
    
    List<String> results = scope.results();  // All successful results
    List<Throwable> errors = scope.errors(); // All errors
}
```

---

## 5️⃣ Migration from Thread Pools

### Before: Thread Pool

```java
// Traditional thread pool
@Configuration
public class AsyncConfig {
    
    @Bean
    public ExecutorService taskExecutor() {
        return Executors.newFixedThreadPool(200);  // Limited to 200 threads
    }
}

@Service
public class OrderService {
    
    @Autowired
    private ExecutorService executor;
    
    public Future<Order> processOrderAsync(OrderRequest request) {
        return executor.submit(() -> {
            // Process order (includes I/O)
            return processOrder(request);
        });
    }
}
```

### After: Virtual Threads

```java
// Virtual thread executor
@Configuration
public class AsyncConfig {
    
    @Bean
    public ExecutorService taskExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();  // Unlimited virtual threads!
    }
}

// Service code stays EXACTLY THE SAME!
@Service
public class OrderService {
    
    @Autowired
    private ExecutorService executor;
    
    public Future<Order> processOrderAsync(OrderRequest request) {
        return executor.submit(() -> {
            return processOrder(request);  // Now runs on virtual thread
        });
    }
}
```

### Spring Boot with Virtual Threads

```yaml
# application.yml (Spring Boot 3.2+)
spring:
  threads:
    virtual:
      enabled: true  # Enable virtual threads for request handling
```

```java
// Or configure programmatically
@Configuration
public class WebConfig {
    
    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
}
```

---

## 6️⃣ When to Use Virtual Threads

### Good Use Cases

```java
// 1. I/O-bound operations (database, HTTP calls, file I/O)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Response>> futures = urls.stream()
        .map(url -> executor.submit(() -> httpClient.get(url)))
        .toList();
}

// 2. High-concurrency servers
// Each request gets its own virtual thread
// Can handle millions of concurrent connections

// 3. Replacing async/reactive code
// Before (reactive):
Mono.fromCallable(() -> fetchUser(id))
    .flatMap(user -> fetchOrders(user))
    .subscribe();

// After (virtual threads - simpler!):
Thread.startVirtualThread(() -> {
    User user = fetchUser(id);  // Blocking is fine!
    List<Order> orders = fetchOrders(user);
});
```

### When NOT to Use Virtual Threads

```java
// 1. CPU-bound tasks (no I/O waiting)
// Virtual threads don't help - use parallel streams or ForkJoinPool
IntStream.range(0, 1_000_000)
    .parallel()
    .map(this::cpuIntensiveCalculation)
    .sum();

// 2. When you need thread-local caching
// Virtual threads are cheap - don't pool them!
// Thread-local caches won't be effective

// 3. With heavily synchronized code
// synchronized blocks pin virtual threads
// Migrate to ReentrantLock first

// 4. With native code / JNI
// Native calls pin virtual threads
```

---

## 7️⃣ Best Practices

### Do's

```java
// DO: Use virtual thread executor for I/O tasks
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // Each task gets its own virtual thread
}

// DO: Use ReentrantLock instead of synchronized
private final ReentrantLock lock = new ReentrantLock();

public void doWork() {
    lock.lock();
    try {
        // Critical section
    } finally {
        lock.unlock();
    }
}

// DO: Let virtual threads block
// It's the whole point! Blocking is cheap with virtual threads.
String result = blockingHttpCall();  // Fine!

// DO: Create many virtual threads
// They're cheap - don't pool them
for (int i = 0; i < 1_000_000; i++) {
    Thread.startVirtualThread(() -> handleRequest(i));
}
```

### Don'ts

```java
// DON'T: Pool virtual threads
// This defeats the purpose - they're meant to be created freely
ExecutorService pool = Executors.newFixedThreadPool(100);  // Wrong for virtual threads!

// DON'T: Use synchronized for long operations
synchronized (lock) {
    Thread.sleep(1000);  // Pins virtual thread!
}

// DON'T: Expect thread-local caching to work well
// Virtual threads are short-lived, thread-locals won't persist

// DON'T: Use for CPU-bound work
// Virtual threads help with I/O, not CPU
```

---

## 8️⃣ Interview Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is a virtual thread?**

A: A virtual thread is a lightweight thread managed by the JVM, not the OS. Unlike platform threads (~1MB stack), virtual threads use ~1KB and can be created in millions.

When a virtual thread blocks on I/O, it's "unmounted" from its carrier (platform) thread, allowing the carrier to run other virtual threads. This enables massive concurrency with simple blocking code.

**Q: How do you create a virtual thread?**

A: Several ways:
```java
// 1. Direct creation
Thread.startVirtualThread(() -> doWork());

// 2. Builder pattern
Thread.ofVirtual().start(() -> doWork());

// 3. Executor (recommended)
var executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> doWork());
```

### L5 (Mid-Level) Questions

**Q: What is "pinning" and how do you avoid it?**

A: Pinning occurs when a virtual thread cannot unmount from its carrier thread, typically during:
1. `synchronized` blocks
2. Native method calls

When pinned, the virtual thread occupies the carrier thread even while blocked, reducing concurrency.

Avoid pinning by:
- Using `ReentrantLock` instead of `synchronized`
- Minimizing native calls
- Use `-Djdk.tracePinnedThreads=short` to detect pinning

**Q: When should you NOT use virtual threads?**

A: Avoid virtual threads for:
1. **CPU-bound tasks**: No I/O to wait on, use parallel streams instead
2. **Heavily synchronized code**: Causes pinning, migrate to locks first
3. **Native code**: JNI calls pin threads
4. **Thread-local caching**: Virtual threads are short-lived, caches won't be effective

### L6 (Senior) Questions

**Q: How would you migrate a reactive application to virtual threads?**

A: Step-by-step approach:

1. **Identify I/O operations**: These benefit most from virtual threads
2. **Replace synchronized with ReentrantLock**: Prevent pinning
3. **Replace reactive chains with blocking code**:
   ```java
   // Before
   Mono.fromCallable(() -> fetchUser())
       .flatMap(user -> fetchOrders(user))
       .subscribe();
   
   // After
   User user = fetchUser();  // Blocking OK!
   List<Order> orders = fetchOrders(user);
   ```
4. **Use virtual thread executor**: `Executors.newVirtualThreadPerTaskExecutor()`
5. **Test for pinning**: `-Djdk.tracePinnedThreads=full`
6. **Monitor**: Virtual threads should reduce memory usage and increase throughput

Benefits: Simpler code, better stack traces, easier debugging, familiar programming model.

---

## 9️⃣ One Clean Mental Summary

Virtual threads are lightweight threads (~1KB vs ~1MB) that enable massive concurrency with simple blocking code. When a virtual thread blocks on I/O, it **unmounts** from its carrier (platform) thread, allowing the carrier to run other virtual threads. This means you can have millions of concurrent virtual threads on a few carrier threads. Use `Executors.newVirtualThreadPerTaskExecutor()` for I/O-bound tasks. Avoid **pinning** (when virtual threads can't unmount) by using `ReentrantLock` instead of `synchronized`. Don't pool virtual threads - they're cheap, create them freely. **Structured concurrency** (`StructuredTaskScope`) provides clean lifecycle management for concurrent tasks. Virtual threads are ideal for servers handling many concurrent connections with I/O, but not for CPU-bound work. The key insight: write simple blocking code, get reactive-level scalability.

