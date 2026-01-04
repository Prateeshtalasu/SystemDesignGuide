# Batch Processing

## 0️⃣ Prerequisites

- Understanding of database fundamentals (covered in Phase 3)
- Knowledge of database connection pooling (covered in topic 13)
- Understanding of async processing concepts (basic familiarity)
- Knowledge of transaction management (ACID properties)

**Quick refresher**: Database connection pooling reuses database connections to avoid the overhead of creating new connections for each query. Transactions ensure operations either all succeed (commit) or all fail (rollback), maintaining data consistency.

## 1️⃣ What problem does this exist to solve?

### The Pain Point

When applications need to process large amounts of data:

1. **Individual operations are too slow**: Inserting 100,000 records one-by-one takes hours
2. **Connection pool exhaustion**: Each individual operation holds a connection, exhausting the pool
3. **Transaction overhead**: Each operation in its own transaction creates massive overhead
4. **Network round trips**: Sending 100,000 individual requests creates network latency
5. **Resource waste**: Each operation has setup/teardown costs that add up

### What Systems Look Like Without Batch Processing

Without batch processing:

- **Bulk import takes 8 hours** because each record is inserted individually
- **Connection pool exhausted** when processing large datasets
- **Database overwhelmed** with millions of small transactions
- **Application timeouts** during bulk operations
- **Poor resource utilization** - CPU and network idle between operations

### Real Examples

**Example 1**: E-commerce platform importing 500,000 products nightly. Without batching, each product insert is a separate transaction. Result: Import takes 12 hours, times out, requires manual intervention.

**Example 2**: Analytics system processing user events. Each event is written individually to the database. Result: During peak traffic, connection pool exhausted, events are lost, system fails.

**Example 3**: Migration script moving data between databases. Moving 10 million records one-by-one. Result: Migration takes days, blocks other operations, causes production issues.

## 2️⃣ Intuition and Mental Model

**Think of moving boxes one at a time vs loading a truck:**

Without batch processing, you make 100 trips carrying one box each. Each trip requires:
- Walking to the truck (network round trip)
- Opening the door (connection setup)
- Placing the box (operation)
- Closing the door (transaction commit)
- Walking back (connection release)

With batch processing, you load a cart with 50 boxes, make one trip, and unload them all at once. The overhead is the same, but you move 50x more data per trip.

**The mental model**: Batch processing amortizes fixed costs (connection, transaction, network round trip) across many operations, making each operation much cheaper.

**Another analogy**: Think of batch processing like a shopping list. Instead of going to the store 20 times for one item each, you write a list, go once, and buy everything. The fixed cost (driving to the store) is the same, but you accomplish much more.

## 3️⃣ How it works internally

### Batch Processing Mechanics

#### Step 1: Accumulate Operations
The application collects multiple operations instead of executing them immediately:

```
Individual approach:
Operation 1 → Execute → Wait → Complete
Operation 2 → Execute → Wait → Complete
Operation 3 → Execute → Wait → Complete

Batch approach:
Operation 1 → Add to batch
Operation 2 → Add to batch
Operation 3 → Add to batch
...
Operation N → Add to batch
→ Execute entire batch → Complete all at once
```

#### Step 2: Batch Execution
When batch is ready (size threshold or time threshold reached):

1. **Open connection** (one time for entire batch)
2. **Begin transaction** (one transaction for entire batch)
3. **Prepare statement** (one prepared statement reused)
4. **Execute operations** (repeatedly with different parameters)
5. **Commit transaction** (one commit for entire batch)
6. **Close connection** (one time)

#### Step 3: Batch Size Optimization
Optimal batch size balances:
- **Memory usage**: Larger batches use more memory
- **Transaction duration**: Larger batches hold locks longer
- **Error recovery**: Larger batches have more work to retry on failure
- **Network efficiency**: Larger batches are more efficient but have diminishing returns

Typical batch sizes:
- **Small batches (10-100)**: For operations that might fail, quick feedback
- **Medium batches (100-1000)**: General purpose, good balance
- **Large batches (1000-10000)**: For bulk imports, stable operations

### Database-Level Batching

Databases optimize batch operations internally:

