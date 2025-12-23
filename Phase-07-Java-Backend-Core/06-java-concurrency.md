# 🧵 Java Concurrency Deep Dive

---

## 0️⃣ Prerequisites

Before diving into Java Concurrency, you need to understand:

- **Process**: A running program with its own memory space. Your browser and IDE are separate processes.
- **Thread**: A lightweight unit of execution within a process. Multiple threads share the same memory.
- **CPU Core**: Physical hardware that executes instructions. Modern CPUs have multiple cores.
- **Context Switch**: When the CPU switches from executing one thread to another. Has overhead.
- **Race Condition**: When multiple threads access shared data and the result depends on timing.

Quick mental model:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROCESS vs THREAD                                     │
│                                                                          │
│   PROCESS (Your Java Application)                                       │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                                                                  │   │
│   │   Heap Memory (shared by all threads)                           │   │
│   │   ┌─────────────────────────────────────────────────────────┐   │   │
│   │   │  Objects, Arrays, Class instances                        │   │   │
│   │   └─────────────────────────────────────────────────────────┘   │   │
│   │                                                                  │   │
│   │   Thread 1          Thread 2          Thread 3                  │   │
│   │   ┌────────┐       ┌────────┐       ┌────────┐                 │   │
│   │   │ Stack  │       │ Stack  │       │ Stack  │                 │   │
│   │   │ (own)  │       │ (own)  │       │ (own)  │                 │   │
│   │   └────────┘       └────────┘       └────────┘                 │   │
│   │                                                                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   Key insight: Threads share heap but have their own stack              │
│   This is why concurrent access to shared objects is dangerous          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

If you understand that threads can run simultaneously and share memory, you're ready.

---

## 1️⃣ What Problem Does This Exist to Solve?

### The Pain Point

Imagine you're building a web server that handles 1000 requests per second. Each request takes 100ms to process (database query, computation, etc.).

**Without concurrency (single-threaded)**:

```
Request 1: [====100ms====]
Request 2:                [====100ms====]
Request 3:                                [====100ms====]
...

Time to process 1000 requests: 100,000ms = 100 seconds!
Users wait 100 seconds for their request to start processing!
```

**With concurrency (multi-threaded)**:

```
Thread 1: [Request 1][Request 5][Request 9]...
Thread 2: [Request 2][Request 6][Request 10]...
Thread 3: [Request 3][Request 7][Request 11]...
Thread 4: [Request 4][Request 8][Request 12]...

With 100 threads: All 1000 requests complete in ~1 second!
```

### What Systems Looked Like Before Modern Concurrency

Early Java (pre-1.5) had limited concurrency tools:

```java
// Old way: Manual thread management
Thread thread = new Thread(new Runnable() {
    public void run() {
        // Do work
    }
});
thread.start();

// Synchronization only via synchronized keyword
synchronized(lock) {
    // Critical section
}

// No thread pools, no futures, no atomic operations
// Every team built their own concurrency utilities
// Bugs everywhere!
```

### What Breaks Without Proper Concurrency

1. **Race Conditions**: Data corruption from unsynchronized access
2. **Deadlocks**: Threads waiting forever for each other
3. **Starvation**: Some threads never get CPU time
4. **Memory Visibility Issues**: Changes not seen by other threads
5. **Resource Exhaustion**: Creating too many threads crashes the system

### Real Examples of the Problem

**Therac-25 (1985-1987)**: Race condition in medical device killed patients. Two threads accessed shared state without synchronization.

**Northeast Blackout (2003)**: Race condition in power grid monitoring software contributed to cascading failure affecting 55 million people.

**Twitter's Early Days**: Thread pool exhaustion caused the famous "Fail Whale" as the system couldn't handle concurrent requests.

---

## 2️⃣ Intuition and Mental Model

### The Restaurant Kitchen Analogy

Think of concurrency like a busy restaurant kitchen:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RESTAURANT KITCHEN ANALOGY                            │
│                                                                          │
│   SINGLE-THREADED (One Chef):                                           │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Chef: [Prep Order 1][Cook Order 1][Plate Order 1]              │   │
│   │        [Prep Order 2][Cook Order 2][Plate Order 2]              │   │
│   │        [Prep Order 3]...                                         │   │
│   │                                                                  │   │
│   │  One order at a time. Customers wait forever.                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   MULTI-THREADED (Multiple Chefs):                                      │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Chef 1: [Prep Order 1][Cook Order 1][Plate Order 1]            │   │
│   │  Chef 2: [Prep Order 2][Cook Order 2][Plate Order 2]            │   │
│   │  Chef 3: [Prep Order 3][Cook Order 3][Plate Order 3]            │   │
│   │                                                                  │   │
│   │  Multiple orders in parallel. Fast service!                     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   SHARED RESOURCES (The Oven):                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Only one chef can use the oven at a time!                      │   │
│   │                                                                  │   │
│   │  Chef 1: "I need the oven" → [Uses Oven]                        │   │
│   │  Chef 2: "I need the oven" → [Waits...] → [Uses Oven]          │   │
│   │  Chef 3: "I need the oven" → [Waits...] → [Waits...] → [Uses]  │   │
│   │                                                                  │   │
│   │  The oven is a LOCK. Only one thread can hold it.              │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   DEADLOCK (Knife and Cutting Board):                                   │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Chef 1: Has knife, waiting for cutting board                   │   │
│   │  Chef 2: Has cutting board, waiting for knife                   │   │
│   │                                                                  │   │
│   │  Neither can proceed! Kitchen stops!                            │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key insights**:

- **Thread** = Chef (worker)
- **Shared Resource** = Kitchen equipment (oven, fridge)
- **Lock/Synchronized** = "In Use" sign on equipment
- **Deadlock** = Two chefs waiting for each other's equipment
- **Thread Pool** = Hiring a fixed number of chefs

---

## 3️⃣ How It Works Internally

### Thread Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    THREAD LIFECYCLE                                      │
│                                                                          │
│                        ┌─────────┐                                      │
│                        │   NEW   │  Thread created, not started         │
│                        └────┬────┘                                      │
│                             │ start()                                    │
│                             ▼                                            │
│                        ┌─────────┐                                      │
│                   ┌───►│ RUNNABLE│◄───┐  Ready to run or running       │
│                   │    └────┬────┘    │                                 │
│                   │         │         │                                 │
│         notify()  │         │         │ I/O complete                    │
│         notifyAll()         │         │ sleep() expires                 │
│         lock acquired       │         │ join() returns                  │
│                   │         │         │                                 │
│                   │         ▼         │                                 │
│              ┌────┴────┐         ┌────┴────┐                           │
│              │ WAITING │         │ BLOCKED │  Waiting for lock         │
│              │ TIMED_  │         └─────────┘                           │
│              │ WAITING │                                                │
│              └─────────┘                                                │
│                   │                                                      │
│                   │ wait(), sleep(), join()                             │
│                   │                                                      │
│                             │                                            │
│                             │ run() completes or exception              │
│                             ▼                                            │
│                        ┌─────────┐                                      │
│                        │TERMINATED│  Thread finished                    │
│                        └─────────┘                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Creating Threads

```java
// Method 1: Extend Thread class (not recommended)
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

// Method 2: Implement Runnable (recommended)
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

// Method 3: Lambda (most concise)
Runnable task = () -> {
    System.out.println("Running in: " + Thread.currentThread().getName());
};

// Usage
Thread t1 = new MyThread();
Thread t2 = new Thread(new MyRunnable());
Thread t3 = new Thread(task);

t1.start();  // Don't call run() directly!
t2.start();
t3.start();
```

### The synchronized Keyword

```java
public class Counter {
    private int count = 0;
    
    // WITHOUT synchronization - BROKEN!
    public void incrementUnsafe() {
        count++;  // Not atomic! Read-modify-write
    }
    
    // WITH synchronization - SAFE
    public synchronized void increment() {
        count++;  // Only one thread can execute at a time
    }
    
    // Synchronized block (more granular)
    public void incrementBlock() {
        // Non-critical code here (runs concurrently)
        
        synchronized(this) {
            count++;  // Only this part is synchronized
        }
        
        // More non-critical code (runs concurrently)
    }
    
    // Synchronized on specific lock object
    private final Object lock = new Object();
    
    public void incrementWithLock() {
        synchronized(lock) {
            count++;
        }
    }
}
```

### Why count++ is Not Atomic

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WHY count++ IS DANGEROUS                              │
│                                                                          │
│   count++ is actually THREE operations:                                 │
│   1. READ:  Load count from memory into register                        │
│   2. MODIFY: Add 1 to register                                          │
│   3. WRITE: Store register back to memory                               │
│                                                                          │
│   Two threads incrementing count (initial value = 0):                   │
│                                                                          │
│   Thread 1                    Thread 2                                  │
│   ─────────                   ─────────                                 │
│   READ count (0)                                                        │
│                               READ count (0)                            │
│   ADD 1 (now 1)                                                         │
│                               ADD 1 (now 1)                             │
│   WRITE count (1)                                                       │
│                               WRITE count (1)                           │
│                                                                          │
│   Expected: count = 2                                                   │
│   Actual: count = 1  ← LOST UPDATE!                                    │
│                                                                          │
│   This is a RACE CONDITION                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4️⃣ Simulation-First Explanation

### Scenario: Bank Account Transfer