**PostgreSQL Batch Insert**:
```sql
-- Single insert (slow)
INSERT INTO users (email, name) VALUES ('user1@example.com', 'User 1');
INSERT INTO users (email, name) VALUES ('user2@example.com', 'User 2');
-- ... 10,000 more

-- Batch insert (fast)
INSERT INTO users (email, name) VALUES
  ('user1@example.com', 'User 1'),
  ('user2@example.com', 'User 2'),
  -- ... 10,000 more
  ('user10000@example.com', 'User 10000');
```

**JDBC Batch Updates**:
JDBC allows batching prepared statements:
```java
PreparedStatement stmt = conn.prepareStatement(
    "INSERT INTO users (email, name) VALUES (?, ?)");

for (User user : users) {
    stmt.setString(1, user.getEmail());
    stmt.setString(2, user.getName());
    stmt.addBatch(); // Add to batch, don't execute
}

stmt.executeBatch(); // Execute all at once
```

## 4️⃣ Simulation-first explanation

### Scenario: Inserting 1000 Users

**Start**: 1 application server, 1 database, need to insert 1000 user records

**Without batching (individual inserts)**:

```java
for (User user : users) {
    // 1. Get connection from pool (5ms)
    Connection conn = dataSource.getConnection();
    
    // 2. Begin transaction (1ms)
    conn.setAutoCommit(false);
    
    // 3. Prepare statement (2ms)
    PreparedStatement stmt = conn.prepareStatement(
        "INSERT INTO users (email, name) VALUES (?, ?)");
    stmt.setString(1, user.getEmail());
    stmt.setString(2, user.getName());
    
    // 4. Execute (1ms)
    stmt.executeUpdate();
    
    // 5. Commit (2ms)
    conn.commit();
    
    // 6. Close statement (1ms)
    stmt.close();
    
    // 7. Return connection to pool (1ms)
    conn.close();
    
    // Total per record: ~13ms
}

// Total time: 1000 × 13ms = 13,000ms = 13 seconds
```

**With batching (batch inserts)**:

```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false);

PreparedStatement stmt = conn.prepareStatement(
    "INSERT INTO users (email, name) VALUES (?, ?)");

int batchSize = 100;
int count = 0;

for (User user : users) {
    stmt.setString(1, user.getEmail());
    stmt.setString(2, user.getName());
    stmt.addBatch();
    count++;
    
    if (count % batchSize == 0) {
        stmt.executeBatch(); // Execute batch of 100
        conn.commit();
        stmt.clearBatch();
    }
}

// Execute remaining
if (count % batchSize != 0) {
    stmt.executeBatch();
    conn.commit();
}

stmt.close();
conn.close();

// Total time: ~200ms (65x faster!)
```

**Performance breakdown**:
- Connection overhead: 13ms → 0.013ms per record (1000x reduction)
- Transaction overhead: 2ms → 0.002ms per record (1000x reduction)
- Network round trips: 1000 → 10 (100x reduction)
- **Total speedup**: 65x faster

### Scenario: Batch Updates

**Updating 50,000 product prices**:

```java
// Without batching
for (Product product : products) {
    stmt = conn.prepareStatement(
        "UPDATE products SET price = ? WHERE id = ?");
    stmt.setBigDecimal(1, product.getPrice());
    stmt.setLong(2, product.getId());
    stmt.executeUpdate();
    conn.commit();
}
// Time: ~650 seconds (10.8 minutes)

// With batching
PreparedStatement stmt = conn.prepareStatement(
    "UPDATE products SET price = ? WHERE id = ?");

for (Product product : products) {
    stmt.setBigDecimal(1, product.getPrice());
    stmt.setLong(2, product.getId());
    stmt.addBatch();
    
    if (++count % 500 == 0) {
        stmt.executeBatch();
        conn.commit();
        stmt.clearBatch();
    }
}
// Time: ~15 seconds (43x faster)
```

## 5️⃣ How engineers actually use this in production

### Real-World Practices

#### 1. Spring Batch for Large-Scale Processing