```java
public class BankAccount {
    private double balance;
    private final String accountId;
    
    public BankAccount(String accountId, double initialBalance) {
        this.accountId = accountId;
        this.balance = initialBalance;
    }
    
    public synchronized void deposit(double amount) {
        balance += amount;
    }
    
    public synchronized void withdraw(double amount) {
        if (balance >= amount) {
            balance -= amount;
        } else {
            throw new IllegalStateException("Insufficient funds");
        }
    }
    
    public synchronized double getBalance() {
        return balance;
    }
}
```

### Transfer Without Proper Synchronization (BROKEN)

```java
// BROKEN: Not atomic across two accounts!
public void transferBroken(BankAccount from, BankAccount to, double amount) {
    from.withdraw(amount);  // What if exception here?
    to.deposit(amount);     // This never runs!
}
```

### Trace: Deadlock Scenario

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEADLOCK SCENARIO                                     │
│                                                                          │
│   Thread 1: Transfer $100 from Account A to Account B                   │
│   Thread 2: Transfer $50 from Account B to Account A                    │
│                                                                          │
│   BROKEN CODE:                                                          │
│   public void transfer(BankAccount from, BankAccount to, double amt) {  │
│       synchronized(from) {           // Lock 'from' first               │
│           synchronized(to) {         // Then lock 'to'                  │
│               from.withdraw(amt);                                        │
│               to.deposit(amt);                                           │
│           }                                                              │
│       }                                                                  │
│   }                                                                      │
│                                                                          │
│   EXECUTION:                                                            │
│   ═══════════════════════════════════════════════════════════════════   │
│                                                                          │
│   Thread 1                         Thread 2                             │
│   ─────────                        ─────────                            │
│   synchronized(A) ← ACQUIRED                                            │
│                                    synchronized(B) ← ACQUIRED           │
│   synchronized(B) ← WAITING...                                          │
│                                    synchronized(A) ← WAITING...         │
│                                                                          │
│   ╔═══════════════════════════════════════════════════════════════════╗ │
│   ║  DEADLOCK! Both threads waiting for each other forever!           ║ │
│   ╚═══════════════════════════════════════════════════════════════════╝ │
│                                                                          │
│   Thread 1 holds A, wants B                                             │
│   Thread 2 holds B, wants A                                             │
│   Neither can proceed!                                                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Fix: Consistent Lock Ordering

```java
// FIXED: Always lock in consistent order
public void transfer(BankAccount from, BankAccount to, double amount) {
    // Determine lock order by account ID (or any consistent ordering)
    BankAccount first = from.getAccountId().compareTo(to.getAccountId()) < 0 ? from : to;
    BankAccount second = first == from ? to : from;
    
    synchronized(first) {
        synchronized(second) {
            from.withdraw(amount);
            to.deposit(amount);
        }
    }
}

// Now both threads will always lock A before B (or B before A)
// No circular wait = No deadlock!
```

---

## 5️⃣ How Engineers Actually Use This in Production

### Real Systems at Real Companies

**Netflix's Thread Pool Strategy**:

```java
// Netflix uses separate thread pools for different operations
// to prevent one slow service from blocking others

@Configuration
public class ThreadPoolConfig {
    
    // Thread pool for API calls
    @Bean("apiExecutor")
    public ExecutorService apiExecutor() {
        return new ThreadPoolExecutor(
            10,                      // Core pool size
            50,                      // Max pool size
            60L, TimeUnit.SECONDS,   // Keep-alive time
            new LinkedBlockingQueue<>(1000),  // Queue capacity
            new ThreadFactoryBuilder()
                .setNameFormat("api-thread-%d")
                .build(),
            new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
        );
    }
    
    // Separate pool for database operations
    @Bean("dbExecutor")
    public ExecutorService dbExecutor() {
        return Executors.newFixedThreadPool(20);
    }
    
    // Separate pool for cache operations
    @Bean("cacheExecutor")
    public ExecutorService cacheExecutor() {
        return Executors.newCachedThreadPool();
    }
}
```

**Amazon's Async Processing**:

```java
// Using CompletableFuture for non-blocking operations
public class OrderService {
    
    private final ExecutorService executor;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    public CompletableFuture<OrderResult> processOrderAsync(Order order) {
        // Start all independent operations in parallel
        CompletableFuture<InventoryResult> inventoryFuture = 
            CompletableFuture.supplyAsync(
                () -> inventoryService.reserve(order),
                executor
            );
        
        CompletableFuture<PaymentResult> paymentFuture = 
            CompletableFuture.supplyAsync(
                () -> paymentService.authorize(order),
                executor
            );
        
        // Combine results when both complete
        return inventoryFuture
            .thenCombine(paymentFuture, (inventory, payment) -> {
                if (inventory.isSuccess() && payment.isSuccess()) {
                    return OrderResult.success(order.getId());
                }
                return OrderResult.failure("Processing failed");
            })
            .thenCompose(result -> {
                // Send notification after order processed
                return notificationService.sendAsync(order, result)
                    .thenApply(v -> result);
            })
            .exceptionally(ex -> {
                log.error("Order processing failed", ex);
                return OrderResult.failure(ex.getMessage());
            });
    }
}
```