**Netflix's data pipeline**: They use Spring Batch for processing millions of records:
- Chunk-oriented processing (read, process, write in chunks)
- Job restartability (can resume from failure)
- Parallel processing (multiple threads)
- Monitoring and reporting

**Example**: Processing user activity logs
- Read 10,000 records
- Transform/validate
- Write to database in batches
- Repeat until all records processed

#### 2. Bulk Import Patterns

**Uber's data ingestion**:
- CSV/JSON file uploads
- Parse and validate in memory
- Batch insert to database (10,000 records per batch)
- Progress tracking and error handling
- Rollback on critical errors

#### 3. Event Processing Batching

**Amazon's event processing**:
- Collect events in memory buffer
- Batch write to database every 5 seconds OR when buffer reaches 1000 events
- Reduces database writes by 100x
- Tradeoff: Slight delay in event availability (acceptable for analytics)

#### 4. Migration Scripts

**Facebook's database migrations**:
- Migrate data in batches to avoid locking tables
- Process 10,000 records at a time
- Pause between batches to allow other operations
- Checkpoint progress (can resume if interrupted)

### Production Workflow

**Step 1: Identify Batch Opportunities**
- Look for loops that execute database operations
- Identify bulk import/export operations
- Find repetitive operations that can be grouped

**Step 2: Implement Batching**
- Use JDBC batch APIs or ORM batch features
- Set appropriate batch sizes
- Handle batch errors gracefully

**Step 3: Monitor and Tune**
- Measure batch processing performance
- Adjust batch sizes based on results
- Monitor for connection pool exhaustion
- Watch for lock contention

## 6️⃣ How to implement or apply it

### Java/Spring Boot Implementation

#### Example 1: JDBC Batch Inserts

**Basic batch insert**:
```java
@Service
public class UserBatchService {
    
    @Autowired
    private DataSource dataSource;
    
    public void batchInsertUsers(List<User> users) throws SQLException {
        String sql = "INSERT INTO users (email, name, created_at) VALUES (?, ?, ?)";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            conn.setAutoCommit(false); // Enable batching
            
            int batchSize = 100;
            int count = 0;
            
            for (User user : users) {
                stmt.setString(1, user.getEmail());
                stmt.setString(2, user.getName());
                stmt.setTimestamp(3, Timestamp.from(user.getCreatedAt()));
                stmt.addBatch();
                
                if (++count % batchSize == 0) {
                    stmt.executeBatch();
                    conn.commit();
                    stmt.clearBatch();
                }
            }
            
            // Execute remaining
            if (count % batchSize != 0) {
                stmt.executeBatch();
                conn.commit();
            }
        }
    }
}
```

#### Example 2: JPA Batch Processing

**Hibernate batch configuration**:
```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
        jdbc.batch_versioned_data: true
```

**Batch insert with JPA**:
```java
@Service
@Transactional
public class UserBatchService {
    
    @Autowired
    private EntityManager entityManager;
    
    public void batchInsertUsers(List<User> users) {
        int batchSize = 50;
        
        for (int i = 0; i < users.size(); i++) {
            User user = users.get(i);
            entityManager.persist(user);
            
            // Flush and clear every batchSize entities
            if (i > 0 && i % batchSize == 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }
        
        // Flush remaining
        entityManager.flush();
    }
}
```

**Why flush and clear?**
- `flush()`: Sends batched SQL to database
- `clear()`: Clears persistence context (frees memory, prevents memory leaks)

#### Example 3: Spring Batch for Large Files

**Maven dependency**:
```xml
<dependency>
    <groupId>org.springframework.batch</groupId>
    <artifactId>spring-batch-core</artifactId>
</dependency>
```

**Batch job configuration**:
```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {
    
    @Bean
    public Job importUserJob(JobRepository jobRepository,
                             Step importStep) {
        return new JobBuilder("importUserJob", jobRepository)
            .incrementer(new RunIdIncrementer())
            .flow(importStep)
            .end()
            .build();
    }
    
    @Bean
    public Step importStep(JobRepository jobRepository,
                          PlatformTransactionManager transactionManager,
                          UserItemReader reader,
                          UserItemProcessor processor,
                          UserItemWriter writer) {
        return new StepBuilder("importStep", jobRepository)
            .<UserInput, User>chunk(1000, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
```

**Item Reader**:
```java
@Component
public class UserItemReader implements ItemReader<UserInput> {
    
    private List<UserInput> users;
    private int index = 0;
    
    @PostConstruct
    public void init() {
        // Load from CSV, JSON, or database
        users = loadUsersFromFile();
    }
    
    @Override
    public UserInput read() {
        if (index < users.size()) {
            return users.get(index++);
        }
        return null; // Signals end of input
    }
}
```

**Item Processor**:
```java
@Component
public class UserItemProcessor implements ItemProcessor<UserInput, User> {
    
    @Override
    public User process(UserInput input) throws Exception {
        // Transform and validate
        User user = new User();
        user.setEmail(input.getEmail().toLowerCase());
        user.setName(input.getName());
        
        // Validate
        if (user.getEmail() == null || !isValidEmail(user.getEmail())) {
            throw new ValidationException("Invalid email: " + input.getEmail());
        }
        
        return user;
    }
}
```

**Item Writer**:
```java
@Component
public class UserItemWriter implements ItemWriter<User> {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public void write(List<? extends User> users) throws Exception {
        // Spring Batch calls this with chunks (1000 items)
        // JPA will batch these based on hibernate.jdbc.batch_size
        userRepository.saveAll(users);
    }
}
```

**Running the job**:
```java
@Service
public class ImportService {
    
    @Autowired
    private JobLauncher jobLauncher;
    
    @Autowired
    @Qualifier("importUserJob")
    private Job importUserJob;
    
    public void importUsers() throws JobExecutionException {
        JobParameters params = new JobParametersBuilder()
            .addLong("time", System.currentTimeMillis())
            .toJobParameters();
        
        jobLauncher.run(importUserJob, params);
    }
}
```

#### Example 4: Batch Updates

**Batch update service**:
```java
@Service
public class ProductBatchService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public int batchUpdatePrices(List<ProductPriceUpdate> updates) {
        String sql = "UPDATE products SET price = ?, updated_at = ? WHERE id = ?";
        
        List<Object[]> batchArgs = updates.stream()
            .map(update -> new Object[]{
                update.getPrice(),
                Timestamp.from(Instant.now()),
                update.getProductId()
            })
            .collect(Collectors.toList());
        
        int[] results = jdbcTemplate.batchUpdate(sql, batchArgs);
        
        return Arrays.stream(results).sum();
    }
}
```

#### Example 5: Error Handling in Batches

**Robust batch processing with error handling**:
```java
@Service
public class RobustBatchService {
    
    @Autowired
    private DataSource dataSource;
    
    public BatchResult batchInsertWithErrorHandling(List<User> users) {
        BatchResult result = new BatchResult();
        String sql = "INSERT INTO users (email, name) VALUES (?, ?)";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            conn.setAutoCommit(false);
            int batchSize = 100;
            int count = 0;
            
            for (User user : users) {
                try {
                    stmt.setString(1, user.getEmail());
                    stmt.setString(2, user.getName());
                    stmt.addBatch();
                    count++;
                    
                    if (count % batchSize == 0) {
                        executeBatch(stmt, conn, result);
                    }
                } catch (SQLException e) {
                    result.addError(user, e);
                    // Continue with next user
                }
            }
            
            // Execute remaining
            if (count % batchSize != 0) {
                executeBatch(stmt, conn, result);
            }
            
        } catch (SQLException e) {
            result.setFatalError(e);
        }
        
        return result;
    }
    
    private void executeBatch(PreparedStatement stmt, Connection conn, 
                             BatchResult result) throws SQLException {
        try {
            int[] updateCounts = stmt.executeBatch();
            conn.commit();
            result.addSuccessCount(updateCounts.length);
        } catch (BatchUpdateException e) {
            // Partial failure - some succeeded, some failed
            int[] updateCounts = e.getUpdateCounts();
            SQLException nextException = e.getNextException();
            
            // Rollback this batch
            conn.rollback();
            
            // Log and track errors
            for (int i = 0; i < updateCounts.length; i++) {
                if (updateCounts[i] == Statement.EXECUTE_FAILED) {
                    result.addBatchError(i, nextException);
                } else {
                    result.addSuccessCount(1);
                }
            }
        }
    }
}

@Data
class BatchResult {
    private int successCount = 0;
    private List<ErrorRecord> errors = new ArrayList<>();
    private SQLException fatalError;
    
    void addSuccessCount(int count) {
        successCount += count;
    }
    
    void addError(User user, SQLException e) {
        errors.add(new ErrorRecord(user, e));
    }
    
    void addBatchError(int index, SQLException e) {
        errors.add(new ErrorRecord(null, e));
    }
}
```