### Real Workflows and Tooling

**Thread Dump Analysis**:

```bash
# Get thread dump from running JVM
jstack <pid> > thread_dump.txt

# Or programmatically
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(true, true);
```

**Detecting Deadlocks**:

```java
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();

if (deadlockedThreads != null) {
    ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(deadlockedThreads);
    for (ThreadInfo info : threadInfos) {
        log.error("Deadlocked thread: {}", info.getThreadName());
        log.error("Waiting for lock: {}", info.getLockName());
        log.error("Held by: {}", info.getLockOwnerName());
    }
}
```

### Production War Stories

**The Thread Pool Exhaustion**:

A team used a shared thread pool for all operations:

```java
// DISASTER: One pool for everything
ExecutorService executor = Executors.newFixedThreadPool(10);

// Slow database queries block all threads
executor.submit(() -> slowDatabaseQuery());  // Blocks for 30 seconds
executor.submit(() -> slowDatabaseQuery());  // Blocks
// ... all 10 threads blocked

// Fast API calls can't execute!
executor.submit(() -> fastApiCall());  // Queued forever!
```

**The fix: Bulkhead pattern**:

```java
// FIXED: Separate pools for different operations
ExecutorService dbPool = Executors.newFixedThreadPool(5);
ExecutorService apiPool = Executors.newFixedThreadPool(10);
ExecutorService cachePool = Executors.newFixedThreadPool(3);

// Slow DB queries don't affect API calls
dbPool.submit(() -> slowDatabaseQuery());
apiPool.submit(() -> fastApiCall());  // Runs immediately!
```

---

## 6️⃣ How to Implement: Complete Examples

### ExecutorService and Thread Pools

```java
import java.util.concurrent.*;

public class ThreadPoolExamples {
    
    public static void main(String[] args) throws Exception {
        
        // 1. Fixed Thread Pool
        // Use when: Known, bounded number of tasks
        ExecutorService fixedPool = Executors.newFixedThreadPool(4);
        
        // 2. Cached Thread Pool
        // Use when: Many short-lived tasks, pool grows/shrinks automatically
        ExecutorService cachedPool = Executors.newCachedThreadPool();
        
        // 3. Single Thread Executor
        // Use when: Tasks must execute sequentially
        ExecutorService singlePool = Executors.newSingleThreadExecutor();
        
        // 4. Scheduled Thread Pool
        // Use when: Tasks need to run at specific times or intervals
        ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(2);
        
        // 5. Custom Thread Pool (recommended for production)
        ThreadPoolExecutor customPool = new ThreadPoolExecutor(
            4,                              // Core pool size
            10,                             // Maximum pool size
            60L, TimeUnit.SECONDS,          // Keep-alive time for idle threads
            new ArrayBlockingQueue<>(100),  // Work queue
            new ThreadFactory() {
                private int counter = 0;
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r);
                    t.setName("custom-pool-" + counter++);
                    t.setDaemon(false);
                    return t;
                }
            },
            new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
        );
        
        // Submit tasks
        
        // Runnable: No return value
        fixedPool.submit(() -> {
            System.out.println("Task running in: " + Thread.currentThread().getName());
        });
        
        // Callable: Returns a value
        Future<Integer> future = fixedPool.submit(() -> {
            Thread.sleep(1000);
            return 42;
        });
        
        // Get result (blocks until complete)
        Integer result = future.get();  // 42
        
        // Get with timeout
        Integer resultWithTimeout = future.get(5, TimeUnit.SECONDS);
        
        // Schedule tasks
        scheduledPool.schedule(
            () -> System.out.println("Delayed task"),
            5, TimeUnit.SECONDS
        );
        
        scheduledPool.scheduleAtFixedRate(
            () -> System.out.println("Periodic task"),
            0,      // Initial delay
            10,     // Period
            TimeUnit.SECONDS
        );
        
        // Shutdown properly
        fixedPool.shutdown();  // Stop accepting new tasks
        try {
            if (!fixedPool.awaitTermination(60, TimeUnit.SECONDS)) {
                fixedPool.shutdownNow();  // Force shutdown
            }
        } catch (InterruptedException e) {
            fixedPool.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### Locks (ReentrantLock, ReadWriteLock, StampedLock)

```java
import java.util.concurrent.locks.*;

public class LockExamples {
    
    // ReentrantLock - More flexible than synchronized
    private final ReentrantLock lock = new ReentrantLock();
    private int counter = 0;
    