## 7️⃣ Tradeoffs, pitfalls, and common mistakes

### Tradeoffs

#### 1. Batch Size: Performance vs Memory vs Lock Duration
- **Larger batches**: Better performance, more memory, longer locks
- **Smaller batches**: Less memory, shorter locks, more overhead
- **Sweet spot**: Usually 100-1000 for inserts, 100-500 for updates

#### 2. Transaction Scope: All-or-Nothing vs Partial Success
- **Large transaction**: Faster, but all fails if one fails
- **Small transactions**: Can partially succeed, but more overhead

#### 3. Error Handling: Fail Fast vs Continue on Error
- **Fail fast**: Easier to debug, but lose entire batch
- **Continue on error**: More resilient, but complex error handling

### Common Pitfalls

#### Pitfall 1: Batch Size Too Large
**Problem**: Memory exhaustion, long locks, connection timeout
```java
// Bad: 100,000 records in one batch
stmt.executeBatch(); // Out of memory or timeout
```

**Solution**: Use reasonable batch sizes (100-1000)
```java
// Good: Process in chunks
if (count % 500 == 0) {
    stmt.executeBatch();
}
```

#### Pitfall 2: Forgetting to Clear Batch
**Problem**: Batch keeps growing, memory leak
```java
// Bad: Never clearing batch
stmt.addBatch(); // Keeps accumulating
stmt.executeBatch(); // Executes ALL previous batches again!
```

**Solution**: Clear batch after execution
```java
// Good: Clear after execution
stmt.executeBatch();
stmt.clearBatch();
```

#### Pitfall 3: Not Handling Partial Batch Failures
**Problem**: Some records in batch succeed, some fail, inconsistent state
```java
// Bad: No error handling
stmt.executeBatch(); // Some might fail
conn.commit(); // Commits successful ones, but what about failures?
```

**Solution**: Handle BatchUpdateException
```java
// Good: Handle partial failures
try {
    stmt.executeBatch();
    conn.commit();
} catch (BatchUpdateException e) {
    conn.rollback(); // Rollback entire batch
    // Handle errors
}
```

#### Pitfall 4: Batching Read Operations
**Problem**: Batching SELECT queries doesn't help (each still executes separately)
```java
// Bad: Batching doesn't help SELECT
for (Long id : ids) {
    stmt.setLong(1, id);
    stmt.addBatch(); // Doesn't batch SELECT queries effectively
}
stmt.executeBatch(); // Executes each SELECT separately anyway
```

**Solution**: Use IN clause or joins for reads
```java
// Good: Single query with IN clause
String sql = "SELECT * FROM users WHERE id IN (?, ?, ...)";
// Or use WHERE id IN (:ids) with parameter array
```

#### Pitfall 5: Mixing Batch and Non-Batch Operations
**Problem**: Breaking batch optimization
```java
// Bad: Mixing operations breaks batching
stmt.addBatch(); // Add to batch
stmt.executeUpdate(); // Executes immediately, breaks batch
stmt.addBatch(); // Batch is now broken
```

**Solution**: Keep batch operations separate
```java
// Good: All batch operations together
stmt.addBatch();
stmt.addBatch();
stmt.executeBatch(); // Execute all at once
```

### Performance Gotchas

#### Gotcha 1: Connection Pool Exhaustion
**Problem**: Long-running batch holds connection, exhausting pool
**Solution**: Use smaller batches, release connection between batches

#### Gotcha 2: Lock Contention
**Problem**: Large batch holds locks, blocking other operations
**Solution**: Use smaller batches, process during low-traffic periods

#### Gotcha 3: Memory Usage
**Problem**: Large batches consume memory
**Solution**: Stream processing, process in chunks

## 8️⃣ When NOT to use this

### When Batching Isn't Needed

1. **Single operations**: If you're only doing one operation, batching adds complexity without benefit

2. **Read operations**: Batching SELECT queries doesn't provide the same benefits as batching writes

3. **Real-time requirements**: If you need immediate feedback, individual operations are better

4. **Operations that must succeed individually**: If each operation must succeed/fail independently, batching complicates error handling

### Anti-Patterns

1. **Batching everything**: Not all operations benefit from batching

2. **Ignoring errors**: Batch operations can partially fail, must handle errors carefully

3. **No size limits**: Unlimited batch sizes can cause memory issues

## 9️⃣ Comparison with Alternatives

### Batch Processing vs Async Processing

**Batch Processing**: Groups operations and executes together
- Pros: Simple, synchronous, easy to reason about
- Cons: Blocks thread, must wait for completion

**Async Processing**: Executes operations asynchronously
- Pros: Non-blocking, better resource utilization
- Cons: More complex, harder error handling

**Best practice**: Use batch processing for bulk operations, async for independent operations

### Batch Processing vs Stream Processing

**Batch Processing**: Process data in fixed-size chunks
- Pros: Simple, predictable resource usage
- Cons: Latency (must accumulate batch)

**Stream Processing**: Process data as it arrives
- Pros: Low latency, real-time
- Cons: More complex, variable resource usage

**Best practice**: Use batch processing for bulk imports/exports, stream processing for real-time data

### Batch Processing vs Bulk APIs

**Batch Processing**: Application-level batching
- Pros: Works with any database, flexible
- Cons: Application complexity

**Bulk APIs**: Database-provided bulk operations
- Pros: Optimized by database, simpler code
- Cons: Database-specific, less flexible

**Best practice**: Use database bulk APIs when available, fall back to application batching

## 🔟 Interview follow-up questions WITH answers

### Question 1: How do you determine the optimal batch size?

**Answer**:
Factors to consider:
1. **Memory constraints**: Larger batches use more memory
2. **Transaction duration**: Larger batches hold locks longer
3. **Error recovery**: Larger batches have more work to retry
4. **Network efficiency**: Larger batches reduce network round trips
5. **Database limitations**: Some databases have batch size limits

**Process**:
1. Start with a conservative size (100-500)
2. Measure performance (throughput, latency, memory)
3. Gradually increase batch size
4. Stop when performance plateaus or issues appear (memory, locks)
5. Consider different sizes for different operations

**Typical sizes**:
- **Small (10-100)**: Operations that might fail, need quick feedback
- **Medium (100-1000)**: General purpose, good balance
- **Large (1000-10000)**: Bulk imports, stable operations, dedicated batch jobs

**Follow-up**: "What if batch size is too large?"
- Memory exhaustion (OutOfMemoryError)
- Connection timeouts (batch takes too long)
- Lock contention (holding locks too long, blocking others)
- Large rollbacks on failure (expensive to undo)

### Question 2: How do you handle errors in batch processing?

**Answer**:
**Strategies**:

1. **All-or-nothing (transaction rollback)**:
   - Wrap entire batch in transaction
   - On any error, rollback entire batch
   - Retry entire batch
   - Best for: Operations that must be consistent

2. **Per-item error handling**:
   - Track which items succeeded/failed
   - Continue processing remaining items
   - Collect errors for later processing
   - Best for: Independent operations, bulk imports

3. **Batch-level error handling**:
   - If batch fails, retry individual items
   - Use smaller batches on retry
   - Best for: Mixed reliability operations

**Implementation**:
```java
try {
    int[] results = stmt.executeBatch();
    conn.commit();
} catch (BatchUpdateException e) {
    // Partial failure
    int[] updateCounts = e.getUpdateCounts();
    // Check which items failed
    for (int i = 0; i < updateCounts.length; i++) {
        if (updateCounts[i] == Statement.EXECUTE_FAILED) {
            // Item i failed, handle separately
        }
    }
    conn.rollback(); // Or partial commit if supported
}
```