    public void incrementWithLock() {
        lock.lock();  // Acquire lock
        try {
            counter++;
        } finally {
            lock.unlock();  // ALWAYS unlock in finally!
        }
    }
    
    // Try lock with timeout
    public boolean tryIncrementWithTimeout() throws InterruptedException {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                counter++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;  // Couldn't acquire lock
    }
    
    // ReadWriteLock - Multiple readers OR one writer
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private Map<String, String> cache = new HashMap<>();
    
    public String read(String key) {
        readLock.lock();  // Multiple threads can read simultaneously
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }
    
    public void write(String key, String value) {
        writeLock.lock();  // Exclusive access for writing
        try {
            cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }
    
    // StampedLock - Optimistic reading (Java 8+)
    private final StampedLock stampedLock = new StampedLock();
    private double x, y;
    
    public void move(double deltaX, double deltaY) {
        long stamp = stampedLock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            stampedLock.unlockWrite(stamp);
        }
    }
    
    public double distanceFromOrigin() {
        // Optimistic read - no locking overhead if no write happens
        long stamp = stampedLock.tryOptimisticRead();
        double currentX = x;
        double currentY = y;
        
        // Check if a write occurred during our read
        if (!stampedLock.validate(stamp)) {
            // Write occurred, fall back to read lock
            stamp = stampedLock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                stampedLock.unlockRead(stamp);
            }
        }
        
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

### CompletableFuture (Async Programming)

```java
import java.util.concurrent.*;

public class CompletableFutureExamples {
    
    private final ExecutorService executor = Executors.newFixedThreadPool(4);
    
    public void examples() throws Exception {
        
        // 1. Create and complete manually
        CompletableFuture<String> cf1 = new CompletableFuture<>();
        cf1.complete("Done!");  // Complete with value
        // cf1.completeExceptionally(new RuntimeException("Failed"));
        
        // 2. Run async task (no return value)
        CompletableFuture<Void> cf2 = CompletableFuture.runAsync(() -> {
            System.out.println("Running async");
        });
        
        // 3. Supply async (with return value)
        CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> {
            return "Hello";
        }, executor);  // Custom executor
        
        // 4. Chain transformations
        CompletableFuture<Integer> cf4 = CompletableFuture
            .supplyAsync(() -> "Hello")
            .thenApply(s -> s + " World")      // Transform result
            .thenApply(String::length);         // Transform again
        
        // 5. Chain async operations
        CompletableFuture<String> cf5 = CompletableFuture
            .supplyAsync(() -> getUserId())
            .thenCompose(userId -> getUser(userId))  // Returns another CF
            .thenCompose(user -> getOrders(user));
        
        // 6. Combine two futures
        CompletableFuture<String> cf6 = CompletableFuture
            .supplyAsync(() -> "Hello")
            .thenCombine(
                CompletableFuture.supplyAsync(() -> "World"),
                (s1, s2) -> s1 + " " + s2
            );
        
        // 7. Run after completion (no access to result)
        CompletableFuture<Void> cf7 = CompletableFuture
            .supplyAsync(() -> "Hello")
            .thenRun(() -> System.out.println("Done!"));
        
        // 8. Handle errors
        CompletableFuture<String> cf8 = CompletableFuture
            .supplyAsync(() -> {
                if (Math.random() > 0.5) throw new RuntimeException("Failed");
                return "Success";
            })
            .exceptionally(ex -> {
                System.out.println("Error: " + ex.getMessage());
                return "Default";
            });
        
        // 9. Handle both success and error
        CompletableFuture<String> cf9 = CompletableFuture
            .supplyAsync(() -> "Hello")
            .handle((result, ex) -> {
                if (ex != null) {
                    return "Error: " + ex.getMessage();
                }
                return result.toUpperCase();
            });
        
        // 10. Wait for all futures
        CompletableFuture<Void> allOf = CompletableFuture.allOf(cf3, cf4, cf5);
        allOf.join();  // Wait for all to complete
        
        // 11. Wait for any future
        CompletableFuture<Object> anyOf = CompletableFuture.anyOf(cf3, cf4, cf5);
        Object firstResult = anyOf.join();  // First to complete
        
        // 12. Timeout (Java 9+)
        CompletableFuture<String> cf10 = CompletableFuture
            .supplyAsync(() -> slowOperation())
            .orTimeout(5, TimeUnit.SECONDS)
            .exceptionally(ex -> "Timeout!");
        
        // 13. Complete with default on timeout (Java 9+)
        CompletableFuture<String> cf11 = CompletableFuture
            .supplyAsync(() -> slowOperation())
            .completeOnTimeout("Default", 5, TimeUnit.SECONDS);
    }
    
    private String getUserId() { return "user123"; }
    private CompletableFuture<String> getUser(String id) { 
        return CompletableFuture.supplyAsync(() -> "User: " + id); 
    }
    private CompletableFuture<String> getOrders(String user) { 
        return CompletableFuture.supplyAsync(() -> "Orders for " + user); 
    }
    private String slowOperation() {
        try { Thread.sleep(10000); } catch (InterruptedException e) {}
        return "Done";
    }
}
```

### Atomic Types

```java
import java.util.concurrent.atomic.*;

public class AtomicExamples {
    
    // AtomicInteger - Thread-safe integer operations
    private final AtomicInteger counter = new AtomicInteger(0);
    
    public void atomicOperations() {
        counter.incrementAndGet();  // ++counter (returns new value)
        counter.getAndIncrement();  // counter++ (returns old value)
        counter.addAndGet(5);       // counter += 5
        counter.compareAndSet(5, 10);  // If value is 5, set to 10
        counter.updateAndGet(x -> x * 2);  // Atomic update with function
    }
    
    // AtomicReference - Thread-safe object reference
    private final AtomicReference<String> ref = new AtomicReference<>("initial");
    
    public void atomicReference() {
        ref.set("new value");
        String old = ref.getAndSet("another value");
        ref.compareAndSet("another value", "final value");
        ref.updateAndGet(s -> s.toUpperCase());
    }
    
    // AtomicLong - For counters, IDs
    private final AtomicLong idGenerator = new AtomicLong(0);
    
    public long nextId() {
        return idGenerator.incrementAndGet();
    }
    
    // LongAdder - Better for high contention counters (Java 8+)
    private final LongAdder adder = new LongAdder();
    
    public void highContentionCounter() {
        adder.increment();  // Faster than AtomicLong under contention
        adder.add(10);
        long sum = adder.sum();  // Get current value
    }
}
```

### volatile Keyword

```java
public class VolatileExample {
    
    // volatile ensures visibility across threads
    // Changes are immediately visible to all threads
    private volatile boolean running = true;
    
    public void runLoop() {
        while (running) {  // Without volatile, might loop forever!
            // Do work
        }
    }
    
    public void stop() {
        running = false;  // Other thread sees this immediately
    }
    
    // volatile does NOT make operations atomic!
    private volatile int counter = 0;
    
    public void brokenIncrement() {
        counter++;  // STILL NOT ATOMIC! Use AtomicInteger instead
    }
    
    // Use volatile for flags, not for compound operations
}
```

### ThreadLocal

```java
public class ThreadLocalExample {
    
    // Each thread gets its own copy
    private static final ThreadLocal<SimpleDateFormat> dateFormat = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
    
    // In a web application: store request context
    private static final ThreadLocal<RequestContext> requestContext = 
        new ThreadLocal<>();
    
    public String formatDate(Date date) {
        // Each thread uses its own SimpleDateFormat instance
        // SimpleDateFormat is NOT thread-safe, so this is important!
        return dateFormat.get().format(date);
    }
    
    public void processRequest(HttpServletRequest request) {
        try {
            // Set context at start of request
            requestContext.set(new RequestContext(request));
            
            // Now any code in this thread can access the context
            handleRequest();
            
        } finally {
            // ALWAYS clean up to prevent memory leaks!
            requestContext.remove();
        }
    }
    
    public static RequestContext getCurrentContext() {
        return requestContext.get();
    }
    
    // InheritableThreadLocal - Child threads inherit parent's value
    private static final InheritableThreadLocal<String> traceId = 
        new InheritableThreadLocal<>();
}
```

---

## 7️⃣ Tradeoffs, Pitfalls, and Common Mistakes

### Common Mistakes

**1. Not Handling InterruptedException Properly**

```java
// WRONG: Swallowing the interrupt
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // Don't just log and continue!
}

// RIGHT: Restore interrupt status
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();  // Restore interrupt flag
    throw new RuntimeException("Thread interrupted", e);
}
```

**2. Using synchronized Everywhere**

```java
// WRONG: Over-synchronization kills performance
public synchronized void method1() { /* ... */ }
public synchronized void method2() { /* ... */ }
public synchronized void method3() { /* ... */ }
// All methods block each other!

// RIGHT: Synchronize only what's necessary
public void method1() {
    // Non-critical code
    synchronized(this) {
        // Only critical section
    }
    // More non-critical code
}
```

**3. Forgetting to Shutdown ExecutorService**

```java
// WRONG: ExecutorService never shuts down
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(task);
// Program never exits! Threads keep running

// RIGHT: Always shutdown
ExecutorService executor = Executors.newFixedThreadPool(10);
try {
    executor.submit(task);
} finally {
    executor.shutdown();
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
}
```

**4. Creating Too Many Threads**

```java
// WRONG: Thread per request
for (Request request : requests) {
    new Thread(() -> process(request)).start();  // 10,000 threads!
}

// RIGHT: Use thread pool
ExecutorService executor = Executors.newFixedThreadPool(100);
for (Request request : requests) {
    executor.submit(() -> process(request));  // Bounded threads
}
```

**5. Double-Checked Locking Without volatile**

```java
// BROKEN: Without volatile, might see partially constructed object
private static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {
        synchronized(Singleton.class) {
            if (instance == null) {
                instance = new Singleton();  // Might be reordered!
            }
        }
    }
    return instance;
}

// FIXED: Use volatile
private static volatile Singleton instance;

// OR: Use holder pattern (better)
private static class Holder {
    static final Singleton INSTANCE = new Singleton();
}

public static Singleton getInstance() {
    return Holder.INSTANCE;  // Lazy, thread-safe, no synchronization
}
```

### Performance Considerations

| Mechanism          | Overhead   | Use When                                    |
| ------------------ | ---------- | ------------------------------------------- |
| synchronized       | Medium     | Simple cases, low contention                |
| ReentrantLock      | Medium     | Need tryLock, timeout, multiple conditions  |
| ReadWriteLock      | Medium     | Many readers, few writers                   |
| StampedLock        | Low-Medium | Optimistic reading beneficial               |
| AtomicInteger      | Low        | Single variable updates                     |
| LongAdder          | Very Low   | High-contention counters                    |
| volatile           | Very Low   | Simple flags, visibility only               |

---

## 8️⃣ When NOT to Use Manual Concurrency

### Situations Where Higher-Level Abstractions Are Better

**1. Use Parallel Streams for Simple Parallelism**

```java
// Instead of manual thread pool for data processing
List<Result> results = data.parallelStream()
    .filter(item -> item.isValid())
    .map(item -> process(item))
    .collect(Collectors.toList());
```

**2. Use CompletableFuture for Async Composition**

```java
// Instead of manual Future handling
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> transform(data))
    .thenAccept(result -> save(result));
```

**3. Use Virtual Threads (Java 21+) for I/O-Bound Tasks**

```java
// Instead of platform thread pools
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            // Each task gets its own virtual thread
            // Millions of concurrent tasks possible!
            fetchFromNetwork(i);
        });
    });
}
```

**4. Use Reactive Frameworks for Complex Async Flows**

```java
// Spring WebFlux for non-blocking I/O
Mono.fromCallable(() -> fetchUser(id))
    .flatMap(user -> fetchOrders(user))
    .subscribe(orders -> process(orders));
```

---

## 9️⃣ Comparison with Alternatives

### Concurrency Models Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONCURRENCY MODELS                                    │
│                                                                          │
│   SHARED MEMORY (Java threads):                                         │
│   ─────────────────────────────                                         │
│   Threads share heap memory                                             │
│   Synchronization via locks                                             │
│   Prone to race conditions, deadlocks                                   │
│   Familiar to most developers                                           │
│                                                                          │
│   ACTOR MODEL (Akka):                                                   │
│   ──────────────────                                                    │
│   Actors communicate via messages                                       │
│   No shared state                                                       │
│   Location transparent                                                  │
│   Good for distributed systems                                          │
│                                                                          │
│   CSP (Go goroutines):                                                  │
│   ────────────────────                                                  │
│   Lightweight threads (goroutines)                                      │
│   Communicate via channels                                              │
│   "Don't communicate by sharing memory;                                │
│    share memory by communicating"                                       │
│                                                                          │
│   VIRTUAL THREADS (Java 21+):                                           │
│   ───────────────────────────                                           │
│   Lightweight threads managed by JVM                                    │
│   Millions of concurrent threads                                        │
│   Same programming model as platform threads                            │
│   Great for I/O-bound workloads                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Java Concurrency Mechanisms Comparison

| Mechanism            | Use Case                          | Pros                    | Cons                    |
| -------------------- | --------------------------------- | ----------------------- | ----------------------- |
| synchronized         | Simple mutual exclusion           | Simple, built-in        | Inflexible              |
| ReentrantLock        | Advanced locking                  | tryLock, timeout        | Must unlock manually    |
| ReadWriteLock        | Read-heavy workloads              | Concurrent reads        | Complex                 |
| Semaphore            | Limiting concurrent access        | Rate limiting           | Easy to misuse          |
| CountDownLatch       | Wait for N events                 | Simple coordination     | Single use              |
| CyclicBarrier        | Sync threads at point             | Reusable                | All must arrive         |
| CompletableFuture    | Async composition                 | Powerful API            | Learning curve          |
| Virtual Threads      | I/O-bound tasks (Java 21+)        | Scalable, simple        | Not for CPU-bound       |

---

## 🔟 Interview Follow-Up Questions WITH Answers

### L4 (Entry-Level) Questions

**Q: What is the difference between a process and a thread?**

A: A process is an independent program with its own memory space. Processes are isolated; one process can't directly access another's memory. Creating processes is expensive.

A thread is a lightweight unit of execution within a process. Threads share the same memory space (heap) but have their own stack. Creating threads is cheaper than processes. Multiple threads can run concurrently within a process.

In Java, your application is a process. Each thread you create shares the heap memory, which is why concurrent access to shared objects needs synchronization.

**Q: What is the difference between synchronized and volatile?**

A: `synchronized` provides mutual exclusion (only one thread can execute the block) and visibility (changes are visible to other threads). It's used for compound operations that need atomicity.

`volatile` only provides visibility. Changes to a volatile variable are immediately visible to all threads. It does NOT provide atomicity. `volatile int x; x++;` is still not thread-safe because increment is three operations (read, modify, write).

Use `synchronized` for compound operations. Use `volatile` for simple flags or when you only need visibility.

### L5 (Mid-Level) Questions

**Q: Explain the difference between ReentrantLock and synchronized.**

A: Both provide mutual exclusion, but ReentrantLock is more flexible:

1. **tryLock()**: Can attempt to acquire without blocking, or with timeout
2. **Interruptible**: Can interrupt a thread waiting for a lock
3. **Fairness**: Can create fair locks (FIFO ordering)
4. **Multiple conditions**: Can have multiple wait/notify conditions
5. **Non-block-structured**: Can lock in one method and unlock in another

The downside is you must manually unlock in a finally block. synchronized automatically releases the lock.

Use synchronized for simple cases. Use ReentrantLock when you need tryLock, timeout, or multiple conditions.

**Q: How would you implement a thread-safe singleton?**

A: Several approaches:

```java
// 1. Eager initialization (simplest)
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    public static Singleton getInstance() { return INSTANCE; }
}

// 2. Holder pattern (lazy, thread-safe)
public class Singleton {
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}

// 3. Enum (Effective Java recommended)
public enum Singleton {
    INSTANCE;
    public void doSomething() { }
}
```

The holder pattern is lazy and thread-safe because class loading is synchronized by the JVM. The enum approach is simplest and handles serialization automatically.

### L6 (Senior) Questions

**Q: How would you design a rate limiter for a high-throughput API?**

A: I'd use a token bucket algorithm with atomic operations:

```java
public class RateLimiter {
    private final AtomicLong tokens;
    private final AtomicLong lastRefillTime;
    private final long maxTokens;
    private final long refillRate;  // tokens per second
    
    public boolean tryAcquire() {
        refillTokens();
        
        long currentTokens = tokens.get();
        if (currentTokens > 0) {
            return tokens.compareAndSet(currentTokens, currentTokens - 1);
        }
        return false;
    }
    
    private void refillTokens() {
        long now = System.nanoTime();
        long last = lastRefillTime.get();
        long elapsed = now - last;
        long tokensToAdd = elapsed * refillRate / 1_000_000_000;
        
        if (tokensToAdd > 0 && lastRefillTime.compareAndSet(last, now)) {
            long newTokens = Math.min(maxTokens, tokens.get() + tokensToAdd);
            tokens.set(newTokens);
        }
    }
}
```

For distributed systems, I'd use Redis with Lua scripts for atomic operations, or a distributed rate limiter like Resilience4j.

**Q: Explain how you would debug a deadlock in production.**

A: Step-by-step approach:

1. **Get thread dump**: `jstack <pid>` or JMX
2. **Look for "BLOCKED" threads**: These are waiting for locks
3. **Find circular dependencies**: Thread A waiting for lock held by Thread B, and vice versa
4. **Check lock ordering**: Inconsistent lock ordering is the usual cause

Prevention strategies:
- Always acquire locks in consistent order
- Use tryLock with timeout
- Minimize synchronized scope
- Use higher-level concurrency utilities

For detection, I'd add:
- JMX monitoring for deadlock detection
- Thread dump on timeout
- Logging of lock acquisition order

---

## 1️⃣1️⃣ One Clean Mental Summary

Java concurrency is about multiple threads sharing memory and CPU time. The core challenge is coordinating access to shared data without race conditions (corrupted data), deadlocks (threads waiting forever), or visibility issues (changes not seen). Use `synchronized` for simple mutual exclusion, `volatile` for simple flags, `AtomicInteger` for counters, and `ReentrantLock` when you need more control. For async work, use `ExecutorService` with bounded thread pools (never unbounded!) and `CompletableFuture` for composition. The key principles: minimize shared mutable state, use immutable objects when possible, synchronize only what's necessary, and always use try-finally to release locks. In modern Java (21+), virtual threads simplify I/O-bound concurrency dramatically. The goal isn't to use every concurrency tool. It's to write correct, performant code that doesn't deadlock or corrupt data.