### Question 3: When would you use Spring Batch vs manual batching?

**Answer**:
**Use Spring Batch when**:
- Large-scale data processing (millions of records)
- Need job restartability (resume from failure)
- Complex processing workflows (multiple steps)
- Need monitoring and reporting
- Processing files (CSV, JSON, XML)
- Scheduled batch jobs

**Use manual batching when**:
- Simple operations (few hundred records)
- Application-level batching (not dedicated batch jobs)
- Real-time processing with batching (events, API requests)
- Don't need job management features

**Example**: 
- **Spring Batch**: Nightly ETL job processing 10 million records
- **Manual batching**: API endpoint that batches database writes for performance

### Question 4: How does batch processing affect database connection pooling?

**Answer**:
**Without batching**:
- Each operation gets a connection
- Operations are quick, connections returned quickly
- Many connections needed for concurrent operations
- Connection pool might exhaust if too many concurrent operations

**With batching**:
- One connection used for many operations
- Connection held longer (entire batch duration)
- Fewer connections needed (fewer concurrent batches)
- Risk: Long-running batches hold connections, could exhaust pool if too many batches

**Optimization**:
- Use smaller batches to release connections faster
- Limit concurrent batch operations
- Use separate connection pool for batch jobs
- Monitor connection pool usage

### Question 5: How would you implement batch processing in a distributed system?

**Answer**:
**Challenges**:
- Coordination across nodes
- Duplicate processing prevention
- Error handling across nodes
- Progress tracking

**Approaches**:

1. **Partition-based batching**:
   - Partition data by key (user_id, date, etc.)
   - Each node processes its partition
   - No coordination needed

2. **Work queue with batching**:
   - Items added to distributed queue (RabbitMQ, Kafka)
   - Workers consume in batches
   - Idempotency keys prevent duplicates

3. **Leader-based coordination**:
   - Leader node coordinates batches
   - Assigns batches to workers
   - Tracks progress centrally

**Example with Kafka**:
```java
// Consumer with batch processing
@KafkaListener(topics = "events")
public void processBatch(List<ConsumerRecord<String, Event>> records) {
    List<Event> events = records.stream()
        .map(ConsumerRecord::value)
        .collect(Collectors.toList());
    
    // Process batch
    eventService.batchInsert(events);
}
```

### Question 6 (L5/L6): How would you design a batch processing system that handles failures and can resume?

**Answer**:
**System components**:

1. **Checkpointing**: Save progress after each batch
   - Store: Last processed ID, batch number, timestamp
   - Storage: Database table or distributed store (Redis, etcd)

2. **Idempotency**: Ensure operations are safe to retry
   - Use idempotency keys
   - Check if already processed before processing

3. **Error handling**: Multiple retry strategies
   - Exponential backoff for transient errors
   - Dead letter queue for permanent failures
   - Alerting for repeated failures

4. **Resume capability**: Can restart from last checkpoint
   - Read checkpoint on startup
   - Skip already processed items
   - Continue from last successful batch

**Architecture**:
```
Input Source → Batch Reader → Checkpoint Manager
                              ↓
                    Batch Processor → Error Handler
                              ↓
                    Batch Writer → Checkpoint Update
                              ↓
                    Progress Tracker
```

**Implementation considerations**:
- Atomic checkpoint updates (use transactions)
- Distributed locks for multi-node scenarios
- Monitoring and alerting on failures
- Manual intervention capabilities (pause, resume, skip)

## 1️⃣1️⃣ One clean mental summary

Batch processing groups multiple database operations together and executes them in a single round trip, dramatically improving performance by amortizing fixed costs (connection setup, transaction overhead, network latency) across many operations. The key is finding the optimal batch size that balances performance gains with memory usage and lock duration. Implementation involves using JDBC batch APIs, JPA batch features, or frameworks like Spring Batch for large-scale processing. Critical considerations include error handling (all-or-nothing vs per-item), connection pool management, and the tradeoffs between batch size, performance, and resource usage. Always measure and tune batch sizes based on your specific workload and database characteristics.

